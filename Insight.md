# Insight Explorer for Verus

**NOTE:** For all downloads mentioned below you are encouraged use the [Verus Signature verification tool](https://verus.io/verify-signatures) or a local standalone Verus daemon to make sure the files are authentic and have not been tampered with. Additionally, the setup described below is in no way production ready but is meant to illustrate the general process only. **System hardening, firewalling, signature verification and other measures are outside of the scope of this guide. You will have to take care of it for yourself!**

## Server

A VPS with 8GB of RAM, anything from 60GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.

## Operating System

This guide tailored to and tested on `Debian 12 "bookworm"` but should probably also work on Debian-ish derivatives like `Devuan` or `Ubuntu` and others. Before starting, please install the latest updates and prerequisites.

```bash
apt update
apt upgrade
apt install wget libgomp1 git python build-essential libzmq3-dev libzmq5
```
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

Download the **latest** (`v1.2.12-1` used in this example) Verus binaries from the [GitHub Releases Page](https://github.com/VerusCoin/VerusCoin/releases). Unpack, move them into place and clean up like so:

```bash
wget https://github.com/VerusCoin/VerusCoin/releases/download/v1.2.12-1/Verus-CLI-Linux-v1.2.12-1-amd64.tgz
tar xf Verus-CLI-Linux-v1.2.12-1-amd64.tgz; tar xf Verus-CLI-Linux-v1.2.12-1-amd64.tar.gz
mv verus-cli/{fetch-params,fetch-bootstrap,verusd,verus} ~/bin
rm -rf verus-cli Verus-CLI-Linux-v1.2.12-1-amd64.t*
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

Create (and where necessary, adapt) a `VRSC.conf` file that has the necessary additional settings for Insight (namely `zmqpubrawtx` and `zmqpubhashblock`).

```bash
cat << EOF > ~/.komodo/VRSC/VRSC.conf
##
## verus insight node config
##

# insight-related options
zmqpubrawtx=tcp://127.0.0.1:27487
zmqpubhashblock=tcp://127.0.0.1:27487

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

## NodeJS, Insight

Create a user account to run `insight` from and switch to it.

```bash
useradd -m -d /home/insight -s /bin/bash insight
su - insight
```

Prepare the `~/bin` directory and add it to the users' `PATH`.

```bash
mkdir ~/bin
echo export PATH=\"${PATH}:/home/insight/bin\" >> ~/.bashrc
```

Install NodeJS v20 using `nvm.sh` like this:

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

To activate the changes, log out of and back into the `insight` account.

```bash
exit
su - insight
```

Install and activate NodeJS v20.

```
nvm install 20
nvm use 20
```

Within the `insight` account scope, globally install `pm2` and `bitcore-node` as shown below.

```bash
npm -g install pm2 git+https://github.com/VerusCoin/bitcore-node-komodo.git
```

Since we opted to use the newest NodeJS version the v0.4.3 insight API supports, we'll need to go a slightly different route than shown in other guides. Prepare a directory structure for Insight.

```bash
cd ~
mkdir -p ~/insight.VRSC/node_modules
```

Create a `package.json` file, this is used to integrate Insight with `pm2`.

```bash
cat << EOF > ~/insight.VRSC/package.json
{
  "scripts": {
    "start": "bitcore-node start"
  },
  "description": "Verus Bitcore Node",
  "repository": "https://github.com/VerusCoin/bitcore-node-komodo",
  "license": "MIT",
  "readme": "README.md",
  "dependencies": {
    "bitcore-lib-komodo":  "git+https://github.com/VerusCoin/bitcore-lib-komodo.git",
    "bitcore-node-komodo": "git+https://github.com/VerusCoin/bitcore-node-komodo.git",
    "insight-api-komodo":  "git+https://github.com/VerusCoin/insight-api-komodo.git",
    "insight-ui-komodo":   "git+https://github.com/VerusCoin/insight-ui-komodo.git"
  }
}
EOF
```

Create a `bitcore-node.json` file, this is the Insight configuration file.

```bash
cat << EOF > ~/insight.VRSC/bitcore-node.json
{
  "network": "mainnet",
  "ip": "127.0.0.1",
  "port": 3002,
  "services": [
    "bitcoind",
    "api",
    "insight-ui-komodo",
    "web"
  ],
  "servicesConfig": {
    "bitcoind": {
      "connect": [
        {
          "rpchost": "127.0.0.1",
          "rpcport": 27486,
          "rpcuser": "verus",
          "rpcpassword": "OBVIOUSLY-EDIT-HERE",
          "zmqpubrawtx": "tcp://127.0.0.1:27487"
        }
      ]
    },
    "api":{
      "disableRateLimiter": false,
      "rateLimiterOptions": {
        "whitelist": ["::ffff:127.0.0.1","127.0.0.1"],
        "whitelistLimit": 500000,
        "whitelistInterval": 3600000,
        "blacklist": [],
        "blacklistLimit": 0,
        "blacklistInterval": 3600000,
        "limit": 10,
        "interval": 60000
      }
    }
  }
}
EOF
```

Enter the `node_modules` directory, clone the needed repositories and install the required submodules for the `insight-api-komodo` repository.

```bash
cd ~/insight.VRSC/node_modules
git clone https://github.com/VerusCoin/insight-ui-komodo
git clone https://github.com/VerusCoin/insight-api-komodo ./api
cd api
npm install --production
```

Create a `blocked.json` file in the root of the `vrsc.insight` folder

```bash
cat << EOF > ~/insight.VRSC/blocked.json
{
  "addresses": [
    "RTqQe58LSj2yr5CrwYFwcsAQ1edQwmrkUU"
  ],
  "reason": "These addresses cause daemon performance issues and are blocked from queries"
}
```

Now launch Insight using `pm2` and follow the log output to make sure Insight launches allright.

```bash
cd ~/insight.VRSC
pm2 start --name insight.VRSC "npm start"; pm2 log insight.VRSC
```

A successful launch looks like this:

```bash
0|insight.VRSC  | [2020-12-29T14:59:28.178Z] info: Using config: /home/insight/insight.VRSC/bitcore-node.json
0|insight.VRSC  | [2020-12-29T14:59:28.180Z] info: Using network: livenet
0|insight.VRSC  | [2020-12-29T14:59:28.181Z] info: Starting bitcoind
0|insight.VRSC  | [2020-12-29T14:59:28.247Z] info: Komodo Daemon Ready
0|insight.VRSC  | [2020-12-29T14:59:28.248Z] info: Starting web
0|insight.VRSC  | [2020-12-29T14:59:28.255Z] info: Starting insight-api-komodo
0|insight.VRSC  | [2020-12-29T14:59:28.256Z] info: Starting insight-ui-komodo
0|insight.VRSC  | [2020-12-29T14:59:28.256Z] info: Bitcore Node ready
0|insight.VRSC  | [2020-12-29T14:59:28.738Z] warn: ZMQ connection delay: tcp://127.0.0.1:27487
0|insight.VRSC  | [2020-12-29T14:59:28.738Z] info: ZMQ connected to: tcp://127.0.0.1:27487
0|insight.VRSC  | [2020-12-29T15:00:08.416Z] info: Komodo Height: 1329705 Percentage: 100.00
0|insight.VRSC  | [2020-12-29T15:01:31.224Z] info: Komodo Height: 1329706 Percentage: 100.00
```

Insight is now listening at http://127.0.0.1:3002. As mentioned in the beginning of this document, this is not a production ready setup but a proof of concept guide. In order to be able to reach the finished installation from the outside, you need to setup a webserver to proxy back and forth between the internet and the Insight deployment. For proper operation, the webserver does need to support `websocket` proxying.

### Enable `logrotate`

As `root` user, create a file called `/etc/logrotate.d/verus-insight` with these contents:

```
/home/verus/.komodo/VRSC/debug.log
/home/insight/.pm2/logs/insight.VRSC-out.log
/home/insight/.pm2/logs/insight.VRSC-error.log
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

Switch to the `insight` user. Edit the `crontab` using `crontab -e` and add this to the appropriate place:

```crontab
PATH=".:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:/home/insight/bin:/home/insight/.nvm/versions/node/v9.11.2/bin"
@reboot cd /home/insight/insight.VRSC && pm2 start --name insight.VRSC "npm start" 1>/dev/null 2>&1
```
**NOTE:** with every NodeJS update, the last part of the `PATH` variable from the `insight` crontab may change since it has a version number in it.
