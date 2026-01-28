# Verus Node with staking wallet

## Server

A VPS with 4GB of RAM, anything above 50GB SSD storage and 2 CPU cores that are able to handle AES-NI is the absolute minimum requirement. Start following the guide while logged in as `root`.


## Operating System

This guide tailored to and tested on `Debian 9 "Stretch"`. Before starting, please install the latest updates:

```
apt update
apt -y upgrade
```

## wallet

The packages required in order to compile a VerusCoin wallet can be installed like this:

```
apt -y install build-essential git pkg-config libc6-dev m4 g++-multilib autoconf \
                   libtool ncurses-dev unzip git python python-zmq zlib1g-dev wget \
                   libcurl4-openssl-dev bsdmainutils automake curl
```


Create a useraccount for the wallet. Switch to that account.

```
useradd -m -d /home/verus -s /bin/bash verus
su - verus
```

Now, clone the source tree and build the binaries:

```
git clone https://github.com/VerusCoin/VerusCoin
cd VerusCoin
./zcutil/fetch-params.sh
./zcutil/build.sh -j$(nproc)
```

After that is done, create a `~/bin` directory and copy over the binaries. Strip the debug symbols.

```
mkdir ~/bin
cp src/verusd src/verus src/verus-tx ~/bin
strip ~/bin/verus*
```

Now, lets create the data directory. Then, get the bootstrap and unpack it there.

```
mkdir -p ~/.komodo/VRSC
cd ~/.komodo/VRSC
wget https://bootstrap.veruscoin.io/VRSC-bootstrap.tar.gz
tar xf VRSC-bootstrap.tar.gz
rm VRSC-bootstrap.tar.gz
```

Create `~/.komodo/VRSC/VRSC.conf` and include the parameters listed below, adapt the ones that need adaption.
A resonably secure `rpcpassword` can be generated using this command:
`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1`.

```
server=1
listen=1
listenonion=0
maxconnections=256

# logging related options
logtimestamps=1
logips=1
shrinkdebugfile=0

# how many blocks to check on startup
checkblocks=64

# indexing options
txindex=1
addressindex=1
timestampindex=1
spentindex=1

# make sure ipv4 & ipv6 is used
bind=0.0.0.0
bind=::

# rpc settings
rpcuser=veruscoin
rpcpassword=your-secret-veruscoin-rpc-password
rpcport=27486
rpcthreads=256
rpcworkqueue=1024
rpcbind=127.0.0.1
rpcallowip=127.0.0.1

# if a peer jacks up more than 25 times in a row, ban it
banscore=25

# stake if possible, although it's probably not helping much
mint=1

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

```

Afterwards, start the daemon again and let it sync the blockchain:

```
verusd -daemon
```

To check the status and know when the initial sync has been completed, issue

```
verus getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. While we wait for this to happen, lets continue.

## Configuration Instructions

Shielding is no longer required for mined / staked VerusCoins. We will need a public but don't need a z-address, but make one anyway for this. Switch to the `verus` user and generate the addresses:

```
verus getnewaddress
verus z_getnewaddress
```

Next, we will dump the private keys of these addresses for safety reasons.
For the public address, use

```
verus dumpprivkey <public VerusCoin address>
```

For the z-address, use

```
verus z_exportkey <VerusCoin z-address>
```

## Further considerations

None of the topics below is strictly necessary, but most of them are recommended.

### Improving SSH security

If you remember the good old `rand=4; // chosen by fair dice roll` comic, you're probably doing this anyways. If you don't go google the comic, you might have missed a laugh there!

As `root`, generate a proper `/etc/ssh/moduli` like this:

```
ssh-keygen -G "/root/moduli.candidates" -b 4096
mv /etc/ssh/moduli /etc/ssh/moduli.old
ssh-keygen -T /etc/ssh/moduli -f "/root/module.candidates"
rm "/root/moduli.candidates"
```

Add the recommended changes from [CiperLi.st](https://cipherli.st) to `/etc/ssh/sshd_config`, also make sure that `PermitRootLogin` is at least set to `without-password`. Then remove and re-generate your host keys like this:

```
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

To finish, restart the ssh server:

```
/etc/init.d/sshd restart
```

### Autostart using `cron`

Switch to the `verus` user. Edit the `crontab` using `crontab -e` and include the lines below:

```
@reboot /home/verus/bin/verusd -daemon 1>/dev/null 2>&1
```

### Increase open files limit

Add this to your `/etc/security/limits.conf`:

```
* soft nofile 1048576
* hard nofile 1048576
```

Reboot to activate the changes. Alternatively you can make sure all running processes are restarted from within a shell that has been launched _after_ the above changes were put in place.


### Change swapping behaviour

If your system has a lot of RAM, you can change the swapping behavior to only swap when necessary. Edit `/etc/sysctl.conf` to include this setting:

```
vm.swappiness=1
```

The range is `1-100`. The *lower* the number, the *later* the system will start swapping stuff out. Run below command to activate the change, alternatively reboot the machine:

```
sysctl -p /etc/sysctl.conf
```

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this:

```
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.

### Enable `ufw` firewall
First setup the required ports for access.
If you configured your SSH access to another port (not a bad idea), change the port accordingly.
If you don't need SSH access, don't run the `SSH port` line.
If you do not want your node to accept incoming connections, do not run the `Verus P2P port` line.
```bash
ufw allow from any to any port 22 comment "SSH port"
ufw allow from any to any port 27485 proto tcp comment "Verus P2P port"
ufw enable
```
If you are connected through a SSH connection, do not disconnect until you have verified with a new connection that you can still make a connection to your node.
