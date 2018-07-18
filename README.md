# Public node

####################
## Setup Apt
####################

apt-get update
apt-get -y upgrade

apt-get install -y unattended-upgrades

echo 'APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";' > /etc/apt/apt.conf.d/20auto-upgrades

####################
## Retrieve Binaries
####################

apt-get install -y curl

# Grab binaries before enabling firewall

curl -L https://github.com/xtzbaker/tezos/releases/download/betanet/tezos-node -o /usr/local/bin/tezos-node
echo "baeace087167dfa6a1f1cbdbb0c735f702f7f9a248364c5a8d1a4a4be873633d  /usr/local/bin/tezos-node" | sha256sum -c

###########
## Firewall
###########

apt-get install -y ufw 

ufw default deny outgoing
ufw default deny incoming
# SSH - manage this in the security group for the server to enable/disable
ufw allow in 22
# NTP
ufw allow out 123
# DNS
ufw allow out 53
# Tezos
ufw allow out 9732
ufw allow in 9732

ufw enable

# TODO any other hardening
# TODO sort out the sshd config
# TODO remove any dodgy ssh keys (added by cloud provider for example)
# TODO encrypt?

############################
## Cleanup unwanted packages
############################

apt-get remove -y curl wget python python3 make gcc
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





