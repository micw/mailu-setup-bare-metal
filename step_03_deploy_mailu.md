# Setup of Mailu on top of K3S

## Make some cofiguration decisions and deploy mailu

The mailu deployment options are well documented in the helm chart. Basic decisions that should be made in advance:

* What is the hostname of my mailserver? This is commonly something like `mail.mydomain.com`
    * Note that the hostname must resolve to the mailserver IP address to proceed
      (otherwise let's encrypt certificate generation will fail)
* For which domains do I want to receive emails? Let's start with `mydomain.com`. The domains `MX` DNS-Record must point to the
  mailserver IO address.
    * I'd recommend to start with an unused subdomain until the mail server runs properly and is whitelisted at common mail providers
* What kind of persistence should be used? I use the "one big volume" default setting with the local path provisioner provided by K3S
* What database should be used (sqlite, mysql, postgresql, internal or external managed)?  I use the default embedded sqlite which is
  fine for a small amount of mailboxes

After the decisions are made, the values can be set accordingly. See `kuploy/mailu.yaml` for my preferences. Now mailu can be deployed:

```
kuploy mailu.yaml
```

Mailu is starting up (may take a minute). Then you can log in via the mailserver hostname. Use the initially created admin credentials.


## Setup mailu

After deploying mailu, some initial steps should be done:

* Change the mailadmin password after first login
    * The initial password is stored in clear text in the kubernetes deployment. After the password is changed, the new password will only
      exist as salted hash in the database.
* Create the first email domain and a first account
* Create the postmaster mailbox. You can add a redirect to an external mailbox later. You can create the postmaster mailbox also as alias
  for a different mailbox


## Verify the correct detection of remote IP addresses

⚠️⚠️⚠️ This is very important to avoid to create an open relay ⚠️⚠️⚠️

* Send an email from an external account to your initial account
* Log into mailu and open "Antispam"
* In the history you can see the IP address where the email was received from
* This email address _must_ be a public IP address and it _must not_ be the public IP address of your mail server
    * you can verify this by doing `nslookup THE-IP-ADDRESS-FROM-LOG` which should reverse-resolve to your external email provider

## Warm-up and white-list your mailserver

If you start a new mailserver with a fresh IP address, it is very likely that large mail providers do not accept your emails. Here are some
hints how to improve the situation.

### Re-Verify your DNS settings

* Does the hostname of your mailserver resolves to the mailserver IP address
    * very likely it does, otherwise the test email from above would have failed
* Does the mailserver IP address resolves _exactly_ to the hostname of your mailserver?
    * It should resolve to something like `mail.mydomain.com`, not just to `mydomain.com`

### Add SPF/DKIM/DMARC for your domains

This is important because many mail providers nowadays will clasify non-DKIM emails as spam.

* Click on the first icon on the email domain in the mailu admin interface
* Click on "create keys" and confirm
* add the SPF/DKIM/DMARC DNS entries for your mail domain
    * for SPF use "-all" instead of "~all" to enforce it
* It makes sense to also add the auto discovery DNS entries
* Go to https://mxtoolbox.com/DMARC.aspx and enter your mail domain to check the DMARC record
* Go to https://www.mail-tester.com/spf-dkim-check and enter your mail domain to check the SPF and DKIM record (use dkim as selector)
* Open https://www.mail-tester.com/ and send an email to the generated email address shown at the page. Check the results.

### Check and clean dns blacklists

* go to https://www.dnsbl.info/ and enter your IP address
* cross thumbs that it's not blacklisted anywhere
* If it appears on one or more of the blacklists, click on the link to the blacklist and follor the instructions there to unlist your
  IP address

### Add your IP to whitelists

* The most common DNS whitelist is https://www.dnswl.org/
* Create a free account there
* Go to DNSWL IDS
* Add a new DNSWL Id
* Enter the domain name of your mailserver (`mail.mydomain.com` did not work for me but `mydomain.com` should do it)
* Follow the instructions by adding a special DNS entry or use an email address for verification
* After the domain is confirmed, add the IP address of the mailserver to the DNSWL Id
    * I had to repeat this twice because I got a lookup error
* On successful validation, a change request is created which will be processed within a few days

### Warm-up at several email providers

* Send mails to your own or friend and family addresses at several email providers
    * outlook.com / outlook.de / hotmail.com
    * gmail.com
    * others?
* Verify that the email is received and tagged as spam
    * if it is spam, add your email to the address book of the receiver and mark the email as non-spam

### White-List at hotmail.com (includes outlook.com and other microsoft email domains)

If you receive "Undelivered Mail Returned to Sender" from outlook.com with an error message like this, a manual white-listing must be
done at microsoft. Unfortunately that happens often for fresh mail server IP addresses and also happens from time to time for existing
mail servers.

```
host eur.olc.protection.outlook.com[104.47.4.33] said: 550
    5.7.1 Unfortunately, messages from [***.***.***.***]] weren't sent. Please
    contact your Internet service provider since part of their network is on
    our block list (S3150). You can also refer your provider to
    http://mail.live.com/mail/troubleshooting.aspx#errors.
```

* Open the hotmail blacklist removal form at https://support.microsoft.com/en-us/supportrequestform/8ad563e3-288e-2a61-8122-3ba03d6b8d75
* Fill out the required information
* It takes 24-48 hours to get your IP address white-listed

