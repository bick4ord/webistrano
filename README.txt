Webistrano - Capistrano deployment the easy way

Author:
  Jonathan Weiss <jw@innerewut.de>
  
License: 
  Code: BSD, see LICENSE.txt
  Images: Right to use in their provided form in Webistrano installations. No other right granted.

Original Github Project:
  https://github.com/peritor/webistrano

############################################################################

This fork was created for a few reasons.  The first reason is that Webistrano can be very difficult to get
running without pulling in the proper changes across various github project and applying them to the master 
project, which has stopped taking pull requests.

Another reason is that I have found Webistrano can be extremely difficult to install.  I wanted to
document my process in case it will help anybody else out.

Finally, I required a source change that I couldn't find in any other branches -- the UI should
filter out capistrano tasks that have blank descriptions.  This is because my webistrano 
setup is doing enterprise deployment for a large compiled codebase and is interfacing with a 
binary repository (ivy2) using custom scripts that developers also use (ant).  I'm using capistrano only 
for its ssh remote execution, and the default capistrano tasks don't map to my world.

The following is my shell script and notes to install webistrano.

############################################################################
# installing webistrano with ruby 1.9.3 and rails 2.3.14 on Ubuntu 11.10   #
# using NginX/passenger as the server                                      #
############################################################################

#start with stuff ubuntu needs
sudo apt-get -y install git curl build-essential openssl libreadline6 libreadline6-dev zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-0 libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion mysql-server libmysqlclient-dev

#install ruby version manager
bash < <(curl -s https://rvm.beginrescueend.com/install/rvm)
source "$HOME/.rvm/scripts/rvm"

rvm install 1.9.3
source "$HOME/.rvm/scripts/rvm" && rvm use 1.9.3

gem install rubygems-update
gem update --system
gem install rake

git clone https://github.com/bick4ord/webistrano.git && cd webistrano

bundle install

cp config/database.yml.sample config/database.yml
cp config/webistrano_config.rb.sample config/webistrano_config.rb

#make sure to update the webistrano_config.rb and database.yml to match your needs - see original webistrano documentation

#this command assumes user root with empty password, which is a terrible idea for production servers
mysql -u root -e "CREATE DATABASE webistrano_production;" && mysql -u root -e "CREATE DATABASE webistrano_development;" && mysql -u root -e "CREATE DATABASE webistrano_test;"

RAILS_ENV=production rake db:migrate

#update the db to support longtext for deployments
mysql -u root -e "alter table webistrano_production.deployments change log log LONGTEXT;" webistrano_production

#install packages required for NginX/passenger
sudo apt-get install -y libcurl4-openssl-dev
gem install passenger
passenger-install-nginx-module

#note that I install NginX in the user's home directory /home/YOURUSER/nginx
#if you choose a different location, the nginx configuration must be changed
#the following creates the nginx config for webistrano on localhost:3000
echo "
worker_processes 2;

events {
    worker_connections  512;
    use epoll;
}

http {
    passenger_root ${HOME}/.rvm/gems/ruby-1.9.3-p0/gems/passenger-3.0.11;
    passenger_ruby ${HOME}/.rvm/wrappers/ruby-1.9.3-p0/ruby;

    include       mime.types;
    default_type  application/octet-stream;

    tcp_nopush      on;
    tcp_nodelay     off;
    sendfile        on;

    keepalive_timeout  120;

    server {
        listen 3000;
        server_name localhost;
        root ${HOME}/webistrano/public;   # <--- be sure to point to 'public'!
        passenger_enabled on;
   }

}
" > ~/nginx/conf/nginx.conf

#the following creates start.sh and stop.sh in the user's home directory
echo "
#!/bin/bash -e
source "${HOME}/.rvm/scripts/rvm" && rvm use 1.9.3
${HOME}/nginx/sbin/nginx
" > ~/start.sh

cat <<EOF>> ~/stop.sh
#!/bin/bash -x
killall nginx
EOF

chmod +x ~/*.sh

