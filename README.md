# WIP

Controller `/etc/network/interfaces`

```
auto ens33
iface ens33 inet static
      address 192.168.69.1
```

Controller DHCP Server install

```
root@controller:~# apt install isc-dhcp-server
```

Controller DHCP config `/etc/dhcp/dhcpd.conf`

```
subnet 192.168.69.0 netmask 255.255.255.0 {
  range 192.168.69.2 192.168.69.254
  option domain-name-servers “cometstar.net.id”;
  option domain-name “cometstar.net.id”;
# option routers 192.168.69.1;
  option broadcast-address 192.168.69.255;
  default-lease-time 600;
}

~~~

host web {
  hardware ethernet 00:00:00:00:00:00; # MAC address Web VM
  fixed-address 192.168.69.2;
}

host operator {
  hardware ethernet 00:00:00:00:00:00; # MAC address Operator VM
  fixed-address 192.168.69.3;
}
```

Install Ansible on operator

```
root@operator:~# apt install ansible sudo
root@operator:~# usermod -a -G sudo (username)
```

Operator Hosts file `/etc/hosts`

```
192.168.69.1 controller
192.168.69.2 web
```

Ansible inventory file [/home/ops/hosts.ini](https://github.com/Sprtcrnbry/UPASJ/blob/main/hosts.ini)

Ansible playbook file [/home/ops/create_300_users.yaml](https://github.com/Sprtcrnbry/UPASJ/blob/main/create_300_users.yaml)

Ansible playbook file [/home/ops/dns_server.yaml](https://github.com/Sprtcrnbry/UPASJ/blob/main/dns_server.yaml)

Operator SSH public key generate

```
ops@operator:~$ ssh-keygen
```

Operator SSH public key copy to servers

```
ops@operator:~$ ssh-copy-id controller
ops@operator:~$ ssh-copy-id web
```

DNS config master `(/home/ops/named.conf.master > Controller /etc/bind/named.conf.local)`

```
zone "cometstar.net.id" {
type master;
file "/etc/bind/db.internal";
allow-transfer { 192.168.69.2; };
also-notify { 192.168.69.2; };
};
```

DNS config slave `(/home/ops/named.conf.slave > Web /etc/bind/named.conf.local)`

```
zone "cometstar.net.id" {
type slave;
file "/var/cache/bind/db.internal";
masters { 192.168.69.1; };
};
```

Copy default DNS config (install bind9 on operator for config)

```
ops@operator:~$ sudo apt install bind
ops@operator:~$ cp /etc/bind/db.local /home/ops/db.internal
```

DNS config record `(/etc/bind/db.internal)`

```
$TTL 604800
@   IN   SOA  cometstar.net.id. cometstar.net.id. (
                    2       ; Serial
               604800       ; Refresh
                86400       ; Retry
              2419200       ; Expire
               604800 )     ; Negative Cache TTL
;
@             IN  NS    controller.cometstar.net.id.
@             IN  A     192.168.69.2
www           IN  A     192.168.69.2
mail          IN  A     192.168.69.2
controller    IN  A     192.168.69.1
web           IN  A     192.168.69.2
operator      IN  A     192.168.69.3
```

Run playbook on operator with

```
ops@operator:~$ ansible-playbook -i hosts.ini -K create_300_users.yaml
ops@operator:~$ ansible-playbook -i hosts.ini -K dns_server.yaml
```

Check users on controller with

```
root@controller:~# ls /home
```

Install postfix, dovecot on web

~~~
root@web:~# apt install postfix dovecot-imapd dovecot-pop3d
root@web:~# dpkg-reconfigure postfix
~~~

Make sure Email has either STARTTLS or SSL/TLS authentication enabled and working.

Postfix config `/etc/postfix/main.cf`

```
home_mailbox = Maildir/
```

Postfix config `/etc/postfix/master.cf`

```
submission inet n - y - - smtpd
# -o syslog_name=postfix/submission

- o smtpd_tls_security_level=encrypt
- o smtpd_sasl_auth_enable=yesWWW index /var/www/html/index.html

~~~

smtps inet n - y - - smtpd
# -o syslog_name=postfix/smtps

- o smtpd_tls_wrappermode=yes
- o smtpd_sasl_auth_enable=yes
```

Dovecot config `/etc/dovecot/conf.d/10-mail.conf`

```
mail_location = maildir:~/Maildir
```

Dovecot config `/etc/dovecot/conf.d/10-ssl.conf`

```
ssl_cert = </etc/dovecot/private/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.key
```

Install web server

```
root@web:~# apt install apache
```

Web server content

```
# echo “<h1>Welcome to the night sky</h1>” > /var/www/html/index.html
```

You can check index.html if it contains or if accessed shows “Welcome to the night sky” and check if https is working.

Install mariadb and roundcube

```
root@web:~# apt install mariadb-server
root@web:~# apt install roundcube
```

`/etc/apache2/sites-available/mail.conf`

```
ServerName mail.cometstar.net.id

ServerAdmin webmaster@cometstar.net.id
DocumentRoot /var/lib/roundcube/public_html
```

Install mariadb and roundcube

```
root@web:~# a2ensite mail
```

`/etc/roundcube/config.inc.php`

```
$config['default_host'] = 'mail.cometstar.net.id';
$config['smtp_server'] = 'mail.cometstar.net.id';
$config['smtp_port'] = 25;
$config['smtp_user'] = '';
$config['smtp_pass'] = '';
```

ProFTPD is recommended, but if user has already configured vsftpd/pureftpd or similar
software skip to testing.

`/etc/proftpd/proftpd.conf`

```
DefaultRoot ~
```

Make sure FTP can login using users created by ansible and read/write into the nested
directory. You can use FileZilla program on Operator.

```
ops@operator:~$ sudo apt install filezilla
```



