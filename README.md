# disposable-mail-cli
Temporary/disposable mailbox services are nearly blocked everywhere and there are also lists which collect these services, so an admin can block them.
I wanted to create a disposable mailservice myself, which is easy to use, secure and also compatible on unsecured/strange sites too.

We only use `postfix` and `mutt` for that, you will love it! 😃

## Here we go!
My setup is based on Ubuntu 20.04 LTS, but you can adopt it to nearly any OS, because I only use common services/packages.

Because we only need an SMTP server, the geographical location is not really important, but I can recommend the Hetzner Cloud (2,49€/m ex. VAT) which is using a DC in Germany 🇩🇪 or Finland 🇫🇮, also with high availability with CEPH disks!

## Requirements
### DNS
I want to go cheap and secure, so I can recommend [OVH DNS](https://www.ovh.com/world/domains/prices/) with an `.ovh` TLD, because the zone file management is great and they implement DNSSEC for free!
You can delete all DNS zone entries, we only need the following ones: (`cybacoffee.ovh` is my example!)
```
mail.cybacoffee.ovh.  IN	A     1.2.3.4	
cybacoffee.ovh.       IN	MX    10 mail.cybacoffee.ovh.
```

### VPS
We are creating a VPS with the hostname `mail.cybacoffee.ovh` and an static IP.

# Setup mailserver
We are updating our server and install needed packages:

```
# apt update && apt upgrade -y
# apt install postfix mutt mailutils fail2ban 
```

During the install, Postfix should ask us, which mail configuration we want to use, select "Internet Site". If there isn't any input mask, use `dpkg-reconfigure postfix` to force it.
System Mail name is the TLD `cybacoffee.ovh`, without the mail subdomain.

For our disposable "feature", we are using a "catch all" feature, so we can receive any mail on any alias `( e.g. fnweifoewifew@cybacoffee.ovh`) without adding a new mailbox.

We add a OS user, which is receiving all mails and we can use our `mutt` client, to read our mails via SSH, no web GUI etc required. (wuhu, no web vulnerabilites!)
To be RFC conform, we add the usual aliases and redirect them to our real mail like `<redirectMail@protonmail.com>`.

```
# adduser --disabled-password <user>
# vi /etc/postfix/virtual
```

```
@cybacoffee.ovh <user>

# rfc2142 aliases
info@cybacoffee.ovh       <redirectMail@protonmail.com>
marketing@cybacoffee.ovh  <redirectMail@protonmail.com>
sales@cybacoffee.ovh      <redirectMail@protonmail.com>
support@cybacoffee.ovh    <redirectMail@protonmail.com>
abuse@cybacoffee.ovh      <redirectMail@protonmail.com>
noc@cybacoffee.ovh        <redirectMail@protonmail.com>
security@cybacoffee.ovh   <redirectMail@protonmail.com>
postmaster@cybacoffee.ovh <redirectMail@protonmail.com>
hostmaster@cybacoffee.ovh <redirectMail@protonmail.com>
usenet@cybacoffee.ovh     <redirectMail@protonmail.com>
news@cybacoffee.ovh       <redirectMail@protonmail.com>
webmaster@cybacoffee.ovh  <redirectMail@protonmail.com>
www@cybacoffee.ovh        <redirectMail@protonmail.com>
uucp@cybacoffee.ovh       <redirectMail@protonmail.com>
ftp@cybacoffee.ovh        <redirectMail@protonmail.com>
```

```
# postmap /etc/postfix/virtual
```

Now we add the following line to our postfix main config `/etc/postfix/main.cf`:
```
virtual_alias_maps = hash:/etc/postfix/virtual
```

Now restart postfix `systemctl restart postfix`.

## Delete mails regularly
If you want, you can delete all mails regulary `(@weekly, @monthly, @daily)` with an short crontab entry:
```
<user># crontab -e
@weekly > /var/spool/mail/<user>
```

## Securing our Server
### Secure TLS Postfix config
Use the TLS Generator from Mozilla: https://ssl-config.mozilla.org
I recommend to use the `Intermediate` mode, because TLS should also be backward compatible.

### Generate Let's Encrypt TLS certificate
Use the `acme.sh` script from @Neilpang https://github.com/acmesh-official/acme.sh and install it.
I recommend, when possible, use the DNS API from your DNS Hoster, like OVH, to challenge the certificate creation, so you haven't to open up an additional HTTP port and also can use it behind a firewall.

E.g. generate one for [OVH](https://github.com/acmesh-official/acme.sh/wiki/How-to-use-OVH-domain-api):
```
acme.sh --issue -d cybacoffee.ovh -d mail.cybacoffee.ovh --dns dns_ovh
```

Add the new certificate chain to your postfix `main.cf`:
```
smtpd_tls_cert_file=/root/.acme.sh/cybacoffee.ovh/fullchain.cer
smtpd_tls_key_file=/root/.acme.sh/cybacoffee.ovh/cybacoffee.ovh.key
```

#### Test your TLS security
Use the sites like https://ssl-tools.net/mailservers and https://mxtoolbox.com/SuperTool.aspx (`Test Email Server`) to verify your security level, everything should be `green`.


### Firewall
Enable incoming SMTP and SSH and block everything else with UFW:
```
# ufw allow ssh
# ufw allow smtp
# ufw enable
```

### Block brute force / malicious IPs with fail2ban
After installing fail2ban, SSH is automatically be watched. We are activating SMTP additionally:

```
# cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
# cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
# vi /etc/fail2ban/jail.local
```
Search for `[postfix]` and `[postfix-rbl]` and enable it with `enabled = true`. Under the `[DEFAULT]` section, you can set the global settings like the bantime `bantime  = 1y`.

### Ubuntu Live Kernel Patches
If you are using Ubuntu, implement the [Live Kernel Patches](https://ubuntu.com/security/livepatch)! 🔒

Done!

Have Fun! 😁
