```
# Public node

####################
## Setup Apt
####################

apt-get update
apt-get -y upgrade

####################
## Retrieve Binaries
####################

apt-get install -y curl

# Grab binaries before enabling firewall

curl -L https://github.com/xtzbaker/tezos/releases/download/betanet/tezos-node -o /usr/local/bin/tezos-node
chmod uog+x
echo "baeace087167dfa6a1f1cbdbb0c735f702f7f9a248364c5a8d1a4a4be873633d  /usr/local/bin/tezos-node" | sha256sum -c

###########
## Firewall
###########

apt-get install -y ufw 

ufw default deny outgoing
ufw default deny incoming
# SSH - manage this in the security group for the server to enable/disable
# ufw allow in 22
# NTP
ufw allow out 123
# DNS
ufw allow out 53
# Tezos
ufw allow out 9732
ufw allow in 9732

############################
## Cleanup unwanted packages
############################

apt-get remove -y curl wget python python3 make gcc openssh-client openssh-sftp-server ssh
apt -y autoremove

####################
## Tezos Config
####################

apt-get install -y libhidapi-dev

tezos-node identity generate 26.


################
##Sumlogic Config
####################

#Setup SumoLogic
#THis needs outgoing 443 in the firwall, probably not a great idea.....
#curl -L https://collectors.us2.sumologic.com/rest/download/linux/64 -o SumoCollector.sh
#chmod +x SumoCollector.sh
#./SumoCollector.sh -q -Vsumo.accessid=<accessId> -Vsumo.accesskey=<accessKey> -VsyncSources=/home/tezos/tezos/nohup.out -Vcollector.name=tezos-public

```


# Public node

####################
## Setup Apt
####################

apt-get update
apt-get -y upgrade

####################
## Retrieve Binaries
####################

apt-get install -y libhidapi-dev libev4 curl

curl -L https://github.com/xtzbaker/tezos/releases/download/002-PsYLVpVv/tezos-node-x86_64-linux -o /usr/local/bin/tezos-node

############################
## Cleanup unwanted packages
############################

apt-get remove -y curl wget python python3 make gcc openssh-client openssh-sftp-server ssh
apt -y autoremove

###########
## Configure Firewall
###########

# Allow related connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH for now, we'll probably remove this
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# ALlow Tezos P2P connections
iptables -A INPUT -p tcp --dport 9732 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --dport 9732 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# Allow NBD - Your cloud provider probably requires this although your port may vary
iptables -A OUTPUT -p tcp --dport 4160 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# Allow DNS
iptables -A OUTPUT -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow NTP
iptables -A INPUT -p udp --source-port 123:123 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p udp --destination-port 123:123 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow DHCP
iptables -A INPUT -i eth0 -p udp --dport 67:68 --sport 67:68 -j ACCEPT

# Allow loopback interface
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

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

iptables  -P INPUT   DROP
iptables  -P FORWARD DROP
iptables  -P OUTPUT   DROP

# Make persistent after reboot
iptables-save

####################
## Tezos Config
####################

useradd -m -p $(openssl rand -base64 48 | openssl passwd -crypt -stdin) -s /bin/bash tezos

echo "e4b0cf0d39d86e4fd451f31a793053e265aa2ac4c723e92a5dba51d3f4bb9522  /usr/local/bin/tezos-node" | sha256sum -c
chown tezos:tezos /usr/local/bin/tezos-node
chmod u+x /usr/local/bin/tezos-node

su - tezos

tezos-node identity generate 26.

exit

####
## Setup the service
####
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
systemctl status tezos.service


echo '
sleep 5 # Give time to exit
ufw enable # Apply the firewall rules
apt-get remove openssh-server # uninstall sshd
' > /tmp/after-logut.sh

chmod u+x /tmp/after-logut.sh

nohup /tmp/after-logut.sh & && exit


###
# sshd config
##

echo "Protocol 2
Port 22
Protocol 2
# Disallow root SSH access
PermitRootLogin no
# Only allow tezos user
AllowUsers tezos
# Useless feature so disable it
UseDNS no
# Login grace time
LoginGraceTime 20
# Disallow password authentication
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
AuthenticationMethods publickey
PubkeyAuthentication yes
X11Forwarding No
IgnoreRhosts yes
RhostsRSAAuthentication no
HostbasedAuthentication no

# FROM Mozilla - https://infosec.mozilla.org/guidelines/openssh

# Supported HostKey algorithms by order of preference.
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key

# Specifies the available KEX (Key Exchange) algorithms.
KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha256

# Specifies the ciphers allowed
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

#Specifies the available MAC (message authentication code) algorithms
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com

# LogLevel VERBOSE logs user's key fingerprint on login. Needed to have a clear audit track of which key was using to log in.
LogLevel VERBOSE

# Log sftp level file access (read/write/etc.) that would not be easily logged otherwise.
Subsystem sftp  /usr/lib/ssh/sftp-server -f AUTHPRIV -l INFO

# Use kernel sandbox mechanisms where possible in unprivileged processes
# Systrace on OpenBSD, Seccomp on Linux, seatbelt on MacOSX/Darwin, rlimit elsewhere.
UsePrivilegeSeparation sandbox" > /etc/ssh/sshd_config

################
##Sumlogic Config
####################

#Setup SumoLogic
#THis needs outgoing 443 in the firwall, probably not a great idea.....
#curl -L https://collectors.us2.sumologic.com/rest/download/linux/64 -o SumoCollector.sh
#chmod +x SumoCollector.sh
#./SumoCollector.sh -q -Vsumo.accessid=<accessId> -Vsumo.accesskey=<accessKey> -VsyncSources=/home/tezos/tezos/nohup.out -Vcollector.name=tezos-public




@startuml

cloud Internet {
  [Tezos Network] as tezosnetwork
  [DNS Servers] as dns
  [NTP Servers] as ntp

  () "53" as dnsport
  () "123" as ntpport
  note "We are assuming time is important so allowing NTP" as ntpnote
  ntpnote .. ntp
  () "9732" as tezosport

  tezosport - tezosnetwork
  ntpport - ntp
  dnsport - dns
}

package "My Virtual Private Cloud" {

package "Public Security Group" {
  node "Public Server" as publicserver {
    () "9732" as publicrpc
    [Tezos Node] as publicnode
    publicrpc - publicnode

    publicnode -> dnsport
    publicnode => tezosport
    tezosnetwork -> publicrpc
  }

  publicserver -> ntpport
}


package "Private Security Group" {

  node "Private Server" as privateserver  {
    [Tezos Node] as privatenode

    privatenode -> publicrpc
    privateserver -> ntpport
  }
}

}
@enduml
