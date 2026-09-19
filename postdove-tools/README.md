# Pre-requisites
Postfix and Dovecot must already be installed and fully configured.  
Dovecot must be configured to use passwd-files for user authentication.

# Description
More or less, the management scripts perform the operations described below

&nbsp;  
### setup
- inspect the Postfix and Dovecot configuration parameters in order to determine the locations of the different data files
- create the 'postnimda.cf' configuratoin file used by the rest of the scripts
- attempt to create the *virtual domains* home directory on the local filesystem, if not already created.  

**Notes:**  
All virtual domains and their associated virtual user accounts will reside under the *virtual domains* home directory.  
The exact location of this directory is determined by Postfix's *virtual_mailbox_base* parameter.  
The `setup` script must be re-run every time the relevant file paths are changed in Postfix or Dovecot  

&nbsp;  
### alias-add
- add a new alias to Postfix - *virtual_alias_maps*

Example:
```
./alias-add alias@domain.tld
```

**Notes:**  
The domain used by the alias and by the target must already be in the system and must be the same.  
The target must be a real, already existing, mailbox address. It can't be another alias.  
Duplicate aliases are not allowed.

&nbsp;  
### domain-add
- create a new, randomly generated, local UNIX user and group to be used as the owner of this *virtual domain*'s home directory
- create the *virtual domain*'s home directory on the local filesystem
- add the virtual domain to Postfix - *virtual_mailbox_domains*
- add ownership information to Postfix - *virtual_uid_maps* and *virtual_gid_maps*
  
Example:
```
./domain-add domain.tld
```

&nbsp;  
### domain-del
- remove the virtual domain from Postfix - *virtual_mailbox_domains*
- remove the virtual users associated with this virtual domain from Postfix
- remove the virtual users associated with this virtual domain from Dovecot
- **remove the home directory of this virtual domain from the local filesystem, including all associated user mailboxes** !
- **remove the local UNIX user and group that owned the *virtual domain*'s home directory** !

Example:
```
./domain-del domain.tld
```

&nbsp;  
### user-add
- add the user to Postfix
- add the user to Dovecot
- create the user's home directory on the local filesystem; the owner is the owner of the virtual domain under which the user is added
- create the user's Maildir directory under the user's home directory
- generate the userkey for mailbox encryption

Example:
```
./user-add user@domain.tld
```

**Notes:**  
On new user creation, a userkey is generated even if encryption is not selected for the user.  
On such a case, when encryption was initially not selected, to start encrypting mails at a later time, 
you can manually change for the selected user the value of the **userdb_mail_crypt_save_version** field from **0** to **2** in the userdb file.  
However, keep in mind that with this approach only new mail will be encrypted. All mail received/sent till that point will remain unencrypted.

&nbsp;  
### user-del
- delete the user from Postfix
- delete the user from Dovecot
- **delete the user's home directory from the local filesystem, including all associated mail files** !

Example:
```
./user-del user@domain.tld
```

&nbsp;  
### user-set-pass
- set a new password for a specified user
- set a new password for the user's userkey

Example:
```
./user-set-pass user@domain.tld
```

&nbsp;  
### user-block
- add a user to the block list, denying access to the user's IMAP account - dovecot.deny

Example:
```
./user-block user@domain.tld
```

&nbsp;  
### user-unblock
- remove a user from the block list, restoring access to the user's IMAP account - dovecot.deny

Example:
```
./user-unblock user@domain.tld
```
