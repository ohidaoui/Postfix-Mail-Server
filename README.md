# Postfix-Mail-Server
Setting up a Mail Server using Postfix, Courier IMAP, MySQL and Thunderbird.

------
  
  
## Introduction  

In the following, we are going to configure a mail server using **Postfix**, **Courier IMAP**, and **MySQL** on Ubuntu.  
To test the configuration we will be using Thunderbird as a mail client.

## Prerequisites
- A primary DNS server

### A and MX Records
Add **A** and **MX** Records as follows :
- Forward zone
```
mail    IN      A       192.168.225.40
@       IN      MX      10      mail
```
- Reverse zone
```
40      IN      PTR     mail.me.lan.
```
