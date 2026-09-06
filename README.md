# Postdove

A set of, somewhat opinionated, **bash** scripts for installing a clean Postfix + Dovecot environment  
with as few *moving parts* as possible, while still providing a fully functioning mailing system capable of hosting mail for multiple domains.  

&nbsp;
# Requirements
Tested to work on fresh, clean **Ubuntu 20.04** and **Debian 10 "buster"** official cloud images, with the following packages:
| package                    | version | source                        |
|----------------------------|---------|-------------------------------|
| postfix                    | 3.4.13  | Ubuntu repository             |
|                            | 3.4.14  | Debian repository             |
| dovecot                    | 2.3.14  | Dovecot's official repository |
|                            | 2.3.15  | Dovecot's official repository |
| postfix-policyd-spf-python | 2.9.2   | Ubuntu repository             |
|                            |         | Debian repository             |
| opendkim                   | 2.11.0  | Ubuntu repository             |
|                            |         | Debian repository             |
| spamass-milter             | 0.4.0   | Ubuntu repository             |
|                            |         | Debian repository             |
| spamassassin               | 3.4.4   | Ubuntu repository             |
|                            |         | Debian repository             |

As a general rule, the installer and the management tools should work without modifications 
and should produce a fully functioning emailing environment on a *debian* based system that meets the following package requirements:  

| package  | version   | status                                            |
|----------|-----------|---------------------------------------------------|
| apt      | >= 1.5    | installed                                         |
| openssl  | >= 1.1.1  | installed                                         |
| whiptail | >= 0.52   | installed                                         |
| awk      | >= 4.1.0  | installed                                         |
| postfix  | >= 3.3    | present in the distribution's official repository |

For example, it should probably also work with **Ubuntu 18.04**.  

However, all the possible package combinations were not tested. 
Please note that incompatibilities could also arise due to the different dependencies of the main packages listed below.

### No guarantees provided !  

&nbsp;
# Packages
The following main packages, along with their dependencies, will be installed:
- postfix
- postfix-policyd-spf-python
- opendkim
- spamassassin
- spamass-milter
- dovecot &sup1;
- unbound &sup2;

### Notes:
1. Dovecot is installed from Dovecot's official repository located at https://repo.dovecot.org/  
2. Installation of **Unbound is optional** but highly recommended. It can be skipped during installation.

&nbsp;
# Install
Before running the installer make sure that:
1. Your server's host name and its fully-qualified domain name are properly configured. Running `hostname -f` should return something of the form `hostname.domain.tld`
2. You have properly set up **DNS** records for your server's fully-qualified domain name (**A** and **PTR**) and its parent domain (at least **MX**). You should also add a **SPF** record and, after the installer
completes, the generated **DKIM** record for the parent domain. This step of adding MX, SPF and DKIM records must be repeated for every new domain that you add to your system.
  
Next, run the installer as `root` and follow the on-screen instructions.

&nbsp;
# Resulting setup
The default setup is mainly focused on **hosted domains**.  
See http://www.postfix.org/VIRTUAL_README.html for more details.  

At the filesystem level, each hosted domain is owned by a randomly generated local UNIX account.  
  
Hosted domains, along with their mailboxes, are stored in `/home/virtualmail` .  
As far as the management tools are concerned, this directory is specified by the **virtual_mailbox_base** Postfix parameter, 
even when the final delivery is not done through Postfix's **virtual** delivery agent. 
Most of the directory based operations performed by the **postnimda** management tools are relative to this location. 
This parameter should be kept in sync with Dovecot's **mail_home** and **mail_location** parameters.


If you plan on using the included **postnimda** management tools for managing domains and mailboxes, you should really not edit the main configuration files manually. 
Some parts of the management tools might rely on the configuration files being set up in a certain way.
  
Hosted domain mailboxes are stored in **Maildir** format.  
While Postfix and Dovecot maintain their own separate list of users, these lists are in sync and should always remain in sync.


A few more notes about the resulting setup:

### Postfix
All configuration and Postfix related data files are located in `/etc/postfix`.  
Indexed files based on hashing are used for storing the necessary configuration.  
You can read more about this at http://www.postfix.org/DATABASE_README.html#types
  
Local aliases are stored in `/etc/postfix/smtpd_local_aliases`.  
Virtual aliases are stored in `/etc/postfix/smtpd_map_virtual_aliases`.  
  
Locally submitted mail to local users - for example through `sendmail` for recipients such as `user@localhost` or `user@[127.0.0.1]` - can be delivered through Postfix's **local** delivery agent 
to **UNIX-style mailboxes** located in the directory specified by the **mail_spool_directory** Postfix parameter, which is `/var/mail` by default. 
However, please note that these mailboxes are not accessible through IMAP with the default setup.  
Instead of delivering to UNIX-style mailboxes, you should consider setting up aliases for local users and forward mail to a hosted domain.
  
Delivery from external sources to addresses of the form `user@[ip]` is disabled by default.  

### Dovecot
All configuration is located in the `/etc/dovecot/dovecot.conf` file. The split configuration files under `/etc/dovecot/conf.d/` are ignored.  
Passwd-files are used as a backend for storing user information, including hashed passwords.

### SPF
SPF checking is performed using **postfix-policyd-spf-python**

### OpenDKIM and DKIM
Generated keys are stored in the `/etc/dkimkeys` directory. Mappings are stored in the `/etc/opendkim` directory.

### SpamAssassin
DNS-based whitelist (DNSWL) queries are disabled by default.  
The custom configuration is located in the `/etc/spamassassin/local.cf` file. Feel free to customize SpamAssassin according to your needs.

### TLS certificates
By default, the installer generates a self-signed certificate to be used by Postfix and Dovecot. The resulting key and certificate files are placed into
`/etc/ssl/private/local/key.pem` and `/etc/ssl/certs/local/fullchain.pem` respectively. While this might be OK for testing purposes, it is highly recommended that you obtain and use a real certificate.

&nbsp;
# Postfix virtual vs Dovecot LMTP

All versions following 1.1.0, use Postfix's **local** and Dovecot's **LMTP** agent for final delivery by default.  
With version 1.1.0, by default, mail will be delivered through Postfix's **local** and **virtual** delivery agents. 
If you prefer or need to use Dovecot's **LMTP** agent for final delivery, the following extra steps are needed:

1. Install the dovecot-lmtpd package
```
apt install dovecot-lmtpd
```

2. Edit dovecot.conf and add 'lmtp' to the list of protocols
```
protocols = imap lmtp
```

3. Edit dovecot.conf and setup the lmtp service along with a unix socket which will be used by Postfix to pass mail to Dovecot for delivery.
```
service lmtp {
    unix_listener /var/spool/postfix/private/dovecot-lmtp {
        group = postfix
        mode = 0600
        user = postfix
    }
}
```

4. Edit Postfix's main.cf and change the virtual_transport parameter to:
```
virtual_transport = lmtp:unix:private/dovecot-lmtp
```
5. Restart Postfix and Dovecot
```
systemctl restart postfix
systemctl restart dovecot
```

Depending on your objectives, using Dovecot's LMTP server for final delivery might come with some advantages, 
such as the ability to use **Sieve** scripts or the ability to encrypt mail through the **mail-crypt** plugin.  

You can find more info about Dovecot's LMTP server at: https://doc.dovecot.org/configuration_manual/protocols/lmtp_server/#lmtp-server  

With the default configuration, switching between Postfix's `virtual` delivery agent and Dovecot's `LMTP` for final delivery can be done easily by changing the `virtual_transport` Postfix parameter.

```
virtual_transport = virtual
virtual_transport = lmtp:unix:private/dovecot-lmtp
```

&nbsp;
# Mail encryption

All versions following 1.1.0 use mail encryption by default through the **mail-crypt** Dovecot plugin.  
When adding new users using the **postnimda** management tools, encryption can be disabled on a *per-user* basis.
  
Each user's **userkey** is encrypted with the user's mailbox password. If you lose or forget your password your mails will be unrecoverable.
  
Note that while this type of setup might add an additional layer of security/privacy, it is not without limitations.
For example, if your mail server is hosted on a **VPS**, it might discourage a bored administrator from randomly peeking into your communications, 
but a sufficiently determined one could still decrypt your messages.
