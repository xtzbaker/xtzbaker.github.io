---
layout: post
title:  "Unified Logging for Tezos Nodes"
date:   2018-08-05 07:24:29 +0000
author: xtzbaker
---

To fully secure our nodes we are aiming to remove remote shell access. In order to retain the capability to monitor the server we need to export the logging information somewhere, so before we outline the steps to setup your private node we will cover this topic.  There are a variety of solutions for this:

* [Loggly](https://www.loggly.com/)
* [Splunk](https://www.splunk.com/)
* [Sumologic](https://www.sumologic.com/)
* [ELK](https://www.elastic.co/webinars/introduction-elk-stack)

In the first instance we are going to use [Loggly](https://www.loggly.com/), primarily because it has a very small footprint, but it also has decent free tier.  In addition we can restrict the outbound traffic to a single port which makes the firewall configuration easier.  Ultimately we will move to either Splunk or ELK which we can host and manage ourselves, giving us full control, but until then Loggly will suffice enabling us to move forward with the more interesting parts of securing your baking infrastructure.

It is recommended to do this setup before any other configuration on your server - we will provide a new tutorial for the [Public Node](2018-08-01-secure-tezos-public-node.md) incorpoating this setup in a subsequent post once we have a few other pieces in place.

After signing up for your account on Loggly you will be given a [Customer Token](https://www.loggly.com/docs/customer-token-authentication-token/).  We will add this to an environment variable for use in the scripts that follow:

```bash
export LOGGLY_TOKEN="XXXX" # getthis from your Loggly admin pages
```

Next we will configure the necessary prerequistes for communicating with Loggly securely

```bash
# Install rsyslog-gnutls for communicating over TLS see https://www.loggly.com/docs/rsyslog-tls-configuration/
sudo apt-get install -y rsyslog-gnutls

## Install the Loggly SSL certificates
sudo mkdir -pv /etc/rsyslog.d/keys/ca.d
sudo curl -L https://logdog.loggly.com/media/logs-01.loggly.com_sha12.crt -o /etc/rsyslog.d/keys/ca.d/logs-01.loggly.com_sha12.crt
echo "3bcd557ea8e43599d3fb98fd13857c2fbff5637cb307a3d4bae23e3c8b5c34cc  /etc/rsyslog.d/keys/ca.d/logs-01.loggly.com_sha12.crt" | sha256sum -c
```

Here we configure our software firewall to allow the rsyslog TLS traffic outbound on port 6514

```bash
# Setup outbound traffic on port 6514 for sending the log data to Loggly
sudo iptables -A OUTPUT -p tcp --dport 6514 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 6514 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Finally configure rsyslog to send log data to Loggly, make sure you have set the `LOGGLY_TOKEN` environment variable

```bash
# Modify the rsyslog MaxMessageSize setting
sed -i '/^$MaxMessageSize.*/d' /etc/rsyslog.conf # Delete
sed -i '1s;^;$MaxMessageSize 64k\n;' /etc/rsyslog.conf # Prepend

# Setup the syslog export
printf '# Setup disk assisted queues
$WorkDirectory /var/spool/rsyslog # where to place spool files
$ActionQueueFileName fwdRule1     # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g       # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on     # save messages to disk on shutdown
$ActionQueueType LinkedList       # run asynchronously
$ActionResumeRetryCount -1        # infinite retries if host is down

#RsyslogGnuTLS
$DefaultNetstreamDriverCAFile /etc/rsyslog.d/keys/ca.d/logs-01.loggly.com_sha12.crt

template(name="LogglyFormat" type="string"
string="<%%pri%%>%%protocol-version%% %%timestamp:::date-rfc3339%% %%HOSTNAME%% %%app-name%% %%procid%% %%msgid%% [%s@41058 tag=\\"RsyslogTLS\\"] %%msg%%\\n"
)

# Send messages to Loggly over TCP using the template.
action(type="omfwd" protocol="tcp" target="logs-01.loggly.com" port="6514" template="LogglyFormat" StreamDriver="gtls" StreamDriverMode="1" StreamDriverAuthMode="x509/name" StreamDriverPermittedPeers="*.loggly.com")
' "$LOGGLY_TOKEN" >/etc/rsyslog.d/22-loggly.conf

service rsyslog restart
```

You should now start seeing your logs appear on your Loggly account.  You can test it by executing the following which should log the text `BakeJar` to Loggly:

```
logger BakeJar
```