# ver 1.0

require 'rubygems'
gem 'ci_reporter'
gem 'rspec'
require 'rspec/core/rake_task'
#require 'ci/reporter/rake/rspec' # use this if you're using RSpec

if ENV['GENERATE_REPORTS'] == 'true'
  require 'ci/reporter/rake/rspec'
  task :rspec => 'ci:setup:rspec'
end

GENERATE_REPORTS=true rake rspec

load File.join(File.dirname(__FILE__), "buildwise.rake")

## Settings: Customize here...
# 
BUILDWISE_URL = ENV["BUILDWISE_MASTER"] || "http://buildwise.dev"
BUILDWISE_QUICK_PROJECT_ID = "project-quick-build"
BUILDWISE_FULL_PROJECT_ID  = "project-full-build"
 
FULL_BUILD_MAX_TIME = 60 * 60   # 1 hour
FULL_BUILD_CHECK_INTERVAL = 60  # 1 minute

$test_dir =  File.expand_path(File.dirname(__FILE__))  # change to aboslution path if invoktion is not this directory
# rspec will be created 'spec/reports' under checkout dir

# List tests you want to exclude
#
def excluded_spec_files
  # ["ignore_spec.rb", "bad_test.rb", "selected_scripts_spec.rb"]
  ["#{$test_dir}/spec/passenger_spec.rb", "#{$test_dir}/spec/login_refactored_spec.rb"]
end

def all_specs
  Dir.glob("#{$test_dir}/spec/*_spec.rb")
end

def specs_for_quick_build
  # list test files to be run in a quick build
  [
    "#{$test_dir}/spec/login_spec.rb", 
  ]
end


desc "run all tests in this folder"
RSpec::Core::RakeTask.new("ui_tests:quick") do |t|
  specs_to_be_executed = all_specs # specs_for_quick_build
  
  puts specs_to_be_executed.inspect
  
  specs_to_be_executed -= excluded_spec_files
  specs_to_be_executed.uniq!  
  specs_to_be_executed.each do |a_test|
    specs_to_be_executed.delete(a_test) unless File.exists?(a_test)
  end
  
  puts "[INFO] Tests in order => #{specs_to_be_executed.collect {|x| File.basename(x)}.inspect}"
  t.pattern = FileList[specs_to_be_executed]
end



desc "run quick tests from BuildWise"
task "ci:ui_tests:quick" => ["ci:setup:rspec"] do
  build_id = buildwise_start_build(:project_name => BUILDWISE_QUICK_PROJECT_ID,
  :working_dir => File.expand_path(File.dirname(__FILE__)),
  :ui_test_dir => ["."],
  :excluded => excluded_spec_files
  )
  puts "[Rake] new build id =>|#{build_id}|"
  begin
    # puts "[Rake] Invoke"
    FileUtils.rm_rf("spec/reports") if File.exists?("spec/reports")
    Rake::Task["ui_tests:quick"].invoke
    # puts "[Rake] Invoke Finish"
  ensure
    puts "Finished: Notify build status"
    sleep 2 # wait a couple of seconds to finish writing last test results xml file out
    puts "[Rake] finish the build"
    buildwise_finish_build(build_id)
  end
end


## Full Build
#
#  TODO - how to determin useing RSpec or Cucumber
#
desc "Running tests distributedly"
task "ci:ui_tests:full" => ["ci:setup:rspec"] do
  build_id = buildwise_start_build(:project_name => BUILDWISE_FULL_PROJECT_ID,
                                   :working_dir => File.expand_path(File.dirname(__FILE__)),
                                   :ui_test_dir => ["spec"],
                                   :excluded => excluded_spec_files || [],
                                   :distributed => true
  )

  the_build_status = buildwise_build_ui_test_status(build_id)
  start_time = Time.now

  puts "[Rake] Keep checking build |#{build_id} | #{the_build_status}"
  while ((Time.now - start_time ) < FULL_BUILD_MAX_TIME) # test exeuction timeout
    the_build_status = buildwise_build_ui_test_status(build_id)
    puts "[Rake] #{Time.now} Checking build status: |#{the_build_status}|"
    if the_build_status == "OK"
      exit 0
    elsif the_build_status == "Failed"
      exit -1
    else 
      sleep FULL_BUILD_CHECK_INTERVAL  # check the build status every minute
    end
  end
  puts "[Rake] Execution UI tests expired"
  exit -2
end


desc "run all tests in this folder"
RSpec::Core::RakeTask.new("go") do |t|
  test_files = Dir.glob("*_spec.rb") + Dir.glob("*_test.rb") - excluded_test_files
  t.pattern = FileList[test_files]
  t.rspec_opts = "" # to enable warning: "-w"
end
