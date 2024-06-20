sudo systemctl enable nginx.service
# Install


## when on vms may need -u NONE as a option for vim to keep my sanity



sudo dnf update -y

sudo dnf config-manager --set-enabled crb ## not vm
sudo dnf install -y epel-release ## not vm

sudo dnf groupinstall -y "Development Tools"

sudo dnf install -y sqlite-devel


sudo dnf install -y glib2 glib2-devel


## mongodb


## adding repos not relavent for vms
sudo vim /etc/yum.repos.d/mongodb.repo


[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc



sudo dnf update -y


sudo dnf install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod

----------



## needed to use following on vm rather than the standard dnf install

[goz24vof@v1334 ~]$ sudo dnf module install -y redis:7


sudo dnf install -y redis
sudo systemctl start redis
sudo systemctl enable redis


--------


sudo dnf module install -y nodejs:20

sudo dnf install -y yarnpkg


sudo dnf install -y libyaml-devel
sudo dnf install -y perl


sudo dnf install -y ncurses-devel
sudo dnf install -y bzip2-devel

------
# add user for expvip
sudo useradd polymarker

## or maybe due to vm stuff

sudo luseradd polymarker 


then as said user

-------


git clone https://github.com/rbenv/rbenv.git ~/.rbenv

bashrc

export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"

source .bashrc


git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build



rbenv install 2.7.8
rbenv global 2.7.8



---------


git clone https://github.com/cb2e6f/bioruby-polymarker.git

cd bioruby-polymarker/

git checkout re-migration

---------



gem install bundler:1.17.3


bundle install


yarn install

export NODE_OPTIONS=--openssl-legacy-provider
bin/rails assets:precompile

------------------
## for non-vms
[pm@pm bioruby-polymarker]$ sudo vim /etc/selinux/config 
SELINUX=disabled


-------------
# web server

# not for vms as it shoudl be there
sudo curl --fail -sSLo /etc/yum.repos.d/passenger.repo https://oss-binaries.phusionpassenger.com/yum/definitions/el-passenger.repo

sudo dnf module install -y nginx:1.24


sudo dnf install -y nginx-mod-http-passenger

sudo systemctl enable nginx.service


sudo vim /etc/nginx/conf.d/passenger.conf


----------------------

passenger_root /usr/share/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /usr/bin/ruby;
passenger_instance_registry_dir /var/run/passenger-instreg;

-----

passenger_root /usr/share/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /home/polymarker/.rbenv/shims/ruby;
passenger_instance_registry_dir /var/run/passenger-instreg;
passenger_temp_path /tmp/passenger_temp;

----------------------




sudo systemctl start nginx.service





sudo vim /etc/nginx/nginx.conf
# user nginx;
user polymarker;


------------

<         root         /usr/share/nginx/html;
---
>         root         /home/polymarker/bioruby-polymarker/public;
> 
>       passenger_enabled on;
>       passenger_app_env development; # <-- !important ????


-----------



sudo systemctl restart nginx.service




sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --list-all
sudo firewall-cmd --reload


----------------------


scp and chown data
i


cd /home/polymarker/bioruby-polymarker/public
scp -r goz24vof@n119983:~/polymarker_data/public/files .



----------------------------


bin/rails reference:add[/home/pm/pm_migration_test.yml]



### and do prefecnces


bin/rails setup:preferences[/home/pm/preferences.ini]
[polymarker@v1334 bioruby-polymarker]$ bin/rails setup:preferences[/home/polymarker/polymarker_data/preferences.ini]


[polymarker@v1334 bioruby-polymarker]$ bin/rails reference:add[/home/polymarker/polymarker_data/all_references-20240620.yaml]



------------

## install needed software

mkdir sw
cd sw/

#### or for vm

[goz24vof@v1334 ~]$ cd /data/
[goz24vof@v1334 data]$ ls
References
[goz24vof@v1334 data]$ sudo mkdir sw
[goz24vof@v1334 data]$ sudo chown goz24vof:users sw/

cd sw





wget https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/ncbi-blast-2.15.0+-3.x86_64.rpm

sudo dnf localinstall -y ./ncbi-blast-2.15.0+-3.x86_64.rpm 


wget https://mafft.cbrc.jp/alignment/software/mafft-7.526-gcc_fc6.x86_64.rpm

sudo dnf localinstall -y ./mafft-7.526-gcc_fc6.x86_64.rpm


wget https://github.com/primer3-org/primer3/archive/refs/tags/v2.6.1.tar.gz
tar -xf v2.6.1.tar.gz
cd primer3-2.6.1/src/
make
sudo make install

cd ../..


## on the vms not all the partiions allow execution so for the configure step may need to play about 
## I belive /tmp is fine

wget http://ftp.ebi.ac.uk/pub/software/vertebrategenomics/exonerate/exonerate-2.2.0.tar.gz
tar -xf exonerate-2.2.0.tar.gz 
cd exonerate-2.2.0
./configure 
make
sudo make install 



----------

# sidekiq setup


sudo vim /usr/lib/systemd/system/sidekiq.service
---------------------------------
#                                  
# This file tells systemd how to run Sidekiq as a 24/7 long-running daemon.
#                                                                                                         
# Customize this file based on your bundler location, app directory, etc.
#                                                                                                         
# If you are going to run this as a user service (or you are going to use capistrano-sidekiq)
# Customize and copy this to ~/.config/systemd/user                                                       
# Then run:                                                                                               
#   - systemctl --user enable sidekiq                                                                     
#   - systemctl --user {start,stop,restart} sidekiq                                                       
# Also you might want to run:                                                                             
#   - loginctl enable-linger username                                                                     
# So that the service is not killed when the user logs out.
#                                                                                                         
# If you are going to run this as a system service
# Customize and copy this into /usr/lib/systemd/system (CentOS) or /lib/systemd/system (Ubuntu).
# Then run:                                                                                                                                                                                                          
#   - systemctl enable sidekiq                                                                            
#   - systemctl {start,stop,restart} sidekiq
#             
# This file corresponds to a single Sidekiq process.  Add multiple copies
# to run multiple processes (sidekiq-1, sidekiq-2, etc).
#                                                                                                         
# Use `journalctl -u sidekiq -rn 100` to view the last 100 lines of log output.
#                             
[Unit]
Description=sidekiq   
# start us only once the network and logging subsystems are available,
# consider adding redis-server.service if Redis is local and systemd-managed.
After=syslog.target network.target
                                                                                                          
# See these pages for lots of options:
#                   
#   https://www.freedesktop.org/software/systemd/man/systemd.service.html
#   https://www.freedesktop.org/software/systemd/man/systemd.exec.html
#                       
# THOSE PAGES ARE CRITICAL FOR ANY LINUX DEVOPS WORK; read them multiple
# times! systemd is a critical tool for all developers to know and understand.
#                         
[Service]                                                                                                 
#                        
#      !!!!  !!!!  !!!!
#          
#      !!!!  !!!!  !!!!                                                                                                                                                                                       [0/133]
#                                                                                                                                                                                                                    
# As of v6.0.6, Sidekiq automatically supports systemd's `Type=notify` and watchdog service                                                                                                                          
# monitoring. If you are using an earlier version of Sidekiq, change this to `Type=simple`                                                                                                                           
# and remove the `WatchdogSec` line.                                                                                                                                                                                 
#                                                                                                                                                                                                                    
#      !!!!  !!!!  !!!!                                                                                                                                                                                              
#                                                                                                                                                                                                                    
Type=simple                                                                                                                                                                                                          
NotifyAccess=all        
# If your Sidekiq process locks up, systemd's watchdog will restart it within seconds.
# WatchdogSec=10                                                                                                                                                                                                     
                                                                                                                                                                                                                     
WorkingDirectory=/opt/myapp/current
# If you use rbenv:                                                                                       
ExecStart=/bin/bash -lc 'exec /home/pm/.rbenv/shims/bundle exec sidekiq'                                  
# If you use the system's ruby:                                                                           
# ExecStart=/usr/local/bin/bundle exec sidekiq -e production                                              
# If you use rvm in production without gemset and your ruby version is 2.6.5                 
# ExecStart=/home/deploy/.rvm/gems/ruby-2.6.5/wrappers/bundle exec sidekiq -e production                  
# If you use rvm in production with gemset and your ruby version is 2.6.5                                 
# ExecStart=/home/deploy/.rvm/gems/ruby-2.6.5@gemset-name/wrappers/bundle exec sidekiq -e production      
# If you use rvm in production with gemset and ruby version/gemset is specified in .ruby-version,         
# .ruby-gemsetor or .rvmrc file in the working directory                                                  
# ExecStart=/home/deploy/.rvm/bin/rvm in /opt/myapp/current do bundle exec sidekiq -e production          
                                                                                                          
# Use `systemctl kill -s TSTP sidekiq` to quiet the Sidekiq process                                       
                                                     
# Uncomment this if you are going to use this as a system service                               
# if using as a user service then leave commented out, or you will get an error trying to start the service                                                                                                          
# !!! Change this to your deploy user account if you are using this as a system service !!!               
# User=deploy                               
# Group=deploy
# UMask=0002                                                                                              
                                                                                                          
# Greatly reduce Ruby memory fragmentation and heap usage                                                 
# https://www.mikeperham.com/2018/04/25/taming-rails-memory-bloat/             
Environment=MALLOC_ARENA_MAX=2
                                                     
# if we crash, restart
RestartSec=1                                                                                              
Restart=always                                                                                            
                                                     
# output goes to /var/log/syslog (Ubuntu) or /var/log/messages (CentOS)                                   
StandardOutput=syslog                 
StandardError=syslog
                                                                                                          
# This will default to "bundler" if we don't specify it               
SyslogIdentifier=sidekiq
                                                                                                          
[Install]                                                                                                 
WantedBy=multi-user.target
# Uncomment this and remove the line above if you are going to use this as a user service                 
# WantedBy=default.target

------------------------

sudo systemctl start sidekiq
sudo systemctl enable sidekiq



-------------------

# may be add some notes on selinux in case that is relavent for an install

-----------------



# there is also the cron job for restarting sidekiq if the current issue with some jobs failing and filling the queue have not yet been resolved.





@hourly /usr/sbin/service sidekiq restart







# db bk and restore



[root@v0894 data]# mongodump --db bioruby_polymarker_development --collection preferences --archive="polymarker_preferences_dump-20240620.gz" --gzip






[root@v0894 data]# mongodump --db bioruby_polymarker_development --collection snp_files --archive="polymarker_snp_files_dump-20240620.gz" --gzip


[root@v0894 data]# mongodump --db bioruby_polymarker_development --collection references --archive="polymarker_references_dump-20240620.gz" --gzip




### sidekiq???

[polymarker@v1334 ~]$ gem install sidekiq
[polymarker@v1334 ~]$ cd bioruby-polymarker/
[root@v1334 bioruby-polymarker]# cp examples/sidekiq.service /usr/lib/systemd/system/



sudo systemctl start sidekiq
sudo systemctl enable sidekiq




##### files??


[goz24vof@v1334 ~]$ cd /data/
[goz24vof@v1334 data]$ ls
References  sw
[goz24vof@v1334 data]$ sudo mkdir tmp
[goz24vof@v1334 data]$ sudo chown -R polymarker:users tmp/











