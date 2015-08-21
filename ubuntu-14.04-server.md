
This is a set of commands used to install gatewayd, ripple-coins and ripple-rest on ubuntu 14.04.1 Server
It grew out of an effort to create a dockerfile and docker image of the same
As such, some commentary may not make sense in the context of someone just trying to install gatewayd.
IMPORTANT NOTE: Most of this you can just copypaste and run. Do it slowly, as some lines will stop execution
(like sudo apt-get install -y bunchofstuff).

Finally: This is NOT a script. If you try to run it as such, it will stop when you change users. Start as non-root user and just run the commands.

##BEFORE YOU BEGIN!
Download the ssl script (or just clone this repo)
```
sudo cp ssl.sh /etc/init.d/ssl
sudo chmod +x /etc/init.d/ssl
```

Alternatively, use your own keys. If you intend to use this "script" as-is, though,
you MUST have that and it MUST be executable so keys for your ripple-rest ssl can be generated.
SSL: updates soon to enable ssl

##CREATE USERS
Define password generator, create user pw
```
sudo useradd -U -m -r -s /dev/null restful
sudo useradd -U -m -s /bin/bash shell_user_gatewayd
sudo adduser shell_user_gatewayd sudo

randpw(){ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-16};echo;}
shell_user_gatewaydPW=`randpw 20`
echo "shell_user_gatewayd:$shell_user_gatewaydPW" | sudo chpasswd
#above command untested, ask @jzlcdh about his experience
export SHELL_USER_GATEWAYDPW=$shell_user_gatewaydPW
#BROKEN FIXME exporting variable does not export.
#Workaround: print the password, copy and use as necessary after changing to user shell_user_gatewayd
echo "pw=$shell_user_gatewaydPW"
sudo su shell_user_gatewayd
cd ~
```

##INSTALL DEPENDENCIES
Update the repository sources list
```
echo "$SHELL_USER_GATEWAYDPW" | sudo -S apt-get update
echo "pw=$SHELL_USER_GATEWAYDPW"

#unset SHELL_USER_GATEWAYDPW
#^THIS WIPES THE PASSWORD FROM SAVED ENVIRONMENT VARIABLE. Careful.

sudo apt-get install -y git python-software-properties python g++ make libpq-dev software-properties-common postgresql postgresql-client

#Add Node.js Repository for 0.1, update, install
sudo add-apt-repository -y ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get -y install nodejs

#Eliminate need for sudo for npm global package installations.
#Further reading:
#http://stackoverflow.com/questions/19352976/npm-modules-wont-install-globally-without-sudo
#http://stackoverflow.com/questions/18212175/npm-yeoman-install-generator-angular-without-sudo/18277225#18277225
sudo npm config set prefix ~/npm
echo 'export PATH="$PATH:$HOME/npm/bin"' >> ~/.bashrc
export PATH="$PATH:$HOME/npm/bin"
```
##INSTALL AND CONFIGURE GATEWAYD AND POSTGRES

```
cd ~
git clone https://github.com/RoastBeefSandwichCo/gatewayd.git
#TODO: release tag

#INSTALL gatewayd dependencies, pm2 separately, save
npm install --global pg grunt grunt-cli forever db-migrate jshint pm2@0.8.15
npm install --save

#CONFIGURE postgres, users, DBs
#generate postgresql passwords
randpw(){ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-16};echo;}
db_user_postgresPW=`randpw 20`
db_user_gatewaydPW=`randpw 20`
db_user_ripple_restPW=`randpw 20`

sudo service postgresql start
sudo su - postgres -c "psql -c \"ALTER USER postgres WITH PASSWORD '$db_user_postgresPW';\""
sudo su - postgres -c "psql -c \"CREATE USER db_user_ripple_rest WITH PASSWORD '$db_user_ripple_restPW';\""
sudo su - postgres -c "psql -c \"CREATE DATABASE ripple_rest_db WITH OWNER db_user_ripple_rest encoding='utf8';\""
sudo su - postgres -c "psql -c \"CREATE USER db_user_gatewayd WITH PASSWORD '$db_user_gatewaydPW';\""
sudo su - postgres -c "psql -c \"CREATE DATABASE gatewayd_db WITH OWNER db_user_gatewayd encoding='utf8';\""

export DATABASE_URL=postgres://db_user_ripple_rest:$db_user_ripple_restPW@localhost:5432/ripple_rest_db

#Set users, passwords, DBs in configs
sed -i "s/postgres:password/db_user_gatewayd:$db_user_gatewaydPW/g" ./config/config.js
sed -i "s/\/ripple_gateway/\/gatewayd_db/g" ./config/config.js
#FIXME: NO SSL sed -i "s/@localhost:5432\/gatewayd_db/@localhost:5432\/gatewayd_db?native=true/g" ./config/config.js
cp lib/data/database.example.json lib/data/database.json
sed -i "s/DATABASE_URL/postgres:\/\/db_user_gatewayd:$db_user_gatewaydPW@localhost:5432\/gatewayd_db?native=true/g" ./lib/data/database.json
```

##INSTALL AND CONFIGURE RIPPLE-REST
```
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
```

##CREATE AND INSTALL STARTUP SCRIPTS

```
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
```
"start-all" can be added as a @reboot cron job or run manually.

##ADDITIONAL GATEWAYD CONFIGURATION (wallets, currencies)
```
#CONFIGURE gatewayd, add wallets, currencies
#When finished, move this up to gatewayd install section
cd ~/gatewayd
#add some example currencies
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

#Must run grunt migrate here
#Not sure why but running it at various other stages does not prevent the error #"/home/shell_user_gatewayd/gatewayd/lib/api/set_hot_wallet.js:22
#      address: address.address,
#                      ^
#TypeError: Cannot read property 'address' of null
#"
grunt migrate

bin/gateway set_hot_wallet rNXW9BmqufSRiZ5gUXMGmNFev3s8Lup4P3 ssowTc8ba2PG9ADTxuzsD4TBzs34M

#For obvious reasons, SET YOUR OWN ADDRESSES AND SECRETS! These are examples.
#NOTE: NEED TO GENERATE THESE DYNAMICALLY
```

##ISSUES/BROKEN:
  - ripple-rest api not accessible except via localhost
  - If the ripple-rest API is run over https, gatewayd cannot connect. You must remove the ssl section from ripple-rest/config.json. Sorry! Working on this.

##TO-DO:
 - use output from wallet generator to edit gatewayd's config.js
 - use sed to add currencies to gatewayd's config.js
 - properly configure ssl for gatewayd<->postgresql
 - figure out what the deal is with grunt migrate
