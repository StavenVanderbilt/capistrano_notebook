# How to use capistrano to implement automatic deployment

## STEP-1 (Dev Machine)
  - create your project
    1. ```rvm use 2.3.1```
    2. ```rvm gemset create demo```
    3. ```rvm use 2.3.1@demo```
    4. ```gem install bundler```
    5. ```gem install rails -v 5.0.1```
    6. ```sudo apt-get install libmysqlclient-dev```
    6. ```rails new demo -d mysql```
  - configure config/database.yml

  ```
  default: &default
    adapter: mysql2
    encoding: utf-8
    pool: 5
    username: root
    passwrod: your_password
    socket: /va/run/mysqld.sock

  development:
    <<: *default
    database: demo_development

  test:
    <<: *default
    database: demo_test

  production:
    <<: *default
    database: demo_production
    username: root
    password: <%= ENV['DEMO_DATABASE_PASSEORD'] %>
  ```
  - create database for the project
  ```RAILS_ENV=development rails db:drop db:create db:migrate```

  - use scaffold to create user
  ```rails g scaffold user```

  - edit config/routes.rb to configure root_path
  ```root 'users#index' ```

  - edit .gitignore, add db/schema.rb into it
  ```db/schema.rb```

  - continue to migrate the database
  ```RAILS_ENV=development rails db:migrate```

  - start your server
  ```rails s -b 0.0.0.0 -p 3000```

  - push your code into your remote repository
  ```
  git init
  ```

  ```
  git remote add origin git@bitbucket.org:Stavenvanderbilt/demo.git
  ```

  ```
  git push -u origin master
  ```


## STEP-2 (Target Machine, as production or staging machine [Server])

  ### Create deploy user for automatic deployment
  
  - Create user 
  Prerequisites: switch root role to execute the following command
  ```
  #useradd -d /home/deploy -s /bin/bash -m deploy
  #passwd deploy
  #your_password
  ```

  - Set sudo permission for deploy
  ```
  sudo vim /etc/sudoers
  deploy    ALL=(ALL)    ALL

  ```

  ### Configure remote login via ssh

  - Install openssh-server
  ```sudo apt-get install openssh-server```

  - Edit openssh configuration
  ```vim /etc/ssh/sshd_config```

  ```
  #default port，the range from 1025 to 65536
  Port 22

  # Not Allow root user to login via ssh
  PermitRootLogin no


  #forbiden login using the password
  PasswordAuthentication no
  PermitEmptyPasswords no
  PasswordAuthentication yes

  # At the end of this configuration file, add more one line to permit
  # which user can login via ssh
  AllowUsers deploy
  ```

  - restart openssh service
  ```sudo service sshd restart```


  ### Install Mysql Client and Server
  ```
  $sudo apt-get install mysql--server-5.6
  $sudo apt-get install mysql--client-5.6
  $sudo service mysql restart
  ```

  - Modify the password for root user via mysql command
  ```
  mysql -uroot -p
  typing your password

  use mysql;
  select user,host,password from user;
  update user set password=password('your_new_password') where  user='root';
  ```

  - Configure allow to remote access
  ```
  mysql -uroot -p
  typing your password

  grant all on *.* to root@'%' identified by 'password';
  flush privileges;
  ```

  - Must restart mysqld service
  ```sudo service mysql restart```

  ### Install Nginx service

  - use the following command to install

  ```
  sudo apt-get install nginx -y
  ```

  - configure firewall

  ```
  sudo iptalbes -L 
  sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
  #sudo iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT
  sudo iptables -L --line-numbers
  sudo iptables -F
  ```

  ### Install rvm, ruby, gemset, bundler, rails

  ```
  sudo apt-get curl
  curl -L https://get.rvm.io | bash -s stable
  source ~/.rvm/scripts/rvm
  rvm install 2.3.1
  rvm use 2.3.1 --default
  rvm gemset create your_project_name
  rvm use 2.3.1@your_project_name
  gem sources list
  gem sources -r https://rubygems.org/
  gem sources -a https://gems.ruby-china.org/
  bundle config mirror.https://rubygems.org https://gems.ruby-china.org
  vim ~/.gemrc
  gem: "--no-ri --no-rdoc"
  echo "rvm_auto_reload_flag=2" > ~/.rvmrc
  gem install bundler 
  sudo apt-get install libmysqlclient-dev
  gem install mysql2 -v '0.4.5'
  apt-get install build-essential
  gem install therubyracer
  gem install rails -v 5.0.1
  ```

  ### Create directory for your project's deployment
  
  - Create deployment directory and other directories
  ```
  cd ~
  mkdir -p apps/your_project_name
  mkdir -p apps/your_project_name/shared/config

  cd apps/your_project_name
  rvm use 2.3.1
  rvm gemset create your_project_name
  rvm use 2.3.1@your_project_name

  # after execute the `cap staging deploy` command
  bundle install
  RAILS_ENV=production rails db:create
  RAILS_ENV=staging rails db:create
  ```
  - configure environemnt variable
  ```
  rake secret


  vim .bash_profile
  export SECRET_KEY_BASE=123454f5gdf1g541g3d1g3f1g3d1gdfg54dfgd12g1df5g4dfghfg43d1
  export DEMO_DATABASE_PASSEORD=your_password_for_database
  verify it whether it works
  echo $SECRET_KEY_BASE

  ENV["SECRET_KEY_BASE"]
  ENV["DEMO_DATABASE_PASSEORD"]
  
  ```

  #### Create shared/config/database.yml

  ```
  default: &default
    adapter: mysql2
    encoding: utf8
    pool: 5
    username: root
    password: your_password
    socket: /var/run/mysqld/mysqld.sock

  development:
    <<: *default
    database: demo_development

  test:
    <<: *default
    database: demo_test

  staging:
    <<: *default
    database: demo_staging

  production:
    <<: *default
    database: demo_production
    username: root
    password: <%= ENV['DEMO_DATABASE_PASSEORD'] %>
  ```

  #### Create shared/config/puma.rb

  ```
  #!/usr/bin/env puma

  app = "your_project_name"
  app_dir = File.expand_path("../../..", __FILE__)

  puts "+++++++++++++++++++++++++++++++"
  puts app_dir
  puts "+++++++++++++++++++++++++++++++"

  directory "#{app_dir}/current"
  rackup "#{app_dir}/current/config.ru"
  puts "==========================================================================="
  rails_env = ENV["RAILS_ENV"] || ENV["RACK_ENV"] || "staging"
  environment rails_env
  puts rails_env
  puts "==========================================================================="

  pidfile "#{app_dir}/shared/tmp/pids/puma.pid"
  state_path "#{app_dir}/shared/tmp/pids/puma.state"
  stdout_redirect "#{app_dir}/current/log/puma.access.log", "#{app_dir}/current/log/puma.error.log", true
  threads 0,8 
  bind "unix://#{app_dir}/shared/tmp/sockets/#{app}-puma.sock"

  preload_app!

  on_restart do
    puts 'Refreshing Gemfile'
    ENV["BUNDLE_GEMFILE"] = "#{app_dir}/current/Gemfile"
  end
  ```


  #### Create shared/config/configuration.yml

  ```
  # Official Website Version
  version: 0.0.1
  name: Demo
  ```

  #### Create shared/config/secrets.yml

  ```
  # Be sure to restart your server when you modify this file.                                                        

  # Your secret key is used for verifying the integrity of signed cookies.
  # If you change this key, all old signed cookies will become invalid!

  # Make sure the secret is at least 30 characters and all random,
  # no regular words or you'll be exposed to dictionary attacks.
  # You can use `rails secret` to generate a secure secret key.

  # Make sure the secrets in this file are kept private
  # if you're sharing your code publicly.

  development:
    secret_key_base: d16965211db142d1597c55437f476529005d29143136b0f1720391b8f7469021a00120c64c48376557ef77fb437fd115f2a340eadc59a614b1fbacf6113fb555

  test:
    secret_key_base: 2f2b82cba7e0c98d9d7525f6aaf7f499c3cc74a10012687afc57b6ec9fd7c86c33b9e8e8469ca91de3bcb430536bcf23dc123b4ca1d0e76bcdc2ded6e7a46fef

  # Do not keep production secrets in the repository,
  # instead read values from the environment.
  production:
  #  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
    secret_key_base: 2f2b82cba7e0c98d9d7525f6aaf7f499c3cc74a10012687afc57b6ec9fd7c86c33b9e8e8469ca91de3bcb430536bcf23dc123b4ca1d0e76bcdc2ded6e7a46fef

  staging:
    secret_key_base: e595fe0023b232ee4de1ec715cecaf68354ed8c7f24f03fb28967a048be0dcc5a5c61fad3170e5b4e04bdc62a2b22ea7ca259e7686bd97a71efe5bfb8b061a93

  local_production:
  #  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
    secret_key_base: 2f2b82cba7e0c98d9d7525f6aaf7f499c3cc74a10012687afc57b6ec9fd7c86c33b9e8e8469ca91de3bcb430536bcf23dc123b4ca1d0e76bcdc2ded6e7a46fef
  ```


  #### Create shared/config/nginx.conf

  ```
  vim ~/apps/your_project_name/shared/config/nginx.conf


  upstream puma {
    server unix:///home/deploy/apps/your_project_name/shared/tmp/sockets/your_project_name-puma.sock fail_timeout=0;
  }

  server {
    listen 80 deferred;
    server_name 192.168.10.129; # According to your actual situation

    root /home/deploy/apps/your_project_name/current/public;
    access_log /home/deploy/apps/your_project_name/current/log/nginx.access.log;
    error_log /home/deploy/apps/your_project_name/current/log/nginx.error.log info;


    location ^~ /assets/ {
      gzip_static on;
      expires max;
      add_header Cache-Control public;
    }

    try_files $uri/index.html $uri @puma;

    location @puma {
      proxy_pass http://puma;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect off;
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 10M;
    keepalive_timeout 10;
  }

  ```




## STEP-3 (Dev Machine)

  - add capistrano gem into Gemfile
  ```
    group :development do
      # configuration for Capistrano
      gem 'capistrano',         require: false
      gem 'capistrano-rvm',     require: false
      gem 'capistrano-rails',   require: false
      gem 'capistrano-bundler', require: false
      gem 'capistrano3-puma',   require: false
    end
  ```

  - update your gems
  ```bundle install```

  - use cap command to init capistrano
  ```bundle exec cap install```

  - edit Capfile
  ```
    require "capistrano/rvm"
    require "capistrano/bundler"
    require "capistrano/rails"
    require "capistrano/puma"
    install_plugin Capistrano::Puma  # Default puma tasks
  ```

  - edit deploy/staging.rb
  ```
    # staging server machine
    server '192.168.10.129', port: 22, user: "deploy", roles: %w{app db web}
    set :user,             "deploy"
    set :stage,            "staging"
    set :ssh_options,      { :forward_agent => true }
    set :deploy_to,        "/home/#{fetch(:user)}/apps/#{fetch(:application)}"

    set :puma_threads, [0, 4]
    set :puma_workers, 1
    set :puma_env, "#{fetch(:stage)}"
  ```

  - edit deploy/production.rb
  ```
    # production server machine
    server '192.168.10.129', port: 22, user: "deploy",  roles: %w{app db web}
    set :user,            "deploy"
    set :stage,           "production"
    set :ssh_options, { :forward_agent => true }
    set :deploy_to,       "/home/#{fetch(:user)}/apps/#{fetch(:application)}"

    set :puma_threads, [0, 8]
    set :puma_workers, 4
    set :puma_env, "#{fetch(:stage)}"

```
  
  - edit config/deploy.rb
  ```
    lock "~> 3.10.0"

    puts "=========================================================================="
    puts "===============                 deploy.rb                 ================"
    puts "=========================================================================="

    set :application,   "demo"
    set :repo_url,      "git@bitbucket.org:StavenVanderbilt/demo.git"

    set :stages,        ["staging", "production"]
    set :default_stage, "staging"

    # Default value for :linked_files is []
    append :linked_files, 'config/database.yml', 'config/puma.rb', 'config/configuration.yml', 'config/secrets.yml', 'config/nginx.conf'

    # Default value for linked_dirs is []
    append :linked_dirs, 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle'

    # use rvm gemset rather than bundle path for working with launchctl
    set :rvm_type, :user
    set :rvm_ruby_version, "2.3.1@#{fetch(:application)}"
    set :bundle_path, nil
    set :bundle_binstubs, nil
    set :bundle_flags, '--system'

    ## Defaults
    set :format,        :pretty
    set :log_level,     :debug
    set :keep_releases, 5

    set :pty,                 true
    set :use_sudo,            false
    set :puma_user,           fetch(:user)
    set :puma_state,          -> { "#{shared_path}/tmp/pids/puma.state" }
    set :puma_pid,            -> { "#{shared_path}/tmp/pids/puma.pid" }
    set :puma_bind,           -> { "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock" }
    set :puma_conf,           -> { "#{shared_path}/config/puma.rb" }
    set :puma_access_log,     -> { "#{release_path}/log/puma.access.log" }
    set :puma_error_log,      -> { "#{release_path}/log/puma.error.log" }
    set :puma_role,           :app
    set :puma_init_active_record, false
    set :puma_preload_app, true

    def branch_name(default_branch)
     branch = ENV.fetch('BRANCH', default_branch)
     if branch == '.'
       # current branch
       `git rev-parse --abbrev-ref HEAD`.chomp
     else
       branch
     end
    end

    set :branch, branch_name('master')

    namespace :nginx do
     desc 'setup config for nginx server ( how to use: cap stage_environment nginx:set_config )'
     task :set_config do
       on roles(:web) do
         execute :sudo, "rm -f /etc/nginx/sites-enabled/#{fetch(:application)}.conf"
         #execute :sudo, "ln -s #{fetch(:deploy_to)}/current/config/nginx.conf /etc/nginx/sites-enabled/#{fetch(:application)}.conf"
         execute :sudo, "ln -s #{fetch(:deploy_to)}/shared/config/nginx.conf /etc/nginx/sites-enabled/#{fetch(:application)}.conf"
         execute :sudo, "/usr/sbin/nginx -s quit || true"
         execute :sudo, "/usr/sbin/nginx"
       end
     end

     desc "Reload nginx ( how to use: cap stage_environment nginx:reload )"
     task :reload do
       on roles(:web) do
         execute :sudo, "service nginx reload"
       end
     end
    end
  ```

  - create config/nginx.conf (Optional)
  ```
    upstream puma {
      server unix:///home/deploy/apps/your_project_name/shared/tmp/sockets/your_project_name-puma.sock fail_timeout=0;
    }

    server {
      listen 80 deferred;
      server_name 192.168.10.129; #according to your actual situation

      root /home/deploy/apps/your_project_name/current/public;
      access_log /home/deploy/apps/your_project_name/current/log/nginx.access.log;
      error_log /home/deploy/apps/your_project_name/current/log/nginx.error.log info;


      location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
      }

      try_files $uri/index.html $uri @puma;

      location @puma {
        proxy_pass http://puma;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
      }

      error_page 500 502 503 504 /500.html;
      client_max_body_size 10M;
      keepalive_timeout 10;
    }

  ```

  - add all of file and push them onto repository

  ```git add .```
  ```git commit -m "complete to automatic deploy"```
  ```git push -u origin master```

  - start to deploy via cap command
  ```cap staging deploy```

  #### Target machine (install gems when fail to deploy at the first time)
  If you fail to deploy at the first time, you need to go [the target machine] to install gems
  ```
  cd ~/apps/your_project_name/release/XXXX
  rvm use 2.3.1
  rvm gemset create your_project_name
  rvm use 2.3.1@your_project_name
  gem install bundler
  sudo apt-get install libmysqlclient-dev
  apt-get install build-essential
  bundle install
  ```

  ```
  rake secret

  vim .bash_profile
  export SECRET_KEY_BASE=123454f5gdf1g541g3d1g3f1g3d1gdfg54dfgd12g1df5g4dfghfg43d1
  export DEMO_DATABASE_PASSEORD=your_password_for_database
  verify it whether it works
  echo $SECRET_KEY_BASE

  ENV["SECRET_KEY_BASE"]
  ENV["DEMO_DATABASE_PASSEORD"]
  ```

  ```
  RAILS_ENV=staging rails db:create
  RAILS_ENV=production rails db:create
  ```

  ```
  # Leave out the step of inputing the password for id_rsa.pub
  eval "$(ssh-agent -s)"
  ssh-add
  ```
  ```cap staging nginx:set_config```
  ```cap staging nginx:reload```
