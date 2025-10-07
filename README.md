# App Passwords for Dovecot

> [!WARNING]
> These scripts are more or less unmaintained. You should prefer using [Dovecot Web Auth](https://github.com/bennet0496/dovecot_web_auth)
> instead

This project's purpose is to provide a configuration example for dovecot to run with app passwords (2FA). 
It is designed to interoperate with the Roundcube Plugin [mpipks/imap_apppasswd](https://github.com/bennet0496/imap_apppasswd).

Caveat: This repo sets up a normal SQL passdb and post-login script in Dovecot. However, It will be 
difficult/impossible to have precise last-login tracking this way, as it is only reliably possible to 
get Dovecot to run a post-login script for the IMAP (and POP3) service. If for example an MTA (e.g. exim) 
is connected via SASL, these logins would not be tracked.



## Prepare the database
For the database, you can use any host you'd like to hold the data. This doesn't necessarily need 
to be the same host, Roundcube or Dovecot are running on; however, both will need database access.
This host will need to have mariadb (or mysql) installed
```bash
apt install mariadb-server
```

Then create the database, users e.g. with
```sql
CREATE DATABASE mail_auth;
GRANT USAGE ON *.* TO `mailserver`@`localhost` IDENTIFIED BY 'password123';
GRANT USAGE ON *.* TO `roundcube`@`webmail.example.com` IDENTIFIED BY 'password123';

GRANT SELECT ON `mail_auth`.`log` TO `roundcube`@`webmail.example.com`;
GRANT SELECT, SHOW VIEW ON `mail_auth`.`app_passwords_with_log` TO `roundcube`@`webmail.example.com`;
GRANT SELECT, INSERT, UPDATE (`comment`), DELETE ON `mail_auth`.`app_passwords` TO `roundcube`@`webmail.example.com`;

GRANT SELECT ON `mail_auth`.`app_passwords` TO `mailserver`@`localhost`;
GRANT SELECT, INSERT ON `mail_auth`.`log` TO `mailserver`@`localhost`;
```
And the tables from the DDL in `database/DDL.sql`. Replace the passwords and host specifiers
appropriately.

## Setup your mail server
You will need the following to be installed on you mail server
```bash
apt install dovecot-mysql geoip-database geoipupdate
```

### Configure Dovecot
#### Configure the passdb
Now you'll need to configure Dovecot to use a SQL passdb that holds our application passwords.
In `/etc/dovecot/conf.d/10-auth.conf` uncomment the SQL include line
```
# ...
!include auth-sql.conf.ext
# ...
# ...
```
And add the passdb (and only the passdb) to the `auth-sql.conf.ext` file
```
# Authentication for SQL users. Included from 10-auth.conf.
#
# <doc/wiki/AuthDatabase.SQL.txt>

passdb {
  driver = sql

  # Path for SQL configuration file, see example-config/dovecot-sql.conf.ext
  args = /etc/dovecot/dovecot-sql.conf.ext
  override_fields = userdb_passdb=sql
  skip = authenticated
}
```
This species the SQL configuration and sets a custom userdb field that we'll use later in the
postlogin process to detect usage of the passdb. And lastly we want this passdb to be skipped if
another passdb, e.g. LDAP has already authenticated the user, i.e. when they log in with their
real password to Roundcube. It would also make sense to limit the passdb with the real passwords
to only be usable from out Roundcube
```
passdb {
  driver = ldap

  # Path for LDAP configuration file, see example-config/dovecot-ldap.conf.ext
  args = /etc/dovecot/dovecot-ldap.passdb.conf.ext
  
  # Replace with Roundcube IP
  override_fields = allow_nets=1.2.3.4/32 userdb_passdb=ldap
}
```
Again we mark which passwd was used, and we limit it to only successfully authenticate from
out webmailer's IP.

Now we configure the actual SQL statements Dovecot will use to authenticate against our passdb.
In `/etc/dovecot/dovecot-sql.conf.ext` set
```
driver = mysql
connect = host=127.0.0.1 dbname=mail user=mailserver password=password123

password_query = password_query = SELECT uid AS username, password, id AS userdb_passid \
  FROM app_passwords \
  WHERE uid = '%n' AND REGEXP_SUBSTR(password, '[$].*') = ENCRYPT('%w', REGEXP_SUBSTR(password, '[$].*[$]'))
```
Replace the `connect` line appropriately. This retrieves the username, password and password ID as
userdb attribute to we can tie the login to a specific password in our post-login script. We
need to pass the password to the database, as dovecot does not support the result to be multiple 
lines. This mean we transmit the password in plain text to hash in query, therefore the connection 
should be local or SSL/TLS encrypted.

#### Post-login Script (Last login tracking)
We want to show the user when each password was last used and from where. For this we need to
create a [post-login script for the IMAP](https://doc.dovecot.org/admin_manual/post_login_scripting/) 
(and POP3 if you into that sort of thing) service in Dovecot. In `/etc/dovecot/conf.d/10-master.conf` 
to the IMAP service, add `imap-postlogin` the executable line. And create a service `imap-postlogin`
pointing to the script
```
# ...
service imap {
  # Most of the memory goes to mmap()ing files. You may need to increase this
  # limit if you have huge mailboxes.
  #vsz_limit = $default_vsz_limit

  executable = imap imap-postlogin

  # Max. number of IMAP processes (connections)
  process_limit = 1024
}

service imap-postlogin {
  # all post-login scripts are executed via script-login binary
  executable = script-login /usr/local/bin/postlogin.sh

  # the script process runs as the user specified here (v2.0.14+):
  user = dovecot
  # this UNIX socket listener must use the same name as given to imap executable
  unix_listener imap-postlogin {
  }
}

#...
```
The script `dovecot/postlogin.sh` gathers the information and executes the parameter passed 
from the Dovecot script-login binary to finish the process. It depends on the `dovecot/geopip.py`
to gather Geographic information. You will need to have MaxMind GeoLite setup for this to work.
In this script you can also configure names for local networks
```python
local_networks = {
    "Network 1":                 ["172.17.1.0", 24],
    "Network 2":                 ["10.8.0.0", 16],
    "Network 3":                 ["192.168.0.0", 24],
}
```
If your Database server is not local, you will also need to modify the `postlogin.sh` to use the 
correct host and password.

### A word on the SMTP Server
If you use SASL login from your SMTP Server you are mostly set. It will just use the passdb aswell,
however, last-login tracking will not work, as I was unable to get dovecot to fire a post-login
script with the auth service.

If you have separate authentication in you SMTP server, you'll have to set it up to use the database
as well. And for good measure you might also want to look into firing a post-login script from there.

