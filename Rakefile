def execute_command(command)
  if ENV['VERBOSE']
    sh(command)
  else
    output = `#{command} 2>&1`
    raise output unless $?.success?
  end
end

namespace :gem do
  def gem_version
    require File.expand_path('../lib/cocoapods', __FILE__)
    Pod::VERSION
  end

  def gem_filename
    "cocoapods-#{gem_version}.gem"
  end

  desc "Build a gem for the current version"
  task :build do
    sh "gem build cocoapods.gemspec"
  end

  desc "Install a gem version of the current code"
  task :install => :build do
    sh "gem install #{gem_filename}"
  end

  def silent_sh(command)
    output = `#{command} 2>&1`
    unless $?.success?
      puts output
      exit 1
    end
    output
  end

  desc "Run all specs, build and install gem, commit version change, tag version change, and push everything"
  task :release do

    unless ENV['SKIP_CHECKS']
      if `git symbolic-ref HEAD 2>/dev/null`.strip.split('/').last != 'master'
        $stderr.puts "[!] You need to be on the `master' branch in order to be able to do a release."
        exit 1
      end

      if `git tag`.strip.split("\n").include?(gem_version)
        $stderr.puts "[!] A tag for version `#{gem_version}' already exists. Change the version in lib/cocoapods.rb"
        exit 1
      end

      puts "You are about to release `#{gem_version}', is that correct? [y/n]"
      exit if $stdin.gets.strip.downcase != 'y'

      diff_lines = `git diff --name-only`.strip.split("\n")

      if diff_lines.size == 0
        $stderr.puts "[!] Change the version number yourself in lib/cocoapods.rb"
        exit 1
      end

      diff_lines.delete('Gemfile.lock')
      diff_lines.delete('CHANGELOG.md')
      if diff_lines != ['lib/cocoapods.rb']
        $stderr.puts "[!] Only change the version number in a release commit!"
        exit 1
      end
    end

    require 'date'

    # First check if the required Xcodeproj gem has ben pushed
    gem_spec = eval(File.read(File.expand_path('../cocoapods.gemspec', __FILE__)))
    xcodeproj = gem_spec.dependencies.find { |d| d.name == 'xcodeproj' }
    required_xcodeproj_version = xcodeproj.requirement.requirements.first.last.to_s

    puts "* Checking if xcodeproj #{required_xcodeproj_version} exists on the gem host"
    search_result = silent_sh("gem search --all --remote xcodeproj")
    remote_xcodeproj_versions = search_result.match(/xcodeproj \((.*)\)/m)[1].split(', ')
    unless remote_xcodeproj_versions.include?(required_xcodeproj_version)
      $stderr.puts "[!] The Xcodeproj version `#{required_xcodeproj_version}' required by " \
                   "this version of CocoaPods does not exist on the gem host. " \
                   "Either push that first, or fix the version requirement."
      exit 1
    end

    # Ensure that the branches are up to date with the remote
    sh "git pull -s recursive -X theirs"

    puts "* Running specs"
    silent_sh('rake spec:all')

    tmp = File.expand_path('../tmp', __FILE__)
    tmp_gems = File.join(tmp, 'gems')

    Rake::Task['gem:build'].invoke

    puts "* Testing gem installation (tmp/gems)"
    silent_sh "rm -rf '#{tmp}'"
    silent_sh "gem install --install-dir='#{tmp_gems}' #{gem_filename}"

    # puts "* Building examples from gem (tmp/gems)"
    # ENV['GEM_HOME'] = ENV['GEM_PATH'] = tmp_gems
    # ENV['PATH']     = "#{tmp_gems}/bin:#{ENV['PATH']}"
    # ENV['FROM_GEM'] = '1'
    # silent_sh "rake examples:build"

    # Then release
    sh "git commit lib/cocoapods.rb Gemfile.lock CHANGELOG.md -m 'Release #{gem_version}'"
    sh "git tag -a #{gem_version} -m 'Release #{gem_version}'"
    sh "git push origin master"
    sh "git push origin --tags"
    sh "gem push #{gem_filename}"

    # Update the last version in CocoaPods-version.yml
    puts "* Updating last known version in Specs repo"
    specs_branch = 'master'
    Dir.chdir('../Specs') do
      puts Dir.pwd
      sh "git checkout #{specs_branch}"
      sh "git pull"

      yaml_file  = 'CocoaPods-version.yml'
      unless File.exist?(yaml_file)
        $stderr.puts "[!] Unable to find #{yaml_file}!"
        exit 1
      end
      require 'yaml'
      cocoapods_version = YAML.load_file(yaml_file)
      cocoapods_version['last'] = gem_version
      File.open(yaml_file, "w") do |f|
        f.write(cocoapods_version.to_yaml)
      end

      sh "git commit #{yaml_file} -m 'CocoaPods release #{gem_version}'"
      sh "git push"
    end
  end
end

namespace :spec do
  def specs(dir)
    FileList["spec/#{dir}/*_spec.rb"].shuffle.join(' ')
  end

  desc "Automatically run specs for updated files"
  task :kick do
    exec "bundle exec kicker -c"
  end

  desc "Run the unit specs"
  task :unit => :unpack_fixture_tarballs do
    sh "bundle exec bacon #{specs('unit/**')} -q"
  end

  desc "Run the functional specs"
  task :functional => :unpack_fixture_tarballs do
    sh "bundle exec bacon #{specs('functional/**')}"
  end

  desc "Run the integration spec"
  task :integration => :unpack_fixture_tarballs do
    sh "bundle exec bacon spec/integration_spec.rb"
  end

  task :all => :unpack_fixture_tarballs do
    sh "bundle exec bacon #{specs('**')}"
  end

  desc "Run all specs and build all examples"
  task :ci => :all do
    sh "./bin/pod setup" # ensure the spec repo is up-to-date
    Rake::Task['examples:build'].invoke
  end

  desc "Rebuild all the fixture tarballs"
  task :rebuild_fixture_tarballs do
    tarballs = FileList['spec/fixtures/**/*.tar.gz']
    tarballs.each do |tarball|
      basename = File.basename(tarball)
      sh "cd #{File.dirname(tarball)} && rm #{basename} && tar -zcf #{basename} #{basename[0..-8]}"
    end
  end

  desc "Unpacks all the fixture tarballs"
  task :unpack_fixture_tarballs do
    tarballs = FileList['spec/fixtures/**/*.tar.gz']
    tarballs.each do |tarball|
      basename = File.basename(tarball)
      Dir.chdir(File.dirname(tarball)) do
        sh "rm -rf #{basename[0..-8]} && tar zxf #{basename}"
      end
    end
  end

  desc "Removes the stored VCR fixture"
  task :clean_vcr do
    sh "rm -f spec/fixtures/vcr/tarballs.yml"
  end

  task :clean_env => [:clean_vcr, :unpack_fixture_tarballs, "ext:cleanbuild"]
end

namespace :examples do
  def examples
    require 'pathname'
    result = []
    examples = Pathname.new(File.expand_path('../examples', __FILE__))
    return [examples + ENV['example']] if ENV['example']
    examples.entries.each do |example|
      next if %w{ . .. }.include?(example.basename.to_s)
      example = examples + example
      next unless example.directory?
      result << example
    end
    result
  end

  desc "Open all example workspaced in Xcode, which recreates the schemes."
  task :recreate_workspace_schemes do
    examples.each do |example|
      Dir.chdir(example.to_s) do
        # TODO we need to open the workspace in Xcode at least once, otherwise it might not contain schemes.
        # The schemes do not seem to survive a SCM round-trip.
        sh "open '#{example.basename}.xcworkspace'"
        sleep 5
      end
    end
  end

  desc "Build all examples"
  task :build do
    execute_command "rm -rf ~/Library/Developer/Shared/Documentation/DocSets/org.cocoapods.*"
    examples.entries.each do |example|
      puts "Building example: #{example}"
      Dir.chdir(example.to_s) do
        execute_command "rm -rf Pods DerivedData"
        execute_command "#{'../../bin/' unless ENV['FROM_GEM']}pod install --verbose --no-update"
        command = "xcodebuild -workspace '#{example.basename}.xcworkspace' -scheme '#{example.basename}'"
        if (example + 'Podfile').read.include?('platform :ios')
          # Specifically build against the simulator SDK so we don't have to deal with code signing.
          command << " -sdk "
          command << Dir.glob("#{`xcode-select -print-path`.chomp}/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator*.sdk").last
        end
        execute_command(command)
      end
    end
  end
end

desc "Initializes your working copy to run the specs"
task :bootstrap do
  puts "Updating submodules"
  execute_command "git submodule update --init --recursive"

  puts "Installing gems"
  execute_command "bundle install"

  puts "Installing tools (Homebrew)"
  execute_command "brew install appledoc" if `which appledoc`.strip.empty?
  execute_command "brew install mercurial" if `which hg`.strip.empty?
end

desc "Run all specs"
task :spec => 'spec:all'

task :default => :spec
