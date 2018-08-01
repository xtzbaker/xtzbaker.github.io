---
layout: post
title:  "Secure Tezos Public Node"
date:   2018-08-01 07:24:29 +0000
author: xtzbaker
---

In this post we will setup a Public relay node.  Once complete you will have a secure server running `tezos-node` as a service. This node will contain nothing particularly sensitive (it will in essence be an empty wallet) and as a consequence can be treated as if it were ephemeral.  This means we don't care about any state on the node and if a problem develops (e.g. the node goes down or develops connectivity issues) we can simply destroy it and create it again from scratch.  In this first iteration, when a rebuild is required we will need to download the whole blockchain again. It will be important to be able to do this in a timely fashion for applying security updates, so we will explore ways to speed this up in a future post.

One of the most interesting things we will do with this node is completely lock down the network connectivity so there is nothing but Tezos related traffic going through it.  In particular no outbound HTTP/HTTPS traffic is allowed. We will keep SSH running on the node but lock it down with a security group on our cloud provider.  In a future iteration we will actually remove SSH to be super paranoid!

To apply security patches we will rebuild this node once a week, with the strict firewall rules in place there is no other way than a rebuild.

First thing we need to do is provision the node on our provider, you may use your cloud providers api or simply do it via the web UI.  We are provisioning a BareMetal Ubuntu Xenial/16.04 (x86 64-bit) server, for the remainder of the code we will assume the ip assigned to this node is `203.0.113.10`.  

Once the server is up and running we recommend you copy your `tezos-node` binary to the server, here we use `scp` but any file transfer mechanism will suffice.  Whilst we provide compiled binaries we strongly recommend you compile your own binaries.

```
curl -L https://github.com/xtzbaker/tezos/releases/download/002-PsYLVpVv/tezos-node-x86_64-linux -o tezos-node
scp tezos-node root@203.0.113.10:/usr/local/bin/tezos-node
```

We are now ready to login to our server and configure it.

```
ssh root@203.0.113.0
```

## Initial setup

To begin we will use `apt` to install the necessary dependencies of Tezos and remove some unnecessary packages:

```
apt-get update
apt-get -y upgrade

# Install dependencies
DEBIAN_FRONTEND=noninteractive apt-get install -y libhidapi-dev libev4 iptables-persistent

# Remove unnecessary packages

apt-get autoremove -y curl wget python python3 make gcc ssh

```

You should review the packages still installed using `apt list --installed` as packages installed by default may vary by your cloud provider

## Network Block Device

Your provider may use a Network Block Device for your HDD/storage.  If so you need to determine what port it is communicating on for your firewall rules.  The below command checks what port `xnbd-client` is connecting out on and puts it into the MY_NBD_PORT variable for later use.  It is likely you may need to cutomise this for your cloud provider if they do not use xnbd-client.

If your storage is not done through nbd you can ignore this and the nbd section of the firewall rules

```
export MY_NBD_PORT=`ss -peanut|grep xnbd-client|awk '{split($0,a); print a[6]}'|awk '{split($0,a,":"); print a[2]}'`
echo "NBD Communicating on port $MY_NBD_PORT"
```


## Firewall configuration

Next we will configure the software firewall using [iptables](https://help.ubuntu.com/community/IptablesHowTo)

A few comments about this configuration.

- We are allowing SSH, DNS, NTP and DHCP traffic
- Tezos traffic in and out is allowed on the default port 9732
- This configuration will log any dropped packets you can view them using `dmesg`

```
# Allow related connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback interface
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow SSH for now, so our connection is not disrupted, we will enable/disable SSH access via our security groups on the cloud provider.  We will tighten this up in future versions
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# Allow Tezos P2P connections
iptables -A INPUT -p tcp --dport 9732 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --dport 9732 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 9732 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# Allow NBD 
iptables -A OUTPUT -p tcp --dport $MY_NBD_PORT -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# Allow DNS
iptables -A OUTPUT -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow NTP
iptables -A INPUT -p udp --source-port 123:123 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p udp --destination-port 123:123 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow DHCP
iptables -A INPUT -i eth0 -p udp --dport 67:68 --sport 67:68 -j ACCEPT

# Setup loggers
iptables -N LOG_AND_DROP_IN
iptables -A LOG_AND_DROP_IN -j LOG --log-prefix "IPTABLES:DROP:IN:"
iptables -A LOG_AND_DROP_IN -j DROP

iptables -N LOG_AND_DROP_OUT
iptables -A LOG_AND_DROP_OUT -j LOG --log-prefix "IPTABLES:DROP:OUT:"
iptables -A LOG_AND_DROP_OUT -j DROP

iptables -N LOG_AND_DROP_FWD
iptables -A LOG_AND_DROP_FWD -j LOG --log-prefix "IPTABLES:DROP:FWD:"
iptables -A LOG_AND_DROP_FWD -j DROP

# Log and drop everything else
iptables  -A INPUT   -j LOG_AND_DROP_IN
iptables  -A FORWARD   -j  LOG_AND_DROP_FWD
iptables  -A OUTPUT  -j LOG_AND_DROP_OUT

# Default to drop everything
iptables  -P INPUT   DROP
iptables  -P FORWARD DROP
iptables  -P OUTPUT   DROP

# If you want to filter ssh traffic then use this
# iptables-save |grep -v "port 22" > /etc/iptables/rules.v4
# If you prefer keeping ssh access then just pipe the whole config out
iptables-save > /etc/iptables/rules.v4 

```

## Tezos Configuration 


```

# Setup the user tezos-node will run under
useradd -m -p $(openssl rand -base64 48 | openssl passwd -crypt -stdin) -s /bin/bash tezos

# Verify the binary and set it up to be executable
echo "e4b0cf0d39d86e4fd451f31a793053e265aa2ac4c723e92a5dba51d3f4bb9522  /usr/local/bin/tezos-node" | sha256sum -c
chown tezos:tezos /usr/local/bin/tezos-node
chmod u+x /usr/local/bin/tezos-node

# Create the tezos identiy for the node
su -c "tezos-node identity generate 26." tezos

# Setup the service
echo '[Unit]
Description=Tezos
After=network.target

[Service]
Type=simple
User=tezos
WorkingDirectory=/home/tezos
ExecStart=/usr/local/bin/tezos-node run --connections 10
Restart=always

[Install]
WantedBy=multi-user.target' > /etc/systemd/system/tezos.service

systemctl daemon-reload
systemctl enable tezos.service
systemctl start tezos.service
```

Your tezos node should now be up and running.  You can validate by checking the logs using `journalctl`, it might take a few seconds once the node starts before you start seeing blcoks validating. Output should look something like the below. 

```
# journalctl -f -u tezos.service
-- Logs begin at Wed 2018-08-01 19:37:37 UTC. --
Aug 01 19:56:35 xxx tezos-node[17054]: Aug  1 19:56:35 - node.main: Not listening to RPC calls.
Aug 01 19:56:35 xxx tezos-node[17054]: Aug  1 19:56:35 - node.main: The Tezos node is now running!
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(1): Worker started for NetXdQprcVkpa:idrHo2wg7jhU
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(2): Worker started for NetXdQprcVkpa:idtktSgf8Lsy
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(3): Worker started for NetXdQprcVkpa:idtCGVJpmEtW
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(4): Worker started for NetXdQprcVkpa:ids5NrMi5ZYV
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(5): Worker started for NetXdQprcVkpa:idshJtmNkqJa
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(6): Worker started for NetXdQprcVkpa:ids4dakBa2Wd
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(7): Worker started for NetXdQprcVkpa:idtaBvd4ZzCt
Aug 01 19:56:36 xxx tezos-node[17054]: Aug  1 19:56:36 - validator.peer(8): Worker started for NetXdQprcVkpa:idsV3asub2cz
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - validator.block: Block BLSqrcLvFtqVCx8WSqkVJypW2kAVRM3eEj2BHgBsB6kb24NqYev succesfully validated
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - validator.block: Pushed: 2018-08-01T19:56:41Z, Treated: 2018-08-01T19:56:41Z, Completed: 2018-08-01T19:57:02Z
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - prevalidator(1): switching to new head BLSqrcLvFtqVCx8WSqkVJypW2kAVRM3eEj2BHgBsB6kb24NqYev
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - prevalidator(1): Pushed: 2018-08-01T19:57:02Z, Treated: 2018-08-01T19:57:02Z, Completed: 2018-08-01T19:57:02Z
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - validator.chain(1): Update current head to BLSqrcLvFtqVCx8WSqkVJypW2kAVRM3eEj2BHgBsB6kb24NqYev (fitness 00::0000000000000001), same branch
Aug 01 19:57:02 xxx tezos-node[17054]: Aug  1 19:57:02 - validator.chain(1): Pushed: 2018-08-01T19:57:02Z, Treated: 2018-08-01T19:57:02Z, Completed: 2018-08-01T19:57:02Z
```

We recommend setting up a security group on your cloud provider that drops inbound connections to port 22 and adding your node to it when you do not need remote access to the node.

In our next post we will setup a private baking node which makes use of this public relay node.
