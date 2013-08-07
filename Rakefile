#!/usr/bin/env rake

require 'mysql2'
require 'open-uri'
require 'erb'

desc "Create the database if it doesn't exist"
task :createdb do
	puts "CREATING DATABASE"
	begin
		client = Mysql2::Client.new(
			host: ENV["RDSHOST"],
			username: ENV["DBUSER"],
			password: ENV["DBPASS"],
			database: ENV['DBNAME'])

		puts "Database already exists. Skipping!"
	rescue
		puts "Creating Database!"
		client = Mysql2::Client.new(
			host: ENV["RDSHOST"],
			username: ENV["RDSROOTUSER"],
			password: ENV["RDSROOTPASS"])
		client.query("CREATE DATABASE IF NOT EXISTS #{ENV['DBNAME']}")
		client.query("GRANT ALL ON `#{ENV['DBNAME']}`.* TO '#{ENV['DBUSER']}'@'%' IDENTIFIED BY '#{ENV['DBPASS']}'")
	end	
end

task :writeconf do
	@conffiles.each do |conf|
		f = IO.read(File.expand_path(conf[:from], __FILE__))
		conf[:replace].each do |r|
			puts r
			f[r[:string]] = r[:with]
		end
		if conf[:to].nil?
			tofile = File.expand_path(conf[:from], __FILE__)
		else
			tofile = File.expand_path(conf[:to], __FILE__)
		end
		t = File.open(tofile, 'w')
		t.write(f)
		t.close
	end
end

task :setupenv do
	sh %{ cap deploy:setup } if ENV['RUNSETUP']
end

task :deploy => [:writeconf, :setupenv] do
	sh %{ cap deploy }
	Rake::Task[:unlockconf].invoke if ENV['RUNSETUP']
end

task :unlockconf do
	sh %{ cap drupal:allow_settings_write }
end

desc "Do tasks specific to Drupal installations"
task :drupal => [:createdb] do
	@conffiles = [
		{
			from: '../sites/default/default.settings.php',
			to: '../sites/default/settings.php',
			replace: [
				{ string: '$DBHOST$', with: ENV['RDSHOST'] },
				{ string: '$DBNAME$', with: ENV['DBNAME'] },
				{ string: '$DBUSER$', with: ENV['DBUSER'] },
				{ string: '$DBPASS$', with: ENV['DBPASS'] }
			]
		}
	] 
	Rake::Task[:deploy].invoke
end

task :wordpress => 'wordpress:deploy'

namespace :wordpress do
	
	task :setup => [:createdb] do
		@builddir = File.join(ENV['WORKSPACE'], 'build')
		Dir.mkdir(@builddir) unless Dir.exists?(@builddir)
		if ENV['RUNSETUP'] and ENV['RUNSETUP'] == true
			sh %{ cap deploy:setup } 
			@saltlist = URI.parse("https://api.wordpress.org/secret-key/1.1/salt/").read
			template = File.read(File.expand_path("../wp-config.php.erb", __FILE__))
			renderer = ERB.new(template)
			result = renderer.result(binding)

			File.open(File.expand_path("../build/wp-config.php", __FILE__), 'w') {|f| f.write(result)}
			sh %{ cap wordpress:upload_config }
		end
	end

	task :fetch_cap do
		#something
	end

	task :deploy => [:fetch_cap, :setup] do
		puts "In deploy"
		sh %{ cap deploy }
	end
end