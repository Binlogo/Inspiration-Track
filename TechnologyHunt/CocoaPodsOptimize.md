# 工程效率优化：CocoaPods 优化

国内外针对通过 CocoaPods 这个依赖工具链如何做二进制缓存支持已经有了诸多方案和讨论，结合前段时间的一些实践，分享下这个话题，算是对前一段时间的一个回顾总结。

## 公开方案与工具汇总

- [swiftyfinch/Rugby](https://github.com/swiftyfinch/Rugby) - CocoaPods 缓存
- [grab/CocoaPods binary cache](https://github.com/grab/cocoapods-binary-cache) - Grab 的 CocoaPods 二进制缓存
- [MeetYouDevs/cocoapods-imy-bincocoapods-imy-bin](https://github.com/MeetYouDevs/cocoapods-imy-bin) - 美柚维护的 CocoaPods 二进制缓存
- [leavez/cocoapods-binary](https://github.com/leavez/cocoapods-binary) - 较早的一个 CocoaPods 二进制插件，目前已不维护，是诸多方案的灵感来源

  - Signal-iOS 正在使用

- 方案汇总：

- [🏈 Rugby: Optimise CocoaPods project | Swifty Finch](https://swiftyfinch.github.io/en/2021-03-09-rugby-story)
  - [知乎 iOS 基于 CocoaPods 实现的二进制化方案](https://zhuanlan.zhihu.com/p/44280283)
  - [基于 CocoaPods 的组件二进制化实践](https://triplecc.github.io/2019/01/21/基于CocoaPods的组件二进制化实践/)
  - [美团 iOS 工程 zsource 命令背后的那些事儿](https://tech.meituan.com/2019/08/08/the-things-behind-the-ios-project-zsource-command.html) - 二进制调试

**注**：以上为截止发布时的一些方案，如对未来更多编译优化话题感兴趣可以访问：[编译优化 - 棒棒彬的第二大脑](https://binlogo.github.io/Knowledge-Track/iOSDev/build-optimization.html)

## CocoaPods-Binary 原理与源码剖析

**关键字**：[Pre-Compiling Dependency](https://guides.cocoapods.org/plugins/pre-compiling-dependencies.html)，依赖预编译

### 解决的问题：

常规的 Pods 依赖安装后，即使没有对依赖 Pods 进行源码修改，Xcode 仍要重新进行编译；这在大型工程中，是十分耗时的不必要操作。

### 解决思路：

依赖安装 `pod install` 阶段对 Pods 中的依赖进行预编译，将编译的二进制产物（如：`.framework` 文件）集成添加到生成的 Xcode 工程中，用以取代源码集成。

CocoaPods Binary 插件通过在 pre-install 阶段添加以下步骤实现：

- 选中指定预编译的 Pod 依赖
- 编译这些依赖目标，将生成的二进制产物缓存起来
- 将依赖使用到的 Podspec 由原有的源码指定，改为指定编译产生的新二进制产物

### 用户使用界面设计：

- 安装：Ruby 统一依赖管理 Gem，在 Gemfile 指定，或直接 `gem install cocoapods-binary`
- 集成：CocoaPods 统一插件方式，在 Podfile 指定 `plugin 'cocoapods-binary'`
- “API”：

- 提供 `:binary => true` 作为依赖指定的额外选项
  - 提供全局选项 `all_binary!` 将所有依赖进行预编译

### 实现

#### Podfile 的使用界面

定义相关的 DSL，保存相关的选项参数：

```ruby
module Pod
    class Podfile
        module DSL

            # Enable prebuiding for all pods
            # it has a lower priority to other binary settings
            def all_binary!
                DSL.prebuild_all = true
            end

            # Enable bitcode for prebuilt frameworks
            def enable_bitcode_for_prebuilt_frameworks!
                DSL.bitcode_enabled = true
            end

            # Don't remove source code of prebuilt pods
            # It may speed up the pod install if git didn't
            # include the `Pods` folder
            def keep_source_code_for_prebuilt_frameworks!
                DSL.dont_remove_source_code = true
            end

            # Add custom xcodebuild option to the prebuilding action
            #
            # You may use this for your special demands. For example: the default archs in dSYMs
            # of prebuilt frameworks is 'arm64 armv7 x86_64', and no 'i386' for 32bit simulator.
            # It may generate a warning when building for a 32bit simulator. You may add following
            # to your podfile
            #
            #  ` set_custom_xcodebuild_options_for_prebuilt_frameworks :simulator => "ARCHS=$(ARCHS_STANDARD)" `
            #
            # Another example to disable the generating of dSYM file:
            #
            #  ` set_custom_xcodebuild_options_for_prebuilt_frameworks "DEBUG_INFORMATION_FORMAT=dwarf"`
            #
            #
            # @param [String or Hash] options
            #
            #   If is a String, it will apply for device and simulator. Use it just like in the commandline.
            #   If is a Hash, it should be like this: { :device => "XXXXX", :simulator => "XXXXX" }
            #
            def set_custom_xcodebuild_options_for_prebuilt_frameworks(options)
                if options.kind_of? Hash
                    DSL.custom_build_options = [ options[:device] ] unless options[:device].nil?
                    DSL.custom_build_options_simulator = [ options[:simulator] ] unless options[:simulator].nil?
                elsif options.kind_of? String
                    DSL.custom_build_options = [options]
                    DSL.custom_build_options_simulator = [options]
                else
                    raise "Wrong type."
                end
            end

            private
            class_attr_accessor :prebuild_all
            prebuild_all = false

            class_attr_accessor :bitcode_enabled
            bitcode_enabled = false

            class_attr_accessor :dont_remove_source_code
            dont_remove_source_code = false

            class_attr_accessor :custom_build_options
            class_attr_accessor :custom_build_options_simulator
            self.custom_build_options = []
            self.custom_build_options_simulator = []
        end
    end
end
```

#### Hook pre-install 阶段

通过 `HooksManager` 对 pre_install 进行 hook， 修改 Pod 安装环境与配置

```ruby
Pod::HooksManager.register('cocoapods-binary', :pre_install) do |installer_context|
  # ...

  # [Check Environment]
  # check user_framework is on
  podfile = installer_context.podfile
  podfile.target_definition_list.each do |target_definition|
    next if target_definition.prebuild_framework_pod_names.empty?
    if not target_definition.uses_frameworks?
      STDERR.puts "[!] Cocoapods-binary requires `use_frameworks!`".red
      exit
    end
  end

  # -- step 1: prebuild framework ---
  # Execute a sperated pod install, to generate targets for building framework,
  # then compile them to framework files.
  require_relative 'helper/prebuild_sandbox'
  require_relative 'Prebuild'

  Pod::UI.puts "🚀  Prebuild frameworks"

  # Fetch original installer (which is running this pre-install hook) options,
  # then pass them to our installer to perform update if needed
  # Looks like this is the most appropriate way to figure out that something should be updated

  update = nil
  repo_update = nil

  include ObjectSpace
  ObjectSpace.each_object(Pod::Installer) { |installer|
    update = installer.update
    repo_update = installer.repo_update
    }

  # control features
  Pod.is_prebuild_stage = true
  Pod::Podfile::DSL.enable_prebuild_patch true  # enable sikpping for prebuild targets
  Pod::Installer.force_disable_integration true # don't integrate targets
  Pod::Config.force_disable_write_lockfile true # disbale write lock file for perbuild podfile
  Pod::Installer.disable_install_complete_message true # disable install complete message

  # make another custom sandbox
  standard_sandbox = installer_context.sandbox
  prebuild_sandbox = Pod::PrebuildSandbox.from_standard_sandbox(standard_sandbox)

  # get the podfile for prebuild
  prebuild_podfile = Pod::Podfile.from_ruby(podfile.defined_in_file)

  # install
  lockfile = installer_context.lockfile
  binary_installer = Pod::Installer.new(prebuild_sandbox, prebuild_podfile, lockfile)

  if binary_installer.have_exact_prebuild_cache? && !update
    binary_installer.install_when_cache_hit!
  else
    binary_installer.update = update
    binary_installer.repo_update = repo_update
    binary_installer.install!
  end


  # reset the environment
  Pod.is_prebuild_stage = false
  Pod::Installer.force_disable_integration false
  Pod::Podfile::DSL.enable_prebuild_patch false
  Pod::Config.force_disable_write_lockfile false
  Pod::Installer.disable_install_complete_message false
  Pod::UserInterface.warnings = [] # clean the warning in the prebuild step, it's duplicated.

  # -- step 2: pod install ---
  # install
  Pod::UI.puts "\n"
  Pod::UI.puts "🤖  Pod Install"
  require_relative 'Integration'
  # go on the normal install step ...
end
```

#### 二进制预编译和缓存

从以上 Hook 的实现代码可以看到，`helper/prebuild_sandbox` 和 `Prebuild` 是预编译的核心代码

```ruby
  # -- step 1: prebuild framework ---
  # Execute a sperated pod install, to generate targets for building framework,
  # then compile them to framework files.
  require_relative 'helper/prebuild_sandbox'
  require_relative 'Prebuild'
```

`helper/prebuild_sandbox` 主要是定义了编译产物的一些路径。

`Prebuild` 主要是定义如何编译对应的 framework，以及对应的缓存策略。

```ruby
require_relative 'rome/build_framework'
require_relative 'helper/passer'
require_relative 'helper/target_checker'


# patch prebuild ability
module Pod
    class Installer

        private

				# ...

        public

        # check if need to prebuild
        def have_exact_prebuild_cache?
            # check if need build frameworks
            return false if local_manifest == nil

            changes = prebuild_pods_changes
            added = changes.added
            changed = changes.changed
            unchanged = changes.unchanged
            deleted = changes.deleted

            exsited_framework_pod_names = sandbox.exsited_framework_pod_names
            missing = unchanged.select do |pod_name|
                not exsited_framework_pod_names.include?(pod_name)
            end

            needed = (added + changed + deleted + missing)
            return needed.empty?
        end


        # The install method when have completed cache
        def install_when_cache_hit!
            # just print log
            self.sandbox.exsited_framework_target_names.each do |name|
                UI.puts "Using #{name}"
            end
        end


        # Build the needed framework files
        def prebuild_frameworks!

            # build options
            sandbox_path = sandbox.root
            existed_framework_folder = sandbox.generate_framework_path
            bitcode_enabled = Pod::Podfile::DSL.bitcode_enabled
            targets = []

            if local_manifest != nil

                changes = prebuild_pods_changes
                added = changes.added
                changed = changes.changed
                unchanged = changes.unchanged
                deleted = changes.deleted

                existed_framework_folder.mkdir unless existed_framework_folder.exist?
                exsited_framework_pod_names = sandbox.exsited_framework_pod_names

                # additions
                missing = unchanged.select do |pod_name|
                    not exsited_framework_pod_names.include?(pod_name)
                end


                root_names_to_update = (added + changed + missing)

                # transform names to targets
                cache = []
                targets = root_names_to_update.map do |pod_name|
                    tars = Pod.fast_get_targets_for_pod_name(pod_name, self.pod_targets, cache)
                    if tars.nil? || tars.empty?
                        raise "There's no target named (#{pod_name}) in Pod.xcodeproj.\n #{self.pod_targets.map(&:name)}" if t.nil?
                    end
                    tars
                end.flatten

                # add the dendencies
                dependency_targets = targets.map {|t| t.recursive_dependent_targets }.flatten.uniq || []
                targets = (targets + dependency_targets).uniq
            else
                targets = self.pod_targets
            end

            targets = targets.reject {|pod_target| sandbox.local?(pod_target.pod_name) }


            # build!
            Pod::UI.puts "Prebuild frameworks (total #{targets.count})"
            Pod::Prebuild.remove_build_dir(sandbox_path)
            targets.each do |target|
                if !target.should_build?
                    UI.puts "Prebuilding #{target.label}"
                    next
                end

                output_path = sandbox.framework_folder_path_for_target_name(target.name)
                output_path.mkpath unless output_path.exist?
                Pod::Prebuild.build(sandbox_path, target, output_path, bitcode_enabled,  Podfile::DSL.custom_build_options,  Podfile::DSL.custom_build_options_simulator)

                # save the resource paths for later installing
                if target.static_framework? and !target.resource_paths.empty?
                    framework_path = output_path + target.framework_name
                    standard_sandbox_path = sandbox.standard_sanbox_path

                    resources = begin
                        if Pod::VERSION.start_with? "1.5"
                            target.resource_paths
                        else
                            # resource_paths is Hash{String=>Array<String>} on 1.6 and above
                            # (use AFNetworking to generate a demo data)
                            # https://github.com/leavez/cocoapods-binary/issues/50
                            target.resource_paths.values.flatten
                        end
                    end
                    raise "Wrong type: #{resources}" unless resources.kind_of? Array

                    path_objects = resources.map do |path|
                        object = Prebuild::Passer::ResourcePath.new
                        object.real_file_path = framework_path + File.basename(path)
                        object.target_file_path = path.gsub('${PODS_ROOT}', standard_sandbox_path.to_s) if path.start_with? '${PODS_ROOT}'
                        object.target_file_path = path.gsub("${PODS_CONFIGURATION_BUILD_DIR}", standard_sandbox_path.to_s) if path.start_with? "${PODS_CONFIGURATION_BUILD_DIR}"
                        object
                    end
                    Prebuild::Passer.resources_to_copy_for_static_framework[target.name] = path_objects
                end

            end
            Pod::Prebuild.remove_build_dir(sandbox_path)


            # copy vendored libraries and frameworks
            targets.each do |target|
                root_path = self.sandbox.pod_dir(target.name)
                target_folder = sandbox.framework_folder_path_for_target_name(target.name)

                # If target shouldn't build, we copy all the original files
                # This is for target with only .a and .h files
                if not target.should_build?
                    Prebuild::Passer.target_names_to_skip_integration_framework << target.name
                    FileUtils.cp_r(root_path, target_folder, :remove_destination => true)
                    next
                end

                target.spec_consumers.each do |consumer|
                    file_accessor = Sandbox::FileAccessor.new(root_path, consumer)
                    lib_paths = file_accessor.vendored_frameworks || []
                    lib_paths += file_accessor.vendored_libraries
                    # @TODO dSYM files
                    lib_paths.each do |lib_path|
                        relative = lib_path.relative_path_from(root_path)
                        destination = target_folder + relative
                        destination.dirname.mkpath unless destination.dirname.exist?
                        FileUtils.cp_r(lib_path, destination, :remove_destination => true)
                    end
                end
            end

            # save the pod_name for prebuild framwork in sandbox
            targets.each do |target|
                sandbox.save_pod_name_for_target target
            end

            # Remove useless files
            # remove useless pods
            all_needed_names = self.pod_targets.map(&:name).uniq
            useless_target_names = sandbox.exsited_framework_target_names.reject do |name|
                all_needed_names.include? name
            end
            useless_target_names.each do |name|
                path = sandbox.framework_folder_path_for_target_name(name)
                path.rmtree if path.exist?
            end

            if not Podfile::DSL.dont_remove_source_code
                # only keep manifest.lock and framework folder in _Prebuild
                to_remain_files = ["Manifest.lock", File.basename(existed_framework_folder)]
                to_delete_files = sandbox_path.children.select do |file|
                    filename = File.basename(file)
                    not to_remain_files.include?(filename)
                end
                to_delete_files.each do |path|
                    path.rmtree if path.exist?
                end
            else
                # just remove the tmp files
                path = sandbox.root + 'Manifest.lock.tmp'
                path.rmtree if path.exist?
            end



        end


        # patch the post install hook
        old_method2 = instance_method(:run_plugins_post_install_hooks)
        define_method(:run_plugins_post_install_hooks) do
            old_method2.bind(self).()
            if Pod::is_prebuild_stage
                self.prebuild_frameworks!
            end
        end


    end
end
```

其中核心调用 xcodebuild 相关方法的核心编译逻辑在 `rome/build_framework.rb`。

```ruby
require 'fourflusher'
require 'xcpretty'

CONFIGURATION = "Release"
PLATFORMS = { 'iphonesimulator' => 'iOS',
              'appletvsimulator' => 'tvOS',
              'watchsimulator' => 'watchOS' }

#  Build specific target to framework file
#  @param [PodTarget] target
#         a specific pod target
#
def build_for_iosish_platform(sandbox,
                              build_dir,
                              output_path,
                              target,
                              device,
                              simulator,
                              bitcode_enabled,
                              custom_build_options = [], # Array<String>
                              custom_build_options_simulator = [] # Array<String>
                              )

  deployment_target = target.platform.deployment_target.to_s

  target_label = target.label # name with platform if it's used in multiple platforms
  Pod::UI.puts "Prebuilding #{target_label}..."

  other_options = []
  # bitcode enabled
  other_options += ['BITCODE_GENERATION_MODE=bitcode'] if bitcode_enabled
  # make less arch to iphone simulator for faster build
  custom_build_options_simulator += ['ARCHS=x86_64', 'ONLY_ACTIVE_ARCH=NO'] if simulator == 'iphonesimulator'

  is_succeed, _ = xcodebuild(sandbox, target_label, device, deployment_target, other_options + custom_build_options)
  exit 1 unless is_succeed
  is_succeed, _ = xcodebuild(sandbox, target_label, simulator, deployment_target, other_options + custom_build_options_simulator)
  exit 1 unless is_succeed

  # paths
  target_name = target.name # equals target.label, like "AFNeworking-iOS" when AFNetworking is used in multiple platforms.
  module_name = target.product_module_name
  device_framework_path = "#{build_dir}/#{CONFIGURATION}-#{device}/#{target_name}/#{module_name}.framework"
  simulator_framework_path = "#{build_dir}/#{CONFIGURATION}-#{simulator}/#{target_name}/#{module_name}.framework"

  device_binary = device_framework_path + "/#{module_name}"
  simulator_binary = simulator_framework_path + "/#{module_name}"
  return unless File.file?(device_binary) && File.file?(simulator_binary)

  # the device_lib path is the final output file path
  # combine the binaries
  tmp_lipoed_binary_path = "#{build_dir}/#{target_name}"
  lipo_log = `lipo -create -output #{tmp_lipoed_binary_path} #{device_binary} #{simulator_binary}`
  puts lipo_log unless File.exist?(tmp_lipoed_binary_path)
  FileUtils.mv tmp_lipoed_binary_path, device_binary, :force => true

  # collect the swiftmodule file for various archs.
  device_swiftmodule_path = device_framework_path + "/Modules/#{module_name}.swiftmodule"
  simulator_swiftmodule_path = simulator_framework_path + "/Modules/#{module_name}.swiftmodule"
  if File.exist?(device_swiftmodule_path)
    FileUtils.cp_r simulator_swiftmodule_path + "/.", device_swiftmodule_path
  end

  # combine the generated swift headers
  # (In xcode 10.2, the generated swift headers vary for each archs)
  # https://github.com/leavez/cocoapods-binary/issues/58
  simulator_generated_swift_header_path = simulator_framework_path + "/Headers/#{module_name}-Swift.h"
  device_generated_swift_header_path = device_framework_path + "/Headers/#{module_name}-Swift.h"
  if File.exist? simulator_generated_swift_header_path
    device_header = File.read(device_generated_swift_header_path)
    simulator_header = File.read(simulator_generated_swift_header_path)
    # https://github.com/Carthage/Carthage/issues/2718#issuecomment-473870461
    combined_header_content = %Q{
#if TARGET_OS_SIMULATOR // merged by cocoapods-binary

#{simulator_header}

#else // merged by cocoapods-binary

#{device_header}

#endif // merged by cocoapods-binary
}
    File.write(device_generated_swift_header_path, combined_header_content.strip)
  end

  # handle the dSYM files
  device_dsym = "#{device_framework_path}.dSYM"
  if File.exist? device_dsym
    # lipo the simulator dsym
    simulator_dsym = "#{simulator_framework_path}.dSYM"
    if File.exist? simulator_dsym
      tmp_lipoed_binary_path = "#{output_path}/#{module_name}.draft"
      lipo_log = `lipo -create -output #{tmp_lipoed_binary_path} #{device_dsym}/Contents/Resources/DWARF/#{module_name} #{simulator_dsym}/Contents/Resources/DWARF/#{module_name}`
      puts lipo_log unless File.exist?(tmp_lipoed_binary_path)
      FileUtils.mv tmp_lipoed_binary_path, "#{device_framework_path}.dSYM/Contents/Resources/DWARF/#{module_name}", :force => true
    end
    # move
    FileUtils.mv device_dsym, output_path, :force => true
  end

  # output
  output_path.mkpath unless output_path.exist?
  FileUtils.mv device_framework_path, output_path, :force => true

end

def xcodebuild(sandbox, target, sdk='macosx', deployment_target=nil, other_options=[])
  args = %W(-project #{sandbox.project_path.realdirpath} -scheme #{target} -configuration #{CONFIGURATION} -sdk #{sdk} )
  platform = PLATFORMS[sdk]
  args += Fourflusher::SimControl.new.destination(:oldest, platform, deployment_target) unless platform.nil?
  args += other_options
  log = `xcodebuild #{args.join(" ")} 2>&1`
  exit_code = $?.exitstatus  # Process::Status
  is_succeed = (exit_code == 0)

  if !is_succeed
    begin
        if log.include?('** BUILD FAILED **')
            # use xcpretty to print build log
            # 64 represent command invalid. http://www.manpagez.com/man/3/sysexits/
            printer = XCPretty::Printer.new({:formatter => XCPretty::Simple, :colorize => 'auto'})
            log.each_line do |line|
              printer.pretty_print(line)
            end
        else
            raise "shouldn't be handle by xcpretty"
        end
    rescue
        puts log.red
    end
  end
  [is_succeed, log]
end



module Pod
  class Prebuild

    # Build the frameworks with sandbox and targets
    #
    # @param  [String] sandbox_root_path
    #         The sandbox root path where the targets project place
    #
    #         [PodTarget] target
    #         The pod targets to build
    #
    #         [Pathname] output_path
    #         output path for generated frameworks
    #
    def self.build(sandbox_root_path, target, output_path, bitcode_enabled = false, custom_build_options=[], custom_build_options_simulator=[])

      return if target.nil?

      sandbox_root = Pathname(sandbox_root_path)
      sandbox = Pod::Sandbox.new(sandbox_root)
      build_dir = self.build_dir(sandbox_root)

      # -- build the framework
      case target.platform.name
      when :ios then build_for_iosish_platform(sandbox, build_dir, output_path, target, 'iphoneos', 'iphonesimulator', bitcode_enabled, custom_build_options, custom_build_options_simulator)
      when :osx then xcodebuild(sandbox, target.label, 'macosx', nil, custom_build_options)
      # when :tvos then build_for_iosish_platform(sandbox, build_dir, target, 'appletvos', 'appletvsimulator')
      when :watchos then build_for_iosish_platform(sandbox, build_dir, output_path, target, 'watchos', 'watchsimulator', true, custom_build_options, custom_build_options_simulator)
      else raise "Unsupported platform for '#{target.name}': '#{target.platform.name}'" end

      raise Pod::Informative, 'The build directory was not found in the expected location.' unless build_dir.directory?

      # # --- copy the vendored libraries and framework
      # frameworks = build_dir.children.select{ |path| File.extname(path) == ".framework" }
      # Pod::UI.puts "Built #{frameworks.count} #{'frameworks'.pluralize(frameworks.count)}"

      # pod_target = target
      # consumer = pod_target.root_spec.consumer(pod_target.platform.name)
      # file_accessor = Pod::Sandbox::FileAccessor.new(sandbox.pod_dir(pod_target.pod_name), consumer)
      # frameworks += file_accessor.vendored_libraries
      # frameworks += file_accessor.vendored_frameworks

      # frameworks.uniq!

      # frameworks.each do |framework|
      #   FileUtils.mkdir_p destination
      #   FileUtils.cp_r framework, destination, :remove_destination => true
      # end
      # build_dir.rmtree if build_dir.directory?
    end

    def self.remove_build_dir(sandbox_root)
      path = build_dir(sandbox_root)
      path.rmtree if path.exist?
    end

    private

    def self.build_dir(sandbox_root)
      # don't know why xcode chose this folder
      sandbox_root.parent + 'build'
    end

  end
end
```

#### 二进制产物或源码集成控制

预编译后，在常规 pod intall 之前对集成逻辑进行调整。核心逻辑在 `Integration.rb`。

```ruby
require_relative 'helper/podfile_options'
require_relative 'helper/feature_switches'
require_relative 'helper/prebuild_sandbox'
require_relative 'helper/passer'
require_relative 'helper/names'
require_relative 'helper/target_checker'


# NOTE:
# This file will only be loaded on normal pod install step
# so there's no need to check is_prebuild_stage



# Provide a special "download" process for prebuilded pods.
#
# As the frameworks is already exsited in local folder. We
# just create a symlink to the original target folder.
#
module Pod
    class Installer
        class PodSourceInstaller

            def install_for_prebuild!(standard_sanbox)
                return if standard_sanbox.local? self.name

                # make a symlink to target folder
                prebuild_sandbox = Pod::PrebuildSandbox.from_standard_sandbox(standard_sanbox)
                # if spec used in multiple platforms, it may return multiple paths
                target_names = prebuild_sandbox.existed_target_names_for_pod_name(self.name)

                def walk(path, &action)
                    return unless path.exist?
                    path.children.each do |child|
                        result = action.call(child, &action)
                        if child.directory?
                            walk(child, &action) if result
                        end
                    end
                end
                def make_link(source, target)
                    source = Pathname.new(source)
                    target = Pathname.new(target)
                    target.parent.mkpath unless target.parent.exist?
                    relative_source = source.relative_path_from(target.parent)
                    FileUtils.ln_sf(relative_source, target)
                end
                def mirror_with_symlink(source, basefolder, target_folder)
                    target = target_folder + source.relative_path_from(basefolder)
                    make_link(source, target)
                end

                target_names.each do |name|

                    # symbol link copy all substructure
                    real_file_folder = prebuild_sandbox.framework_folder_path_for_target_name(name)

                    # If have only one platform, just place int the root folder of this pod.
                    # If have multiple paths, we use a sperated folder to store different
                    # platform frameworks. e.g. AFNetworking/AFNetworking-iOS/AFNetworking.framework

                    target_folder = standard_sanbox.pod_dir(self.name)
                    if target_names.count > 1
                        target_folder += real_file_folder.basename
                    end
                    target_folder.rmtree if target_folder.exist?
                    target_folder.mkpath


                    walk(real_file_folder) do |child|
                        source = child
                        # only make symlink to file and `.framework` folder
                        if child.directory? and [".framework", ".dSYM"].include? child.extname
                            mirror_with_symlink(source, real_file_folder, target_folder)
                            next false  # return false means don't go deeper
                        elsif child.file?
                            mirror_with_symlink(source, real_file_folder, target_folder)
                            next true
                        else
                            next true
                        end
                    end


                    # symbol link copy resource for static framework
                    hash = Prebuild::Passer.resources_to_copy_for_static_framework || {}

                    path_objects = hash[name]
                    if path_objects != nil
                        path_objects.each do |object|
                            make_link(object.real_file_path, object.target_file_path)
                        end
                    end
                end # of for each

            end # of method

        end
    end
end


# Let cocoapods use the prebuild framework files in install process.
#
# the code only effect the second pod install process.
#
module Pod
    class Installer


        # Remove the old target files if prebuild frameworks changed
        def remove_target_files_if_needed

            changes = Pod::Prebuild::Passer.prebuild_pods_changes
            updated_names = []
            if changes == nil
                updated_names = PrebuildSandbox.from_standard_sandbox(self.sandbox).exsited_framework_pod_names
            else
                added = changes.added
                changed = changes.changed
                deleted = changes.deleted
                updated_names = added + changed + deleted
            end

            updated_names.each do |name|
                root_name = Specification.root_name(name)
                next if self.sandbox.local?(root_name)

                # delete the cached files
                target_path = self.sandbox.pod_dir(root_name)
                target_path.rmtree if target_path.exist?

                support_path = sandbox.target_support_files_dir(root_name)
                support_path.rmtree if support_path.exist?
            end

        end


        # Modify specification to use only the prebuild framework after analyzing
        old_method2 = instance_method(:resolve_dependencies)
        define_method(:resolve_dependencies) do

            # Remove the old target files, else it will not notice file changes
            self.remove_target_files_if_needed

            # call original
            old_method2.bind(self).()
            # ...
            # ...
            # ...
            # after finishing the very complex orginal function

            # check the pods
            # Although we have did it in prebuild stage, it's not sufficient.
            # Same pod may appear in another target in form of source code.
            # Prebuild.check_one_pod_should_have_only_one_target(self.prebuild_pod_targets)
            self.validate_every_pod_only_have_one_form


            # prepare
            cache = []

            def add_vendered_framework(spec, platform, added_framework_file_path)
                if spec.attributes_hash[platform] == nil
                    spec.attributes_hash[platform] = {}
                end
                vendored_frameworks = spec.attributes_hash[platform]["vendored_frameworks"] || []
                vendored_frameworks = [vendored_frameworks] if vendored_frameworks.kind_of?(String)
                vendored_frameworks += [added_framework_file_path]
                spec.attributes_hash[platform]["vendored_frameworks"] = vendored_frameworks
            end
            def empty_source_files(spec)
                spec.attributes_hash["source_files"] = []
                ["ios", "watchos", "tvos", "osx"].each do |plat|
                    if spec.attributes_hash[plat] != nil
                        spec.attributes_hash[plat]["source_files"] = []
                    end
                end
            end


            specs = self.analysis_result.specifications
            prebuilt_specs = (specs.select do |spec|
                self.prebuild_pod_names.include? spec.root.name
            end)

            prebuilt_specs.each do |spec|

                # Use the prebuild framworks as vendered frameworks
                # get_corresponding_targets
                targets = Pod.fast_get_targets_for_pod_name(spec.root.name, self.pod_targets, cache)
                targets.each do |target|
                    # the framework_file_path rule is decided when `install_for_prebuild`,
                    # as to compitable with older version and be less wordy.
                    framework_file_path = target.framework_name
                    framework_file_path = target.name + "/" + framework_file_path if targets.count > 1
                    add_vendered_framework(spec, target.platform.name.to_s, framework_file_path)
                end
                # Clean the source files
                # we just add the prebuilt framework to specific platform and set no source files
                # for all platform, so it doesn't support the sence that 'a pod perbuild for one
                # platform and not for another platform.'
                empty_source_files(spec)

                # to remove the resurce bundle target.
                # When specify the "resource_bundles" in podspec, xcode will generate a bundle
                # target after pod install. But the bundle have already built when the prebuit
                # phase and saved in the framework folder. We will treat it as a normal resource
                # file.
                # https://github.com/leavez/cocoapods-binary/issues/29
                if spec.attributes_hash["resource_bundles"]
                    bundle_names = spec.attributes_hash["resource_bundles"].keys
                    spec.attributes_hash["resource_bundles"] = nil
                    spec.attributes_hash["resources"] ||= []
                    spec.attributes_hash["resources"] += bundle_names.map{|n| n+".bundle"}
                end

                # to avoid the warning of missing license
                spec.attributes_hash["license"] = {}

            end

        end


        # Override the download step to skip download and prepare file in target folder
        old_method = instance_method(:install_source_of_pod)
        define_method(:install_source_of_pod) do |pod_name|

            # copy from original
            pod_installer = create_pod_installer(pod_name)
            # \copy from original

            if self.prebuild_pod_names.include? pod_name
                pod_installer.install_for_prebuild!(self.sandbox)
            else
                pod_installer.install!
            end

            # copy from original
            @installed_specs.concat(pod_installer.specs_by_platform.values.flatten.uniq)
            # \copy from original
        end


    end
end

# A fix in embeded frameworks script.
#
# The framework file in pod target folder is a symblink. The EmbedFrameworksScript use `readlink`
# to read the read path. As the symlink is a relative symlink, readlink cannot handle it well. So
# we override the `readlink` to a fixed version.
#
module Pod
    module Generator
        class EmbedFrameworksScript

            old_method = instance_method(:script)
            define_method(:script) do

                script = old_method.bind(self).()
                patch = <<-SH.strip_heredoc
                    #!/bin/sh

                    # ---- this is added by cocoapods-binary ---
                    # Readlink cannot handle relative symlink well, so we override it to a new one
                    # If the path isn't an absolute path, we add a realtive prefix.
                    old_read_link=`which readlink`
                    readlink () {
                        path=`$old_read_link "$1"`;
                        if [ $(echo "$path" | cut -c 1-1) = '/' ]; then
                            echo $path;
                        else
                            echo "`dirname $1`/$path";
                        fi
                    }
                    # ---
                SH

                # patch the rsync for copy dSYM symlink
                script = script.gsub "rsync --delete", "rsync --copy-links --delete"

                patch + script
            end
        end
    end
end
```

以上，基本就是 CocoaPods-Binary 的核心实现逻辑了。

## 总结

结合不同的二进制缓存方案，以及 CocoaPods-Binary 的源码实现，可以看出来，整体的方案是较为容易理解的，但通过 CocoaPods 插件实现时，有非常多的细节需要处理，包括友好的 API 用户交互界面和错误处理提示。
