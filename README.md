# Postfix-Mail-Server
Setting up a Mail Server using Postfix, Courier IMAP, MySQL and Thunderbird.

------
  
  
## Introduction  

In the following, we are going to configure a mail server using **Postfix**, **Courier IMAP**, and **MySQL** on Ubuntu.  
To test the configuration we will be using Thunderbird as a mail client.

## Prerequisites
- A primary DNS server

These are the details that will be used for configuration:

- Domain: ***me.lan***
- MX: ***mail.me.lan***
- Email accounts: user0@me.lan, user1@me.lan
- MySQL database: ***mailserver***
- MySQL user: ***mailuser***
- MySQL password: ***password***

## DNS records
Add **A** and **MX** Records as follows :
- Forward zone
```console
mail    IN      A       192.168.225.40
@       IN      MX      10      mail
```
- Reverse zone
```console
40      IN      PTR     mail.me.lan.
```

Restart the service  
`systemctl restart bind9`  

Make Sure MX Records are Set  
- `dig MX +short me.lan`    
*10 mail.me.lan.* 
- `dig A +short mail.me.lan`  
*192.168.225.40*  

## MySQL
#### Installation
```sh 
apt-get install mysql-client mysql-server 
```
#### Create mysql database and user for Postfix
- **Create a file `create.sql` containing the following**
```sql
CREATE DATABASE mailserver;
CREATE USER 'mailuser'@'127.0.0.1' IDENTIFIED BY 'password';
GRANT SELECT ON mailserver.* TO 'mailuser'@'127.0.0.1';
FLUSH PRIVILEGES;
```
- **Create the database**
```sh
mysql -u root -p < create.sql
```
  
#### Create database tables :
- **create a file `postfix-mysql.sql` containing the following sql code**
```sql
USE mailserver;

CREATE TABLE `virtual_domains` (
 `id`  INT NOT NULL AUTO_INCREMENT,
 `name` VARCHAR(50) NOT NULL,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;;

CREATE TABLE `virtual_users` (
 `id` INT NOT NULL AUTO_INCREMENT,
 `domain_id` INT NOT NULL,
 `email` VARCHAR(120) NOT NULL,
 `clearpw` VARCHAR(106) NOT NULL,
 `cryptpw` VARCHAR(106) NOT NULL,
 `uid` smallint(5) unsigned NOT NULL default '5000',
 `gid` smallint(5) unsigned NOT NULL default '5000',
 `home` varchar(255) NOT NULL default '/var/spool/mail/vmail/',
 `maildir` varchar(255) NOT NULL default 'Maildir/',
 PRIMARY KEY  (`id`),
 UNIQUE KEY `email` (`email`),
 FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
- **Create the tables**
```sh
mysql -u root -p < postfix-mysql.sql 
```
#### Insert mail users  
- **Create a file `insert.sql` containing the following**  
```sql
USE mailserver;

INSERT INTO `virtual_domains` (`name`) VALUES ('me.lan');

INSERT INTO `virtual_users`(`domain_id`,`email`, `clearpw`, `cryptpw`,`maildir`) 
VALUES ('1','user0@me.lan','password@0',SHA2('password@0', CONCAT('$5$', MD5(RAND()))),CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/'));

INSERT INTO `virtual_users`(`domain_id`,`email`, `clearpw`, `cryptpw`,`maildir`) 
VALUES ('1','user1@me.lan','password@1',SHA2('password@1', CONCAT('$5$', MD5(RAND()))),CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/'));
```
- **Insert the users into the database**
```sh
mysql -u root -p < insert.sql
```

#### Enable MySQL logging for troubleshooting Postfix setup
Open *`/etc/mysql/mysql.conf.d/mysqld.cnf`* and uncomment the following lines: 
```properties
general_log_file        = /var/log/mysql/query.log
general_log             = 1
```
#### Restart MySQL:
``` /etc/init.d/mysql restart ```

## Postfix
#### Install Postfix. Select internet site when asked.
```sh
apt-get install postfix postfix-mysql
```
#### Set up the folder for the virtual mail:
```sh
cp /etc/aliases /etc/postfix/aliases  
postalias /etc/postfix/aliases  
mkdir /var/spool/mail/vmail/  
groupadd --system virtual -g 5000  
useradd --system virtual -u 5000 -g 5000  
chown -R virtual:virtual /var/spool/mail/vmail/  
``` 
#### Configuration file : `main.cf`
The configuration file `main.cf` should look :
```properties
smtpd_banner = $myhostname ESMTP $mail_name
biff = no
relayhost = 
inet_interfaces = all
mynetworks_style = host
inet_protocols = ipv4
append_dot_mydomain = no
delay_warning_time = 4h
readme_directory = no

smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls=yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

alias_maps = hash:/etc/postfix/aliases
alias_database = hash:/etc/postfix/aliases
virtual_mailbox_base = /var/spool/mail/vmail
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf

virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

local_recipient_maps =
mydestination =

mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +

unknown_local_recipient_reject_code = 450
maximal_queue_lifetime = 7d
minimal_backoff_time = 1000s
maximal_backoff_time = 8000s
smtp_helo_timeout = 60s
smtpd_recipient_limit = 10
smtpd_soft_error_limit = 3
smtpd_hard_error_limit = 12

smtpd_helo_restrictions = permit_mynetworks, warn_if_reject reject_non_fqdn_hostname, reject_invalid_hostname, permit
smtpd_sender_restrictions = permit_mynetworks, warn_if_reject reject_non_fqdn_sender, reject_unknown_sender_domain, reject_unauth_pipelining, permit
smtpd_client_restrictions = reject_rbl_client sbl.spamhaus.org, reject_rbl_client blackholes.easynet.nl
smtpd_recipient_restrictions = reject_unauth_pipelining, permit_mynetworks, reject_non_fqdn_recipient, reject_unknown_recipient_domain, reject_unauth_destination, permit
smtpd_data_restrictions = reject_unauth_pipelining

smtpd_helo_required = yes
smtpd_delay_reject = yes
disable_vrfy_command = yes
```

#### Postfix’s MySQL Configuration
We are going to create the final three files that we append in the `main.cf` file to tell Postfix how to connect with MySQL.  
First we need to create the `mysql-virtual-mailbox-domains.cf` file.  
```sh
nano /etc/postfix/mysql-virtual-mailbox-domains.cf
```
```properties
user = mailuser
password = password
hosts = 127.0.0.1
dbname = mailserver
query = SELECT name FROM virtual_domains WHERE name='%s'
```
Then we need to create the `mysql-virtual-mailbox-maps.cf` file.  
```sh
nano /etc/postfix/mysql-virtual-mailbox-maps.cf
```
```properties
user = mailuser
password = password
hosts = 127.0.0.1
dbname = mailserver
query = SELECT maildir FROM virtual_users WHERE email='%s'
```
We need to restart Postfix  
`service postfix restart`  
We need to ensure that Postfix finds your domain, so we need to test it with the following command. If it is successful, it should returns ***me.lan***:  
```sh
postmap -q me.lan mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
```
At this moment we are going to ensure Postfix finds your mail users address with the following commands. They should return respectively ***me.lan/user0/*** and ***me.lan/user1/***.  
```sh
postmap -q user0@me.lan mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
postmap -q user1@me.lan mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
```
  
## Courier IMAP
#### Installation
```sh
apt-get install courier-base courier-authdaemon courier-authlib-mysql courier-imap
```
#### Configuration
Open `/etc/courier/imapd` and change the following:  
```properties
ADDRESS=0
```
to  
```properties
ADDRESS=0.0.0.0
```
This will make Courier IMAP to listen on **IPv4** only.  

The file `/etc/courier/authdaemonrc` should be as follows:  
```properties
authmodulelist="authmysql"
authmodulelistorig="authuserdb authpam authpgsql authldap authmysql authsqlite authcustom authpipe"
daemons=5
authdaemonvar=/run/courier/authdaemon
DEBUG_LOGIN=0
DEFAULTOPTIONS=""
LOGGEROPTS=""
```
The file `/etc/courier/authmysqlrc` should looks like:
```properties
MYSQL_SERVER		127.0.0.1
MYSQL_USERNAME		mailuser
MYSQL_PASSWORD		password
MYSQL_PORT		0
MYSQL_OPT		0
MYSQL_DATABASE		mailserver
MYSQL_USER_TABLE	virtual_users
MYSQL_CLEAR_PWFIELD	clearpw
MYSQL_UID_FIELD		uid
MYSQL_GID_FIELD		gid
MYSQL_LOGIN_FIELD	email
MYSQL_HOME_FIELD	home
MYSQL_MAILDIR_FIELD	maildir
```
#### Final step
To ensure that errors of type ***Filesystem notification initialization*** will not occure in mail client side, install **Gamin** package.
```sh
sudo apt-get install gamin
```
And then restart ***courier-imap*** service.
```sh
sudo /etc/init.d/courier-imap restart
```
## Testing
- Ensure that all services are restarted.  
```sh
/etc/init.d/mysql restart
```
```sh
/etc/init.d/postfix restart
```
```sh
/etc/init.d/courier-authdaemon restart
```
```sh
/etc/init.d/courier-imap restart
```
- Make sure Postfix, CourierIMAP and MySQL Services are Running  
`netstat -nltp | egrep '25|143|3306'`

<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/3bf1d613384b81d9086d6a133e02c6a374aaa048/screenshots/ports.PNG"></p>

### Thunderbird
- **Installing Thunderbird :**
```sh
sudo apt-get install thunderbird
```
- **Creating an account :**  
  
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/5876172c67053d01d1dc63d076661e971f47c734/screenshots/create_account_1.PNG"></p>  
  
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/5876172c67053d01d1dc63d076661e971f47c734/screenshots/create_account_2.PNG"></p>

- **Using Telnet to generate necessary folders and files for each user**   

Send two messages  between **user0** and **user1** using ***telnet***, alternating the sender and the reciever each time.
  
Example : from user0@me.lan to user1@me.lan
      
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/6d8c732f1c447eeb75a45f60510eac6e3daf2796/screenshots/telnet_compose_h.PNG"></p>
  
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/6d8c732f1c447eeb75a45f60510eac6e3daf2796/screenshots/telnet_sent.PNG"></p>

After these two operations a `me.lan` directory must be created in ***/var/spool/mail/vmail*** with two subdirectories `user0` and `user1` having the following structure:  

<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/64484e74d7f00dadd82a442398896a4e1193cfc0/screenshots/folders.PNG"></p>
  
- **Compossing a message :**  
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/8ef09f715fce6daee7e76984b8082745a1308fcf/screenshots/mail_compose.PNG"></p>

- **Checking if the message is sent :**  
1. ***user0***'s sent box :
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/64484e74d7f00dadd82a442398896a4e1193cfc0/screenshots/mail_sent.PNG"></p>

2. ***user1***'s inbox :
<p align="center"><img width="80%" src="https://github.com/ohidaoui/Postfix-Mail-Server/blob/64484e74d7f00dadd82a442398896a4e1193cfc0/screenshots/mail_recieved.PNG"></p>
