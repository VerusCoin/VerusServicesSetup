# ElectrumX for VerusCoin

**NOTE:** For all downloads mentioned below you are encouraged use the [Verus Signature verification tool](https://verus.io/verify-signatures) or a local standalone Verus daemon to make sure the files are authentic and have not been tampered with. Additionally, the setup described below is in no way production ready but is meant to illustrate the general process only. **System hardening, firewalling, signature verification and other measures are outside of the scope of this guide. You will have to take care of it for yourself!**

## Server

A VPS with 6GB of RAM, anything from 40GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.

## Operating System

This guide tailored to and tested on `Debian 10 "Buster"` but should probably also work on Debian-ish derivatives like `Devuan` or `Ubuntu` and others. This guide contains `systemd`-specific instuctions below, make sure to adapt them to your init system of choice. Before starting, please install the latest updates and prerequisites. 

```bash
apt update
apt upgrade
apt install wget libgomp1 git python3.7 python3-pip build-essential libleveldb-dev libboost-all-dev
pip3 install multidict chardet plyvel uvloop
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

Download the **latest** (`v0.7.2-6` used in this example) Verus binaries from the [GitHub Releases Page](https://github.com/VerusCoin/VerusCoin/releases). Unpack, move them into place and clean up like so: 

```bash
wget https://github.com/VerusCoin/VerusCoin/releases/download/v0.7.2-6/Verus-CLI-Linux-v0.7.2-6-amd64.tgz
tar xf Verus-CLI-Linux-v0.7.2-6-amd64.tgz; tar xf Verus-CLI-Linux-v0.7.2-6-amd64.tar.gz
mv verus-cli/{fetch-params,fetch-bootstrap,verusd,verus} ~/bin
rm -rf verus-cli Verus-CLI-Linux-v0.7.2-6-amd64.t*
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
## verus electrum node config
##

# electrum doesn't need a wallet
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

## verushashpy

Install the `verushashpy` module as shown below. 

```bash
cd /usr/src
git clone https://github.com/veruscoin/verushashpy
cd verushashpy
python3.7 setup.py install
```

## ElectrumX Installation

Create a new system user for `electrumx`.

```
useradd -rMs /bin/false electrumx
```

Now, check out the ElectrumX repo and install it:

```
cd /usr/src
git clone https://github.com/spesmilo/electrumx
cd electrumx; python3.7 setup.py install
```

## ElectrumX Configuration

Copy over the `systemd` unit file and create a datadir. Assign ownership of the datadir to the `electrumx` user.

```bash
cp /usr/src/electrumx/contrib/systemd/electrumx.service /etc/systemd/system
mkdir -p /electrumdb/VRSC
chown electrumx:electrumx /electrumdb/VRSC
```

Create a config file for electrumx called `/etc/electrumx.conf`. See [here](https://electrumx.readthedocs.io/en/latest/environment.html) for the full list of configuration options.

```bash
cat << EOF >/etc/electrumx.conf
COIN="Verus"
DB_DIRECTORY="/electrumdb/VRSC"
DAEMON_URL="http://verus:OBVIOUSLY-EDIT-HERE@127.0.0.1:27486/"

LOG_FORMAT="%(asctime)s %(levelname)s:%(name)s:%(message)s"
LOG_LEVEL="info"

SERVICES="tcp://0.0.0.0:17485,tcp://[::]:17485,rpc://127.0.0.1:17489,rpc://[::1]:17489"
MAX_SESSIONS="5000"

DB_ENGINE="leveldb"
EVENT_LOOP_POLICY="uvloop"

REQUEST_TIMEOUT="30"
SESSION_TIMEOUT="600"

BANDWIDTH_UNIT_COST="50000"
INITIAL_CONCURRENT="100"
COST_SOFT_LIMIT="0"
COST_HARD_LIMIT="0"

CACHE_MB="1500"
EOF
```

Make sure the VerusCoin wallet is running. You should now be able to start ElectrumX.

```
systemctl start electrumx.service
```

Display the logs with this command:

```
journalctl -fu electrumx.service
```

Enable autostart with this command:

```
systemctl enable electrumx.service
```

Initial sync will take up to 3 hours to complete. Before that is done, ElectrumX will only allow RPC connections via loopback, but no external connections. To check ElectrumX status, do

```
electrumx_rpc getinfo
```

To see info about connected clients, execute

```
electrumx_rpc sessions
```

### Enable `logrotate` for `verusd`

As `root` user, create a file called `/etc/logrotate.d/verus` with these contents:

```
/home/verus/.komodo/VRSC/debug.log
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

### Autostart `verusd` using `cron`


Switch to the `verus` user. Edit the `crontab` using `crontab -e` and add this to the appropriate place:

```crontab
PATH=".:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:/home/verus/bin"
@reboot cd /home/verus/.komodo/VRSC && /home/verus/bin/verusd -daemon 1>/dev/null 2>&1
```
