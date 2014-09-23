
#This is a set of commands used to install gatewayd, ripple-coins and ripple-rest on ubuntu 14.04.1 Server
#It grew out of an effort to create a dockerfile and docker image of the same
#As such, some commentary may not make sense in the context of someone just trying to install gatewayd.
#IMPORTANT NOTE: Most of this you can just copypaste and run. Do it slowly, as some lines will stop execution
#(like sudo apt-get install -y bunchofstuff).

#Finally: This is NOT a script. If you try to run it as such, it will stop when you change users. Start as root
#and just run the commands.

##BEFORE YOU BEGIN!
#Download the ssl script and "sudo cp <filename> /etc/init.d/ssl"
#then "sudo chmod +x /etc/init.d/ssl"
#Alternatively, use your own keys. If you intend to use this "script" as-is, though,
#you MUST have that and it MUST be executable so keys for your ripple-rest ssl can be generated.
#The irony here is you can't *enable* ssl or gatewayd will break (for now).


#Milestone(MS) index:
#Milestone 1: system dependencies installed
#Milestone 2: postgres configured, gatewayd installed and configured
#Milestone 3: ripple-rest installed, configured
#Milestone 3.2: startup scripts installed
#Milestone 3.3: gatewayd additional config (finish, move to MS2)

#Must begin as root. Very important.
#By default, root disabled in Ubuntu. Enable by setting root pw.
#FIXME: change this whole document to sudo instead.

sudo passwd
#<enter passwd 2x>
su root
#<enter passwd>
cd /

##USERS
#Define password generator, create user pw
useradd -U -m -r -s /dev/null restful
useradd -U -m -s /bin/bash shell_user_gatewayd
adduser shell_user_gatewayd sudo

randpw(){ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-16};echo;}
shell_user_gatewaydPW=`randpw 20`
echo "shell_user_gatewayd:$shell_user_gatewaydPW" | chpasswd
export SHELL_USER_GATEWAYDPW=$shell_user_gatewaydPW
echo "pw=$shell_user_gatewaydPW"
su shell_user_gatewayd
cd ~

##DEPENDENCIES
#Update the repository sources list
echo "$SHELL_USER_GATEWAYDPW" | sudo -S apt-get update
echo "pw=$SHELL_USER_GATEWAYDPW"

#unset SHELL_USER_GATEWAYDPW
#THIS WIPES THE PASSWORD FROM SAVED ENVIRONMENT VARIABLE. Careful.

sudo apt-get install -y git python-software-properties python g++ make libpq-dev software-properties-common postgresql postgresql-client
#WARNING!WARNING!WARNING!WARNING!WARNING!WARNING!WARNING!
#paste-execution WILL STOP HERE. Pay attention or you'll skip dependencies.

# Add Node.js Repository for 0.1, update, install
sudo add-apt-repository -y ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get -y install nodejs
#WARNING!WARNING!WARNING!WARNING!WARNING!WARNING!WARNING!
#paste-execution WILL STOP HERE. Pay attention or you'll skip dependencies.

##Milestone 1: system dependencies installed
#make sure SSL script is copied over and made executable
#also set or export SHELL_USER_GATEWAYDPW

cd ~
git clone https://github.com/ripple/gatewayd.git
cd gatewayd/
git checkout cf1281d
#aka v3.27.0

#INSTALL gatewayd dependencies, pm2 separately, save
sudo npm install --global pg grunt grunt-cli forever db-migrate jshint pm2@0.8.15
sudo npm install --save


#CONFIGURE postgres, users, DBs
#generate postgresql passwords
randpw(){ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-16};echo;}
db_user_postgresPW=`randpw 20`
db_user_gatewaydPW=`randpw 20`
db_user_ripple_restPW=`randpw 20`

sudo service postgresql start
sudo su - postgres -c "psql -c \"ALTER USER postgres WITH PASSWORD '$db_user_postgresPW';\""
#change postgres template: http://stackoverflow.com/questions/16736891/pgerror-error-new-encoding-utf8-is-incompatible
sudo su - postgres -c "psql -c \"UPDATE pg_database SET datistemplate = FALSE WHERE datname = 'template1';\""
sudo su - postgres -c "psql -c \"DROP DATABASE template1;\""
sudo su - postgres -c "psql -c \"CREATE DATABASE template1 WITH TEMPLATE = template0 ENCODING = 'UNICODE';\""
sudo su - postgres -c "psql -c \"UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'template1';\""
sudo su - postgres -c "psql -c \"\\c template1\""
sudo su - postgres -c "psql -c \"VACUUM FREEZE;\""

sudo su - postgres -c "psql -c \"create user db_user_ripple_rest with password '$db_user_ripple_restPW';\""
sudo su - postgres -c "psql -c \"create database ripple_rest_db with owner db_user_ripple_rest encoding='utf8';\""
sudo su - postgres -c "psql -c \"create user db_user_gatewayd with password '$db_user_gatewaydPW';\""
sudo su - postgres -c "psql -c \"create database gatewayd_db with owner db_user_gatewayd encoding='utf8';\""

export DATABASE_URL=postgres://db_user_ripple_rest:$db_user_ripple_restPW@localhost:5432/ripple_rest_db?native=true

#Set users, passwords, DBs in configs
sed -i "s/postgres:password/db_user_gatewayd:$db_user_gatewaydPW/g" ./config/config.js
sed -i "s/\/ripple_gateway/\/gatewayd_db/g" ./config/config.js
sed -i "s/@localhost:5432\/gatewayd_db/@localhost:5432\/gatewayd_db?native=true/g" ./config/config.js
cp lib/data/database.example.json lib/data/database.json
sed -i "s/DATABASE_URL/postgres:\/\/db_user_gatewayd:$db_user_gatewaydPW@localhost:5432\/gatewayd_db/g" ./lib/data/database.json

grunt migrate

##MILESTONE2: GWD AND POSTGRES INSTALLED AND CONFIGURED

#INSTALL RIPPLE-REST
git clone https://github.com/ripple/ripple-rest.git
cd ripple-rest
git checkout 501dba7
#aka v1.2.5

#CONFIGURE ripple-rest
#store pw in config
cp config-example.json config.json
sed -i "s/ripple_rest_user:password/db_user_ripple_rest:$db_user_ripple_restPW/g" ./config.json
sed -i "s/@localhost:5432\/ripple_rest_db/@localhost:5432\/ripple_rest_db?native=true/g" ./config.json

#set key file and path
sed -i "s/.\/certs\/server.key/\/etc\/ssl\/server.key/g" ./config.json
sed -i "s/.\/certs\/server.crt/\/etc\/ssl\/server.crt/g" ./config.json

#INSTALL dependencies, run migrations
sudo npm install --global grunt grunt-cli pg
sudo npm install
npm install --save pg

#create SSL certificates
sudo /etc/init.d/ssl start

##MILESTONE 3! FULL RIPPLE-REST INSTALLATION!!! WOO!

#CREATE ripple-rest startup script
echo '#!/bin/sh' > ~/start-rest.sh
echo "sudo service postgresql start" >> ~/start-rest.sh
echo "cd /home/shell_user_gatewayd/gatewayd/ripple-rest" >> ~/start-rest.sh
echo "export DATABASE_URL=postgres://db_user_ripple_rest:$db_user_ripple_restPW@localhost:5432/ripple_rest_db?native=true" >> ~/start-rest.sh
echo "sudo -E -u restful /usr/bin/node server.js &" >> ~/start-rest.sh
chmod +x ~/start-rest.sh
sudo cp ~/start-rest.sh /usr/bin/start-rest && rm ~/start-rest.sh

#CREATE gatewayd startup script
echo '#!/bin/sh' > ~/start-gatewayd.sh
echo "cd ~/gatewayd" >> ~/start-gatewayd.sh
echo "export DATABASE_URL=postgres://db_user_gatewayd:$db_user_gatewaydPW@localhost:5432/gatewayd_db?native=true" >> ~/start-gatewayd.sh
echo "bin/gateway start &" >> ~/start-gatewayd.sh
chmod +x ~/start-gatewayd.sh
sudo cp ~/start-gatewayd.sh /usr/bin/start-gatewayd && rm ~/start-gatewayd.sh

#CREATE start-all script
echo '#!/bin/sh' > ~/start-all.sh
echo "start-rest &" >> ~/start-all.sh
echo "sleep 10" >> ~/start-all.sh
echo "start-gatewayd &" >> ~/start-all.sh
chmod +x ~/start-all.sh
sudo cp ~/start-all.sh /usr/bin/start-all && rm ~/start-all.sh

##MILESTONE 3.2 Startup scripts installed

#CONFIGURE gatewayd, add wallets, currencies
#When ready, move this up to gatewayd install section
cd ~/gatewayd
bin/gateway add_currency USD
bin/gateway add_currency BTC
bin/gateway add_currency LTC
bin/gateway add_currency DOGE
bin/gateway add_currency PHC
bin/gateway set_cold_wallet rhj3RL3SdxYVm8a7TXc7mbK2WoUBNMVuxz

bin/gateway generate_wallet
export DATABASE_URL=postgres://db_user_gatewayd:$db_user_gatewaydPW@localhost:5432/gatewayd_db
#cold wallet address=rhj3RL3SdxYVm8a7TXc7mbK2WoUBNMVuxz secret=ssN5m279DWhFw5KjFVBh2ijMwigPc
#hot wallet:  address=rNXW9BmqufSRiZ5gUXMGmNFev3s8Lup4P3, secret=ssowTc8ba2PG9ADTxuzsD4TBzs34M
#gatewayd must be running
bin/gateway set_hot_wallet rNXW9BmqufSRiZ5gUXMGmNFev3s8Lup4P3 ssowTc8ba2PG9ADTxuzsD4TBzs34M

#For obvious reasons, SET YOUR OWN ADDRESSES AND SECRETS! These are examples.
#NOTE: NEED TO GENERATE THESE DYNAMICALLY

##MILESTONE 3.3 additional gatewayd configuration done! (NOT FINISHED)
##ISSUES:
#sed needs better regex so it can change existing passwords
#(just make a configurator script)