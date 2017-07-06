require 'rubygems'
require 'open-uri'

WAR_LOCATION = File.expand_path(File.join(File.dirname(__FILE__), "jenkins.war"))
CLI_LOCATION = File.expand_path(File.join(File.dirname(__FILE__), "war/WEB-INF/jenkins-cli.jar"))

def build_root
  File.expand_path(File.dirname(__FILE__))
end

def mirror
  "http://mirrors.jenkins-ci.org"
  #"http://mirror.xmission.com/jenkins"
end

def oneops
  "https://s3.amazonaws.com/oneops/download"
end

def updates
  "http://updates.jenkins-ci.org/stable"
  #"http://mirror.xmission.com/jenkins/updates"
end

def version
  "1.651.3"
end

ENV['JENKINS_HOME'] = build_root
ENV['JENKINS_PORT'] = ENV['JENKINS_PORT'] || '3001'
ENV['JENKINS_URL'] = ENV['JENKINS_URL'] || "http://localhost:#{ENV['JENKINS_PORT']}"

desc "install jenkins war"
task :war do
  url = "#{mirror}/war-stable/#{version}/jenkins.war"
  puts "Downloading jenkins war: #{url}"
  File.open("jenkins.war", "wb") do |war_file|
    open(url, 'rb') do |read_file|
      war_file.write(read_file.read)
    end
  end
  system("java", "-jar", "jenkins.war", "--version")
end

desc "install jenkins plugins"
task :plugins do
  FileUtils.mkdir_p('plugins')
  IO.readlines('plugins.txt').each do |p|
    p.chomp!
    name, ver = p.split(':')
    url = "#{mirror}/plugins/#{name}/#{ver || 'latest'}/#{name}.hpi"
    #url = "#{mirror}/plugins/#{p}/latest/#{p}.hpi"
    puts "Downloading plugin: #{url}"
    File.open("#{build_root}/plugins/#{p}.jpi", "wb") do |plugin_file|
      open(url, 'rb') do |read_file|
        plugin_file.write(read_file.read)
      end
    end
    # purge old plugin directory
    system("rm", "-fr", "#{build_root}/plugins/#{p}")
    # https://issues.jenkins-ci.org/browse/JENKINS-19927
    system("touch #{build_root}/plugins/credentials.jpi.pinned")
  end
end

desc "install oneops plugin"
task :oneops do
  FileUtils.mkdir_p('plugins')
  url = "#{oneops}/oneops.hpi"
  puts "Downloading plugin: #{url}"
  File.open("#{build_root}/plugins/oneops.jpi", "wb") do |plugin_file|
    open(url, 'rb') do |read_file|
      plugin_file.write(read_file.read)
    end
  end
  # purge old plugin directory
  system("rm", "-fr", "#{build_root}/plugins/oneops")
end

desc "install jenkins updates"
task :updates do
  update_dir = File.join(build_root, "updates")
  FileUtils.mkdir_p update_dir
  url = "#{updates}/update-center.json"
  puts "Downloading updates: #{url}"
  File.open("#{build_root}/update-center.json", "wb") do |update_file|
    open(url, 'rb') do |read_file|
      update_file.write(read_file.read)
    end
  end
  system("cat #{build_root}/update-center.json | sed '1d;$d' > #{update_dir}/default.json")
end

desc "setup github home"
task :github do
  github = ENV['github']
  if github
    puts "Skipping github setup"
  else
    system("cp", "#{build_root}/build.properties.default", "#{build_root}/build.properties")
    puts "Set github base to #{github}"
  end
end

desc "setup default build.properties"
task :properties do
  if File.file?("#{build_root}/build.properties")
    puts "Skipping build.properties install because it already exists"
  else
    system("cp", "#{build_root}/build.properties.default", "#{build_root}/build.properties")
    puts "Installed build.properties from default template"
  end
end

desc "initialize build numbers"
task :sequence do
  Dir.glob("#{build_root}/jobs/*").select {|f| File.directory? f}.each do |j|
    file = "#{j}/nextBuildNumber"
    File.open(file, 'w') {|f| f.write("1\n") } unless File.file?(file)
  end
  Dir.glob("#{build_root}/jobs/*/modules/*").select {|f| File.directory? f}.each do |j|
    file = "#{j}/nextBuildNumber"
    File.open(file, 'w') {|f| f.write("1\n") } unless File.file?(file)
  end
  puts "Set missing nextBuildNumber for all jobs to 1"
end

desc "setup default sqs queue"
task :sqs do
  queue = ENV['queue']
  if queue
    template_name = "#{build_root}/com.base2services.jenkins.SqsBuildTrigger.xml.orig"
    file_name = "#{build_root}/com.base2services.jenkins.SqsBuildTrigger.xml"
    cfg = File.read(template_name)
    File.open(file_name, "w") {|file| file.puts cfg.gsub(/INSERT_SQS_QUEUE/, queue)}
    puts "Generated SQS config with queue=#{queue} from template file"
  else
    puts "You must specify queue=... paramter after 'rake sqs'"
  end
end

desc "install jenkins server"
task :install => ['war', 'plugins', 'updates', 'properties', 'sequence'] do
  puts "Jenkins server installation done"
end


desc "Start jenkins server"
task :start do
  puts "Starting jenkins server\n"
  httpPort = ENV['JENKINS_PORT'] || '3001'
  ajp13Port = ENV['ajp13Port'] || '-1'
  javatmp = File.join(build_root, "javatmp")
  logdir = File.join(build_root, "log")
  FileUtils.mkdir_p javatmp
  FileUtils.mkdir_p logdir
  cmd = ["java", "-XX:PermSize=512M", "-XX:MaxPermSize=512M", "-Xmn128M", "-Xms512M", "-Xmx512M", "-Djava.io.tmpdir=#{javatmp}", "-jar", WAR_LOCATION]
  cmd << "--httpPort=#{httpPort}"
  cmd << "--ajp13Port=#{ajp13Port}"
  cmd << "--logfile=#{logdir}/jenkins.log"
  puts "#{cmd.join(" ")}\n"
  system("nohup #{cmd.join(" ")} > #{logdir}/system.log 2>&1 &")
end

desc "Stop jenkins server"
task :stop do
  cmd = ["java", "-jar", CLI_LOCATION, "safe-shutdown"]
  puts "Stopping jenkins server\n"
  puts "#{cmd.join(" ")}\n"
  system(*cmd)
end

desc "Wait online jenkins server"
task :online do
  down = true
  puts "Waiting for jenkins server #{ENV['JENKINS_URL']} to come online..."
  while down
    `java -jar #{CLI_LOCATION} wait-node-online "" > /dev/null 2>&1`
    if $? == 0
      puts "Jenkins server is online"
      down = false
    else
      sleep 1
    end
  end
end

desc "Kill jenkins server"
task :kill do
  cmd = "ps -ef | grep jenkins.war | grep -v grep | awk '{print $2}'"
  pid = `#{cmd}`.chomp!
  if pid
    puts "Stopping jenkins server with pid: #{pid}"
    puts `kill #{pid}`
  else
    puts "Jenkins server not running"
  end
end

desc "Restart jenkins server"
task :server => ['kill', 'start', 'online']

namespace :build do
  desc "Build job all"
  task :all do
    cmd = ["java", "-jar", CLI_LOCATION, "build", "all"]
    puts "Build job all\n"
    system(*cmd)
  end

  desc "Build job package"
  task :package do
    cmd = ["java", "-jar", CLI_LOCATION, "build", "package", "-s", "-v", "-w"]
    puts "Build job package\n"
    system(*cmd)
  end

  task :run do
    name = ARGV.last
    cmd = ["java", "-jar", CLI_LOCATION, "build", name, "-s", "-v", "-w"]
    puts "Build job #{name}\n"
    system(*cmd)
    task name.to_sym do ; end
  end
end

desc "Build and package all jobs"
task :build => ['build:all', 'build:package']

desc "Start a build server, build all jobs and stop the server"
task :all => ['server','build','stop','kill']
