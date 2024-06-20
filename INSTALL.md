# Install

Notes on installing polymarker's web interface.

These instructions should be enough to get a working copy of polymarker
running on a fresh install of almalinux 9, some of these steps are not needed
when installing to the VMs I have set up so I will try to note when this is
the case.

## update & enable repos

First we install any updates and enable some repos that we need.

> :note: **If running on one of our managed VMs**: This step is not needed

    sudo dnf update -y
    sudo dnf config-manager --set-enabled crb
    sudo dnf install -y epel-release

## dependencies

Next we need to on stall our dependencies.

First is development tools:

    sudo dnf groupinstall -y "Development Tools"

### mongodb

polymarker also uses mongodb for storing expression values. On a fresh install of
almalinux we need to add the mongodb repo:

> :note: **If running on one of our managed VMs**: adding the repo is not needed as
> it is already provided.

Create the file: /etc/yum.repos.d/mongodb.repo with the following content:

    [mongodb-org-6.0]
    name=MongoDB Repository
    baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
    gpgcheck=1
    enabled=1
    gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc

Then we can install and enable mongodb:

    sudo dnf install -y mongodb-org
    sudo systemctl start mongod
    sudo systemctl enable mongod


## redis

install and enable redis, used by sidekiq to run jobs.

> :note: **If running on one of our managed VMs**: install redis with:
> 
> sudo dnf module install -y redis:7

    sudo dnf install -y redis
    sudo systemctl start redis
    sudo systemctl enable redis


### node

For some javascript goodness we install node and yarn:

    sudo dnf module install -y nodejs:20
    sudo dnf install -y yarnpkg

### ruby dependencies

The follow can be needed for building ruby:

    sudo dnf install -y libyaml-devel
    sudo dnf install -y perl
    
### misc dependencies

I am afraid that the only one of these that I remember the needing component 
for is rails needing sqlite-devel, the others I guess are needed by the 
various applications that polymarker uses.

    sudo dnf install -y sqlite-devel
    sudo dnf install -y glib2 glib2-devel
    sudo dnf install -y ncurses-devel
    sudo dnf install -y bzip2-devel

### nginx & passenger

For serving polymarker we use nginx in conjunction with passenger, so we need to
add the repo for passenger:

> :note: **If running on one of our managed VMs**: adding the repo is not needed as
> it is already provided.

    sudo curl --fail -sSLo /etc/yum.repos.d/passenger.repo https://oss-binaries.phusionpassenger.com/yum/definitions/el-passenger.repo

We can then install:

    sudo dnf module install -y nginx:1.24
    sudo dnf install -y nginx-mod-http-passenger

Now we need to configure nginx and passenger for our setup, first we need to
add or change the following entries in: /etc/nginx/conf.d/passenger.conf to
tell passenger where to find our ruby install:

    passenger_ruby /home/polymarker/.rbenv/shims/ruby;
    passenger_temp_path /tmp/passenger_temp;

We then need to tell nginx to use passenger and where to find polymarker, in:
/etc/nginx/nginx.conf we change the the ngnix user to polymarker (which we will
create shortly):

    <   user nginx;
    ---
    >   user polymarker;

Then we set root to polymarker's public directory and add the configuration
for passenger:

    <   root /usr/share/nginx/html;
    ---
    >   root /home/polymarker/bioruby-polymarker/public;
    >
    > 	passenger_enabled on;
    > 	passenger_app_env development;

We can then start and enable nginx:

    sudo systemctl start nginx.service
    sudo systemctl enable nginx.service

### firewall

We need to change our firewall settings to allow access to our webserver with:

    sudo firewall-cmd --permanent --add-service=http
    sudo firewall-cmd --permanent --list-all
    sudo firewall-cmd --reload

## selinux

If we are a system that doesn't want to be running selinux then we should 
disable it by changing SELINUX in /etc/selinux/config to disabled:

    SELINUX=disabled

## polymarker dependencies

polymarker requires MAFFT, primer3, exonerate and blast to be installed on the 
system so lets install them:

Create a directory to store these:

    mkdir sw
    cd sw/

### blast

    wget https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/ncbi-blast-2.15.0+-3.x86_64.rpm
    sudo dnf localinstall -y ./ncbi-blast-2.15.0+-3.x86_64.rpm
    
### mafft

    wget https://mafft.cbrc.jp/alignment/software/mafft-7.526-gcc_fc6.x86_64.rpm
    sudo dnf localinstall -y ./mafft-7.526-gcc_fc6.x86_64.rpm

### primer3

    wget https://github.com/primer3-org/primer3/archive/refs/tags/v2.6.1.tar.gz
    tar -xf v2.6.1.tar.gz
    cd primer3-2.6.1/src/
    make
    sudo make install
    cd ../..

### exonerate

    wget http://ftp.ebi.ac.uk/pub/software/vertebrategenomics/exonerate/exonerate-2.2.0.tar.gz
    tar -xf exonerate-2.2.0.tar.gz
    cd exonerate-2.2.0
    ./configure
    make
    sudo make install


> :note: **If running on one of our managed VMs**: some volumes will not allow files
> stored on them to be executed preventing installing some of the above, I have 
> found that moving files to /tmp can get around this. 

## polymarker user

We add a user to the system to run things as:

    sudo luseradd polymarker

Unless specified otherwise this is the user for running any following commands.

## install ruby

Now as our polymarker user we can install rbenv & ruby-build and to manage our
ruby versions.

    git clone https://github.com/rbenv/rbenv.git ~/.rbenv
    echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
    echo 'eval "$(rbenv init -)"' >> ~/.bashrc
    echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build
    rbenv install 2.7.8
    rbenv global 2.7.8

## polymarker

Get polymarker's web interface code:

    git clone https://github.com/cb2e6f/bioruby-polymarker.git

Install our ruby gem dependencies:

    cd bioruby-polymarker/
    gem install bundler:1.17.3
    bundle install

Install our javascript dependencies:
    
    export NODE_OPTIONS=--openssl-legacy-provider
    yarn install
    bin/rails assets:precompile

## sidekiq

Install sidekiq so the web interface can run instances of polymarker:

    gem install sidekiq

And as a user that can escalate to root privileges install systemd service 
file and enable:    

    sudo cp /home/polymarker/bioruby-polymarker/examples/sidekiq.service /usr/lib/systemd/system/
    sudo systemctl start sidekiq
    sudo systemctl enable sidekiq

## data

### Reference

polymarker's reference data is stored at /data

    [goz24vof@v1334 ~]$ ls /data/References/
    AST_PRJEB5043_v1  Bo  EI  Hv         Rye  Td          Tu
    Bn                Br  Gm  RefSeq1.0  Ta   Tetraploid


### assets

Once again polymarker's web interface servers some larger binary files not suited to storage in git.
as such these need copying in to place e.g.

    Untracked files:
      (use "git add <file>..." to include in what will be committed)
    	config/credentials.yml.enc
    	public/files/

    no changes added to commit (use "git add" and/or "git commit -a")
    [polymarker@v1334 bioruby-polymarker]$ ls public/files/
    820k_axiom  iSelect

### configuration

We then need to load our references and preferences in to the app with 
something akin to the following:

    bin/rails setup:preferences[/home/polymarker/polymarker_data/preferences.ini]
    bin/rails reference:add[/home/polymarker/polymarker_data/all_references-20240620.yaml]

### email

If we want email notifications functional we need to add a mail_properties.yml 
file to the config directory with the relevant settings for our mail server. 

## sidekiq queue issue

Currently there is some issue with jobs failing and blocking the queue, in 
order to work around this there is a cron job for restarting sidekiq every 
hour:

    crontab -e

    @hourly /usr/sbin/service sidekiq restart
