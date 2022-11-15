# Migrating data from an old mailu installation

This document describes how to migrate from an old mailu installation to a new one. It is recommended to shut down front, postfix, dovecot, admin and roundcube while the migration is perfomred and start it on the new server after everything is migrated _and_ the DNS is switched to the new server.

## Migrate the mailu database

The easiest way is to copy the whole database. For this, stop the mailu pods and copy the content of the database from the old to the new server.
The particular process depends on the database type. For sqlite, you can copy the `admin/main.db` file. For mysql/postgresql use the dump/restore
tools provided with the database. Note that any manually made configuration will be lost on the new server. The server will also have the same DKIM keys that the old server had before.

As an alternative, you can manually re-create the accounts. This allows to migrate domain by domain and requires knowledge about your user's passwords. DKIM keys must be created for each domain and added to the DNS server.

## Migrate mailboxes

Mailboxes can easily synced via rsync. Ensure that the older and new mailserver (postfix/dovecot pods) are not running. For large mailboxes keep the old server running and only stop the new server. Then do an initial rsync. After that stop the old mailserver and do another rsync to copy the delta. Good rsync options are `rsync -avr --numeric-ids --delete-during`.

## Migrate the mailqueue

Depending on the load of your old mail server, the mail queue might still contain unsent emails. You can rsync it to the new server while postfix pod is not running.

## Migrate roundcube settings

You should definitely migrate the roundcube settings so that your users keep their address books and other settings. For sqlite, just copy `roundcube/roundcube.db` to the new server while the dovecot pod is not running.

If the pod name of the mailu-front server has changed, you must update the roundcube database, otherwise users will be re-created. On helm chart starting with version 1.0.0, the front pod is named "mailu-front.mailu.svc.cluster.local". On older chart it's just "mailu-front". On docker or other installations it might differ.

To update the sqlite database, perform the following steps while the dovecot pod is not running.

```
sqlite3 roundcube/roundcube.db
sqlite> update users set mail_host="mailu-front.mailu.svc.cluster.local";
```

## Migrate redis/rspamd cache

While you can copy the data (just copy the files), it is not required to run a new server.

## Upgrade DNS entries

If the new server has a new hostname, change the MX records of all your domains. If you keep the hostname, change the A record for the hostname to the new server. Ensure that all SPI/DKMI/DMARC DNS entries are still correct.

Other mailservers will see that your mailserver is offline while you are migrating and try to deliver emails later. After the DNS records are updated, the delivery ges to the new mailserver.

