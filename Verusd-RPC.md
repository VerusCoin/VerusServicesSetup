# Verusd-RPC

Operating a Lite Wallet server, like Verusd-RPC, requires you to know about systems administration, IT security, databases, software development, coin daemons and other more or less related stuff. Maintaining it can literally be more work than a full-time job

A VPS with 8GB of RAM, anything above 40GB **SSD** storage and 2 CPU cores which knows about AES-NI is the absolute minimum requirement. Generally, having more RAM is more important than having more CPU power here. Additionally, the hypervisor of your VPS _must_ pass through the original CPU designation from its host. See below for an example that will likely lead to trouble.

```bash
lscpu|grep -i "model name"
Model name:            QEMU Virtual CPU version 2.5+
```

Basically, anything in there that is not a real CPU name _may_ cause NodeJS to behave funny despite the `Virtual CPU` having all necessary CPU flags. Be aware and ready to switch servers and/or hosting companies if need be. Start following the guide while logged in as `root`.

## Operating System

This guide is tailored to and tested on `Debian 11 "Bullseye"` but should probably also work on Debian-ish derivatives like `Devuan` or `Ubuntu` and others. Before starting, please install the latest updates and prerequisites.

```bash
apt update
apt -y upgrade
apt -y install libgomp1 git libboost-all-dev libsodium-dev build-essential
```

## Verus Daemon

Create a user for the Verus daemon, switch to that account:

```bash
useradd -m -d /home/verus -s /bin/bash verus
su - verus
```

Download the **latest** (`v0.9.6-1` used in this example) Verus binaries from the [GitHub Releases Page](https://github.com/VerusCoin/VerusCoin/releases). Unpack, move them into place and clean up like so:

```bash
mkdir ~/bin
cd ~/bin
wget https://github.com/VerusCoin/VerusCoin/releases/download/v0.9.6-1/Verus-CLI-Linux-v0.9.6-1-x86_64.tgz
tar xf Verus-CLI-Linux-v0.9.6-1-x86_64.tgz; tar xf Verus-CLI-Linux-v0.9.6-1-x86_64.tar.gz
mv verus-cli/{fetch-params,fetch-bootstrap,verusd,verus} ~/bin
rm -rf verus-cli Verus-CLI-Linux-v0.9.6-1-x86_64.t*
```

Use the supplied script to download a copy of the `zcparams` data. Watch for and fix any occuring errors until you can be sure you successfully have gotten a complete `zcparams` copy.

```bash
fetch-params
# ... a lot of output from wget and sadly no clear conclusion notice
```

Since this node will be running with all indexes enabled, it will need to be synchronized from genesis. Bootstrapping is not an option!

Now, let's create the data and wallet export directory. Then, get the bootstrap and unpack it there.

```bash
mkdir ~/.komodo
mkdir ~/.komodo/VRSC
mkdir ~/export
```

It's time to do the wallet config. A reasonably secure `rpcpassword` can be generated using this command:   

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

Create `~/.komodo/VRSC/VRSC.conf` and include the parameters listed below, adapt the ones that need adaption.
```conf
##

# network options
listen=1
listenonion=0
port=27485
maxconnections=1024

# rpc options
server=1
rpcport=27486
rpcuser=verus
rpcpassword=rpcpassword
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcthreads=256
rpcworkqueue=1024

# indexing options:
insightexplorer=1
idindex=1
txindex=1
addressindex=1
timestampindex=1
spentindex=1

# logging options
logtimestamps=1
logips=1

# miscellaneous options
banscore=64

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
```

Afterwards, start the verus daemon and let it sync the blockchain. as noted before, the indexes require a full synchronization from genesis, so this make take many hours. We'll also watch the debug log for a moment:

```bash
cd ~/.komodo/VRSC; verusd -daemon 1>/dev/null 2>&1; sleep 1; tail -f debug.log
```

Press `ctrl-c` to exit `tail` if it looks alright. To check the status and know when the initial sync has been completed, issue

```bash
verus getinfo
```
When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are not on a fork. Edit the `crontab` using `crontab -e` and include the line below to autostart the wallet:

```crontab
@reboot cd /home/verus/.komodo/VRSC; /home/verus/bin/verusd -daemon 1>/dev/null 2>&1
```

**HINT:** if you can't stand `vi`, do `EDITOR=nano crontab -e` ;-)


## Node.js

Create a new user account to run the pool from. Switch to that user to setup `nvm.sh`:

Instal `nvm`. As root user, do:
```bash
useradd -m -d /home/verusd-rpc -s /bin/bash verusd-rpc
su - verusd-rpc
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
exit
```
Optional, but recommended:
	Test that `nvm` is installed:
	```bash
	su - verusd-rpc
	nvm -v
  exit
	```
	If this does not give you the version you installed (`0.39.3` in this case), `nvm` was **not** (properly) installed.
	If it did return the proper answer, you can continue.

install `node` and `pm2`:
```bash
su - verusd-rpc
nvm install 18
npm install -g pm2
```
Because `nvm.sh` comes without it, we need to add one symlink into its bindir for our installed NodeJS.

```bash
which node
/home/verusd-rpc/.nvm/versions/node/v18.14.0/bin/node
```

Change to the resulting directory and create a symlink like below.

```bash
cd /home/verusd-rpc/.nvm/versions/node/v18.14.0/bin/
ln -s node nodejs
```

## verusd-rpc

(still in the `verusd-rpc` account, home directory) clone the *verusd-rpc* code:
```bash
git clone https://github.com/VerusCoin/bitcoind-rpc.git verusd-rpc
```

```bash
cd ~/verusd-rpc
npm update
npm install
```
Set up the environment file with the proper parameters in the `~/verusd-rpc` directory:
```
touch .env
nano .env
```
put the following data in and update with the specifics of your verusd daemon:
```
RPCPORT=<rpcport from your coin.conf file in the chain data folder>
RPCPASSWORD=<rpcpassword from your coin.conf file in the chain data folder>
RPCUSER=<rpcuser from your coin.conf file in the chain data folder>
RPCIP=127.0.0.1
NODEPORT=8000
```
Exit and save on the way out.

	note: by default it will use the local node (RPCIP=127.0.0.1), but if desired you can configure (both *VERUSD-RPC* and your verusd node) to communicate over the internet.
	Unless you have strict security protocols in place (firewall hole to only allow a specific IP, tunnelling between the two systems) this is not advisable from a security perspective.

	note2: `NODEPORT` can be altered to whatever available port you wish to use. Make sure the firewall is open for that port.


Optional, but recommended:
	testrun `Verusd-RPC` with node:
	```bash
	node httpserver.js
	```
	Now locally test the functionality in a 2nd terminal on the same system (doesn't matter what user account):
	```bash
	curl --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "getinfo", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8000/
	```
	You should get a json reply with the `getinfo` you would also get if you run `verus -chain=<whatever-you-connected-to> getinfo`

Running `Verusd-RPC` for production:
```bash
cd /home/verusd-rpc/verusd-rpc
pm2 start httpserver.js --name Verusd-RPC
```

### Verusd-RPC Autostart

Edit your crontab using `crontab -e` and shove in this line at the bottom:

```crontab
@reboot /bin/sleep 300 && cd /home/verusd-rpc/verusd-rpc && /home/verusd-rpc/.nvm/versions/node/v18.14.0/bin/pm2 start httpserver.js --name verusd-rpc
```

**HINT:** if you can't stand `vi`, do `EDITOR=nano crontab -e` ;-)

## Further considerations

None of the topics below is strictly necessary, but most of them are recommended.

### Enabling firewall

As `root` user:
```bash
ufw allow from any to any port 22 comment "SSH access"
```
If you want to limit the IPs that can access your server over SSH (eg, if you have a fixed IP address or use a SSH-jump server) replace the first `any` with the IP or ip-range. doublecheck this or you will lock yourself out!
```bash
ufw allow from any to any port 80,443 comment "Standard web ports"
```
add multiple mining ports by separating them by commas, analogue to the web ports.
```bash
ufw enable
```
If you are not going to use `NGINX` to proxy:
```bash
ufw allow from any to any port xxxx comment "verusd-rpc (<chain-name>) API port"
```


### Useful DNS resolvers

Empty your `/etc/resolv.conf` and replace it with this:

```conf
# google, https://developers.google.com/speed/public-dns/docs/using
nameserver 8.8.4.4
nameserver 8.8.8.8
nameserver 2001:4860:4860::8844
nameserver 2001:4860:4860::8888

# verisign, https://publicdnsforum.verisign.com/discussion/13/verisign-public-dns-set-up-configuration-instructions
nameserver 64.6.64.6
nameserver 64.6.65.6
nameserver 2620:74:1b::1:1
nameserver 2620:74:1c::2:2

# quad9, https://www.quad9.net/faq/
nameserver 9.9.9.9
nameserver 149.112.112.112
nameserver 2620:fe::fe
nameserver 2620:fe::9

# cloudflare/apnic, https://1.1.1.1/de/
nameserver 1.1.1.1
nameserver 1.0.0.1
nameserver 2606:4700:4700::1111
nameserver 2606:4700:4700::1001

# opendns, https://use.opendns.com, https://www.opendns.com/about/innovations/ipv6/
nameserver 208.67.222.222
nameserver 208.67.220.220
nameserver 2620:119:35::35
nameserver 2620:119:53::53

# see 'man 5 resolv.conf'
options rotate timeout:1 attempts:5
```

### Improving SSH security

If you remember the good old `rand=4; // chosen by fair dice roll` comic, you're probably doing this anyways. If you don't, go Google the comic, you might have missed a laugh there!

As `root`, generate a proper `/etc/ssh/moduli` like this:

```bash
ssh-keygen -G "/root/moduli.candidates" -b 4096
mv /etc/ssh/moduli /etc/ssh/moduli.old
ssh-keygen -T /etc/ssh/moduli -f "/root/module.candidates"
rm "/root/moduli.candidates"
```

Add the recommended changes from [CiperLi.st](https://cipherli.st) to `/etc/ssh/sshd_config`, also make sure that `PermitRootLogin` is at least set to `without-password`. Then remove and re-generate your host keys like this:

```bash
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

To finish, restart the ssh server:

```bash
/etc/init.d/sshd restart
```

### Reverse-proxying `verusd-rpc` behind `nginx`

As `root`, install `nginx` and enable it on boot using these commands:

```bash
apt -y install nginx
update-rc.d enable nginx
```

Create `/etc/nginx/blockuseragents.rules` with these contents:

```conf
map $http_user_agent $blockedagent {
default         0;
~*malicious     1;
~*bot           1;
~*backdoor      1;
~*crawler       1;
~*bandit        1;
}
```

Edit `/etc/nginx/sites-available/default` to look like this:

```conf
include /etc/nginx/blockuseragents.rules;
server {
	if ($blockedagent) {
		return 403;
	}
	if ($request_method !~ ^(GET|HEAD)$) {
		return 444;
	}

	listen 80 default_server;
	listen [::]:80 default_server;
	charset utf-8;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;

	location / {
		proxy_pass http://127.0.0.1:8000/;
		proxy_set_header X-Real-IP $remote_addr;
	}

	location /admin {
		rewrite ^/.* /stats permanent;
	}

}
```

Restart `nginx`:

```bash
/etc/init.d/nginx restart
```

If you've followed the above steps correctly, your `verusd-rpc` is now proxied behind nginx.

### Enable `logrotate`

As `root` user, create a file called `/etc/logrotate.d/verusd-rpc` with these contents:

```conf
/home/verus/.komodo/VRSC/debug.log
/home/verusd-rpc/.pm2/logs/verusd-rpc-out.log
/home/verusd-rpc/.pm2/logs/verusd-rpc-error.log
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

### Increase open files limit

Add this to your `/etc/security/limits.conf`:

```conf
* soft nofile 1048576
* hard nofile 1048576
```

Reboot to activate the changes. Alternatively you can make sure all running processes are restarted from within a shell that has been launched _after_ the above changes were put in place, which usually is a huge pain. Just reboot.


### Networking optimizations

If your pool is expected to receive a lot of load, consider implementing below changes, all as `root`:

Enable the `tcp_bbr` kernel module:

```bash
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules
```

Edit your `/etc/sysctl.conf` to include below settings:

```conf
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.ip_local_port_range = 16001 65530
net.core.somaxconn = 20480
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_limit_output_bytes = 131072
```

Run below command to activate the changes, alternatively reboot the machine:


```bash
sysctl -p /etc/sysctl.conf
```

### Change swapping behaviour

If your system has a lot of RAM, you can change the swapping behaviour to only swap when necessary. Edit `/etc/sysctl.conf` to include this setting:

```conf
vm.swappiness=1
```

The range is `1-100`. The *lower* the number, the *later* the system will start swapping stuff out. Run below command to activate the change, alternatively reboot the machine:

```bash
sysctl -p /etc/sysctl.conf
```

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this:

```bash
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.
