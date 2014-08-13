![image](http://rubyonrails.org/images/rails.png) ![image](http://res.cloudinary.com/iashanmugavel/image/upload/v1407842093/deploy_plzdn2.png)

####Ruby on Rails Application Hosting in [Digital Ocean](https://www.digitalocean.com/?refcode=98ca6c54ec99)  using using [Nginx](http://wiki.nginx.org/Main), [Unicorn](https://github.com/defunkt/unicorn) and [Capistrano](http://capistranorb.com/).

Create a (Ubuntu 14.04 x64) droplet in Digital Ocean.

ssh to root in terminal with your server ip

Access your droplet server IP in system terminal using ssh

	ssh root@123.123.123.123

Add ssh fingerprint and enter password provided Change password

	passwd

Create new user

	adduser username

Set new user privileges

	visudo

Find user privileges section

	#User privilege specification
	root  ALL=(ALL:ALL) ALL

Add your new user privileges under root & cntrl+x then y to save

	username ALL=(ALL:ALL) ALL

Configure SSH

	nano /etc/ssh/sshd_config

Find and change port to one that isn't default(22 is default: choose between 1025..65536)

	Port 22 # change this to whatever port you wish to use
	Protocol 2
	.
	.
	PermitRootLogin no

Reload ssh

	reload ssh

Open new shell and ssh to vps with new username(remember the port, or you're locked out!)

	ssh -p 22 username@123.123.123.123

Update packages on virtual server

	sudo apt-get update
	sudo apt-get install curl


Install latest stable version of rvm

	curl -L get.rvm.io | bash -s stable

Load rvm

	source ~/.rvm/scripts/rvm

Install rvm dependencies

	rvm requirements

Install ruby 2.1.2

	rvm install 2.1.2

Use 2.1.2 as rvm default

	rvm use 2.1.2 --default

Install latest version of rubygems if rvm install didn't

	rvm rubygems current

Install rails gem

	gem install rails --no-ri --no-rdoc


Install postgres

	sudo apt-get install postgresql postgresql-server-dev-9.3
	gem install pg -- --with-pg-config=/usr/bin/pg_config

Create new postgres user

	sudo -u postgres psql
	create user username with password 'password';
	alter role username superuser createrole createdb replication;
	create database projectname_production owner username;

Install git-core

	sudo apt-get install git-core

setup nginx

	sudo apt-get install nginx
	nginx -h
	cat /etc/init.d/nginx
	/etc/init.d/nginx -h
	sudo service nginx start
	cd /etc/nginx

Create a Rails Application in your local which is going to deploy and push your codes into github
	
	rails new projectname 
	touch README.md
	git init
	git add README.md
	git commit -m "first commit"
	git remote add origin git@github.com:*usename*/*repository*.git

Add `unicorn` gem in `Gemfile`
	
	gem 'unicorn'

	cd projectname
	bundle install

create three files in `config` directory, `unicorn.rb`, `unicorn_init.sh` and `nginx.conf`.

	cd config
	touch unicorn.rb
	touch unicorn_init.sh
	touch nginx.conf
	chmod +x config/unicorn_init.sh

Add the following code snippet in `config/nginx.conf` (change projectname and username to match your directory structure!) (also be aware of `client_max_body_size setting`, please look at [nginx documentation](http://nginx.org/en/docs/) for more information!)

	listen 80 default_server deferred;
  	# server_name example.com;
  	root /home/username/apps/projectname/current/public;

  	location ^~ /assets/ {
    	gzip_static on;
    	expires max;
    	add_header Cache-Control public;
  	}

  	try_files $uri/index.html $uri @unicorn;
  	location @unicorn {
    	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    	proxy_set_header Host $http_host;
    	proxy_redirect off;
    	proxy_pass http://unicorn;
  	}

  	error_page 500 502 503 504 /500.html;
  	client_max_body_size 20M;
  	keepalive_timeout 10;
	}


then, in `config/unicorn.rb`

	root = "/home/username/apps/projectname/current"
	working_directory root
	pid "#{root}/tmp/pids/unicorn.pid"
	stderr_path "#{root}/log/unicorn.log"
	stdout_path "#{root}/log/unicorn.log"

	listen "/tmp/unicorn.projectname.sock"
	worker_processes 2
	timeout 30

	# Force the bundler gemfile environment variable to
	# reference the capistrano "current" symlink
	before_exec do |_|
	  ENV["BUNDLE_GEMFILE"] = File.join(root, 'Gemfile')
	end


and then, in `config/unicorn_init.sh`

	#!/bin/sh
	### BEGIN INIT INFO
	# Provides:          unicorn
	# Required-Start:    $remote_fs $syslog
	# Required-Stop:     $remote_fs $syslog
	# Default-Start:     2 3 4 5
	# Default-Stop:      0 1 6
	# Short-Description: Manage unicorn server
	# Description:       Start, stop, restart unicorn server for a specific application.
	### END INIT INFO
	set -e

	# Feel free to change any of the following variables for your app:
	TIMEOUT=${TIMEOUT-60}
	APP_ROOT=/home/username/apps/projectname/current
	PID=$APP_ROOT/tmp/pids/unicorn.pid
	CMD="cd $APP_ROOT; bundle exec unicorn -D -c $APP_ROOT/config/unicorn.rb -E production"
	AS_USER=username
	set -u

	OLD_PIN="$PID.oldbin"

	sig () {
	  test -s "$PID" && kill -$1 `cat $PID`
	}

	oldsig () {
	  test -s $OLD_PIN && kill -$1 `cat $OLD_PIN`
	}

	run () {
	  if [ "$(id -un)" = "$AS_USER" ]; then
	    eval $1
	  else
	    su -c "$1" - $AS_USER
	  fi
	}

	case "$1" in
	start)
	  sig 0 && echo >&2 "Already running" && exit 0
	  run "$CMD"
	  ;;
	stop)
	  sig QUIT && exit 0
	  echo >&2 "Not running"
	  ;;
	force-stop)
	  sig TERM && exit 0
	  echo >&2 "Not running"
	  ;;
	restart|reload)
	  sig HUP && echo reloaded OK && exit 0
	  echo >&2 "Couldn't reload, starting '$CMD' instead"
	  run "$CMD"
	  ;;
	upgrade)
	  if sig USR2 && sleep 2 && sig 0 && oldsig QUIT
	  then
	    n=$TIMEOUT
	    while test -s $OLD_PIN && test $n -ge 0
	    do
	      printf '.' && sleep 1 && n=$(( $n - 1 ))
	    done
	    echo

	    if test $n -lt 0 && test -s $OLD_PIN
	    then
	      echo >&2 "$OLD_PIN still exists after $TIMEOUT seconds"
	      exit 1
	    fi
	    exit 0
	  fi
	  echo >&2 "Couldn't upgrade, starting '$CMD' instead"
	  run "$CMD"
	  ;;
	reopen-logs)
	  sig USR1
	  ;;
	*)
	  echo >&2 "Usage: $0 <start|stop|restart|upgrade|force-stop|reopen-logs>"
	  exit 1
	  ;;
	esac

Add capistrano and rvm capistrano into `Gemfile`

	gem 'capistrano'
	gem 'rvm-capistrano'

Create capfile & config/deploy.rb files

	capify .

In `deploy.rb`

	require "bundler/capistrano"
	require "rvm/capistrano"

	server "123.123.123.123", :web, :app, :db, primary: true

	set :application, "projectname"
	set :user, "username"
	set :port, 22
	set :deploy_to, "/home/#{user}/apps/#{application}"
	set :deploy_via, :remote_cache
	set :use_sudo, false

	set :scm, "git"
	set :repository, "git@github.com:username/#{application}.git"
	set :branch, "master"


	default_run_options[:pty] = true
	ssh_options[:forward_agent] = true

	after "deploy", "deploy:cleanup" # keep only the last 5 releases

	namespace :deploy do
	  %w[start stop restart].each do |command|
	    desc "#{command} unicorn server"
	    task command, roles: :app, except: {no_release: true} do
	      run "/etc/init.d/unicorn_#{application} #{command}"
	    end
	  end

	  task :setup_config, roles: :app do
	    sudo "ln -nfs #{current_path}/config/nginx.conf /etc/nginx/sites-enabled/#{application}"
	    sudo "ln -nfs #{current_path}/config/unicorn_init.sh /etc/init.d/unicorn_#{application}"
	    run "mkdir -p #{shared_path}/config"
	    put File.read("config/database.example.yml"), "#{shared_path}/config/database.yml"
	    puts "Now edit the config files in #{shared_path}."
	  end
	  after "deploy:setup", "deploy:setup_config"

	  task :symlink_config, roles: :app do
	    run "ln -nfs #{shared_path}/config/database.yml #{release_path}/config/database.yml"
	  end
	  after "deploy:finalize_update", "deploy:symlink_config"

	  desc "Make sure local git is in sync with remote."
	  task :check_revision, roles: :web do
	    unless `git rev-parse HEAD` == `git rev-parse origin/master`
	      puts "WARNING: HEAD is not the same as origin/master"
	      puts "Run `git push` to sync changes."
	      exit
	    end
	  end
	  before "deploy", "deploy:check_revision"
	end


Add in `Capfile` file

	load 'deploy'
	load 'deploy/assets'
	load 'config/deploy'

Shake hands with github

	# follow the steps in this guide if receive permission denied(public key)
	# https://help.github.com/articles/error-permission-denied-publickey
	ssh github@github.com

Add ssh key to digitalocean

	cat ~/.ssh/id_rsa.pub | ssh -p 22 username@123.123.123.123 'cat >> ~/.ssh/authorized_keys'

Push all the codes into github

	git add .
	git commit -m "added deployment configuration"
	git push -u origin master


---------------------