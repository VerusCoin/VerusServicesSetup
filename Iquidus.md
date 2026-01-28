# Iquidus Explorer for Verus

**NOTE:** For all downloads mentioned below you are encouraged use the [Verus Signature verification tool](https://verus.io/verify-signatures) or a local standalone Verus daemon to make sure the files are authentic and have not been tampered with. Additionally, the setup described below is in no way production ready but is meant to illustrate the general process only. **System hardening, firewalling, signature verification and other measures are outside of the scope of this guide. You will have to take care of it for yourself!**

## Server

A VPS with 4GB of RAM, anything from 30GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.

## Operating System

This guide tailored to and tested on `Debian 10 "Buster"` but should probably also work on Debian-ish derivatives like `Devuan` or `Ubuntu` and others. Before starting, please install the latest updates and prerequisites. 

```bash
apt update
apt upgrade
apt install wget libgomp1 git python build-essential
```

Additionally, Iquidus requires a MongoDB backend. Please refer to [this](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/) document for MongoDB install instructions.

With the minimum memory requirement above, `dphys-swapfile` will be necessary. It will create a 2GB swap file per default, which is sufficient. In situations where more memory is available, installation of `dphys-swapfile` can be skipped altogether.

```bash
apt install dphys-swapfile
```

## Verus Node

Create a user account for the Verus node and switch to it. 

```bash
useradd -m -d /home/verus -s /bin/bash verus
su - verus
```

Prepare the `~/bin` directory and add it to the users' `PATH`.

```bash
mkdir ~/bin
echo export PATH=\"${PATH}:/home/verus/bin\" >> ~/.bashrc
```

Log out and back into the account to get the new `PATH` into the environment.

```bash
exit
su - verus
```

Download the **latest** (`v0.7.2-8` used in this example) Verus binaries from the [GitHub Releases Page](https://github.com/VerusCoin/VerusCoin/releases). Unpack, move them into place and clean up like so: 

```bash
wget https://github.com/VerusCoin/VerusCoin/releases/download/v0.7.2-6/Verus-CLI-Linux-v0.7.2-8-amd64.tgz
tar xf Verus-CLI-Linux-v0.7.2-8-amd64.tgz; tar xf Verus-CLI-Linux-v0.7.2-8-amd64.tar.gz
mv verus-cli/{fetch-params,fetch-bootstrap,verusd,verus} ~/bin
rm -rf verus-cli Verus-CLI-Linux-v0.7.2-8-amd64.t*
```

Use the supplied script to download a copy of the `zcparams` data. Watch for and fix any occuring errors until you can be sure you successfully have gotten a complete `zcparams` copy.

```bash
fetch-params
# ... a lot of output from wget and sadly no clear conclusion notice
```

Use the supplied script to download and unpack the latest bootstrap into the default data directory. Watch for and fix any occuring errors until you can be sure you successfully got, checksum-verified and unpacked the latest bootstrap into the default Verus data directory location.

```bash
fetch-bootstrap
# ... some output
Enter blockchain data directory or leave blank for default:<return>
Install bootstrap in /home/verus/.komodo/VRSC? ([1]Yes/[2]No)<1><return>
# ... some more output, then, ideally
Bootstrap successfully installed
```

Create (and where necessary, adapt) a `VRSC.conf` file.

```bash
cat << EOF > ~/.komodo/VRSC/VRSC.conf
##
## verus iquidus node config
##

# explorer doesn't need a wallet
disablewallet=1

# network options
listen=1
port=27485
maxconnections=1024

# rpc options
server=1
rpcuser=verus
rpcpassword=OBVIOUSLY-EDIT-HERE
rpcport=27486
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcthreads=64
rpcworkqueue=256

# logging options
logtimestamps=1
logips=1

# debug options
shrinkdebugfile=0
debug=0

# checks
checklevel=4
checkblocks=1440

# addnodes
addnode=167.235.72.129:27485
addnode=136.243.31.102:27485
addnode=77.42.48.58:27485
addnode=136.243.227.138:27485
addnode=135.181.68.2:27485
addnode=136.243.31.101:27485
addnode=157.90.127.142:27485
addnode=136.243.31.97:27485
addnode=77.42.48.57:27485
addnode=95.216.104.214:27485
addnode=136.243.31.98:27485
addnode=135.181.186.8:27485
addnode=157.90.113.198:27485
addnode=157.90.155.113:27485
addnode=136.243.31.96:27485
addnode=95.217.1.76:27485
addnode=77.42.48.56:27485
addnode=45.79.237.198:27485
addnode=172.104.48.148:27485
addnode=66.228.59.168:27485
addnode=176.9.103.23:27485
addnode=162.55.8.164:27485
addnode=136.243.227.137:27485
addnode=152.228.135.1:27485
addnode=51.75.25.174:27485
addnode=162.55.59.82:27485

# EOF
EOF
```

A reasonably secure `rpcpassword` for the above config can be generated with the commands below.

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

Start `verusd` and follow the `debug.log` output to make sure `verusd` syncs to current height and otherwise comes up successfully.

```bash
cd ~/.komodo/VRSC && verusd -daemon 1>/dev/null 2>&1
tail -f debug.log
```

Now exit the `verus` account.

```bash
exit
```

## NodeJS

Create a user account to run `iquidus` from and switch to it. Within this user account `nvm.sh` and ultimately `NodeJS` will be installed.

```bash
useradd -m -d /home/iquidus -s /bin/bash iquidus
su - iquidus
```

Prepare the `~/bin` directory and add it to the users' `PATH`.

```bash
mkdir ~/bin
echo export PATH=\"${PATH}:/home/iquidus/bin\" >> ~/.bashrc
```

Install NodeJS v9 using `nvm.sh` like this: 

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.37.2/install.sh | bash
```

To activate the changes, log out of and back into the `iquidus` account.

```bash
exit
su - iquidus
```

Install and activate NodeJS v12.

```
nvm install 12
nvm use 12
```

Within the `iquidus` account scope, globally install `pm2` as shown below.

```bash
npm -g install pm2
```

## Iquidus Installation

While still logged in as the `iquidus` user, clone the Iquidus repository.

```bash
cd ~/
git clone https://github.com/Oink70/explorer
```

See https://github.com/Oink70/explorer for the basic of install instructions of Iquidus. Read this document thoroughly, it contains information about the MongoDB backend, necessary steps for initial data import (also see right below) and keeping the explorer in sync with the Verus chain. 

**NOTE:** Before starting an init sync, in the resulting installed `node_modules`, we need to disable `json-bigint strict mode` in 3 places: 

```bash
node_modules/bitcoin-node-api/node_modules/bitcoin-core/coverage/src/parser.js.html:251
node_modules/bitcoin-node-api/node_modules/bitcoin-core/dist/src/parser.js:27
node_modules/bitcoin-core/dist/src/parser.js:27
```

Each place has a representation of `strict: true` which needs to be changed to `strict: false` within the respective syntax limits.

To setup Iquidus the way you like it, copy `settings.json.template` to `settings.json` and adapt where necessary. Launch the explorer using `pm2` and follow the log output to make sure Iquidus starts up allright. Ideally, Iquidus is now listening on the IP and Port you have configured.

```bash
cd ~/explorer
pm2 start --name "explorer" "npm start"; pm2 log all
```

### Enable `logrotate`

As `root` user, create a file called `/etc/logrotate.d/verus-iquidus` with these contents:

```
/home/verus/.komodo/VRSC/debug.log
/home/iquidus/.pm2/logs/explorer.VRSC-out.log
/home/iquidus/.pm2/logs/explorer.VRSC-error.log
{
  rotate 14
  daily
  compress
  delaycompress
  copytruncate
  missingok
  notifempty
}
```

### Autostart using `cron`

Switch to the `verus` user. Edit the `crontab` using `crontab -e` and add this to the appropriate place:

```crontab
PATH=".:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:/home/verus/bin"
@reboot cd /home/verus/.komodo/VRSC && /home/verus/bin/verusd -daemon 1>/dev/null 2>&1
```

Switch to the `iquidus` user. Edit the `crontab` using `crontab -e` and add this to the appropriate place:

```crontab
PATH=".:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:/home/iquidus/bin:/home/iquidus/.nvm/versions/node/v12.20.0/bin"
@reboot cd /home/iquidus/explorer && pm2 start --name explorer "npm start" 1>/dev/null 2>&1
```

**NOTE:** with every NodeJS update, the last part of the `PATH` variable from the `iquidus` crontab may change since it has a version number in it.
