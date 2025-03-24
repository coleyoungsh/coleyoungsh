+++
title = "Offline Email With Aerc"
date = "2025-03-23T21:53:40-04:00"

#
# description is optional
#

description = "Aerc mail client setup for offline email, indexing with notmuch,and contact sync"

tags = ["cli", "email", "tutorial"]
+++

Building an offline first email system the Unix way.

## Applications

- [aerc](https://aerc-mail.org/) - mail client
- [notmuch](https://notmuchmail.org/) - index + search mail
- [mbsync](https://isync.sourceforge.io/mbsync.html) - download mail
- [msmtp](https://github.com/marlam/msmtp) - send mail + send queue
- [goimapnotify](https://gitlab.com/shackra/goimapnotify) - mailbox watcher
- [pass](https://www.passwordstore.org/) - password manager
- [pam-gnupg](https://github.com/cruegge/pam-gnupg) - unlock gpg keys
- [khard](https://github.com/lucc/khard) - contact management
- [xandikos](https://github.com/jelmer/xandikos) - contact server (optional)
- [vdirsyncer](https://github.com/pimutils/vdirsyncer) - contact sync (optional)

# Secret Management

To avoid plain text secrets in config files. This guide uses pass to store all
the secrets these applications will need for the setup. Any password manager
that can be invoked via the command line will do so long as it prints the
password to standard out. I won't go over how to setup pass because that's out
of the scope of this guide. It's a well documented project.

# Storing Mail Offline

Most mail clients show you your mail by logging into the server and displaying
it from there. With mbsync we'll be able to download it from the server so that
we can view our inbox without an internet connection.

## mbsync

This is a sample configuration for mbsync calling an external command to obtain
the users imap password using pass.

`~/.mbsyncrc`

```
IMAPStore user@domain.com-remote
Host mail.domain.com
Port 993
User user@domain.com
PassCmd "/usr/bin/pass show domain.com/user"
TLSType IMAPS
CertificateFile /etc/ssl/certs/ca-certificates.crt

MaildirStore user@domain.com
Path ~/.local/share/mail/user@domain.com/
Inbox ~/.local/share/mail/user@domain.com/INBOX
Subfolders Verbatim

Channel user@domain.com
Far :user@domain.com-remote:
Near :user@domain.com:
Create Both
Expunge Both
Patterns *
SyncState *
```

before running mbsync for the first time, the MaildirStore path needs to be
created manually. mbsync will not create it.

To download mail for configured inbox

```
$ mbsync user@domain.com
```

To download mail for all configured inboxes

```
$ mbsync -Va
```

aerc can read from a maildir directly but setting up notmuch will make it
faster, especially with large amounts of mail.

## notmuch

Here is a sample notmuch config. Notice the setting `synchronize_flags=true`.
This will allow us to configure aerc to read from notmuch like maildir without
a query map.

`~/.notmuch-config`

```
[database]
path=.local/share/mail # path to where mbsync is downloading mail

[user]
name=User Name
primary_email=user@domain.com
other_email=user@domain2.com

[new]
tags=inbox;unread
ignore=.uidvalidity;.mbsyncstate;msmtpq.log

[maildir]
synchronize_flags=true
```

To index with notmuch

```
$ notmuch new
```

# Sending Mail Offline

aerc itself can send mail via smtp. However, it can only do so if you're
connected to your mail server. To be able to reply to mail and queue it for
sending without a network connection we'll setup msmtp.

## msmtp

Sample configuration for msmtp using pass for the user's smtp password.

`~/.config/msmtp/config`

```
# Set default values for all following accounts.
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.local/share/mail/msmtp.log

# user@domain.com
account user@domain.com
host mail.domain.com
port 465
tls_starttls   off
from User Name <user@domain.com>
user user@domain.com
passwordeval pass show domain.com/user
```

## msmtpq / msmtp-queue

To enable offline message queueing, use the script provided by msmtp "msmtpq".
I copied this to `/usr/local/bin` so it would be in my path.

```
$ sudo cp /usr/share/doc/msmtp/examples/msmtpq/msmtpq /usr/local/bin/
```

```
$ sudo cp /usr/share/doc/msmtp/examples/msmtpq/msmtp-queue /usr/local/bin/
```

Then change the default directories for logging and queue location. Also, setup
some aliases to interact with the queue more easily.

`~/.zshrc` or `~/.bashrc` or whatever shell you're using.

```
# msmtpq dirs
export MSMTPQ_Q="${XDG_DATA_HOME:-$HOME/.local/share}/mail/queue"
export MSMTPQ_LOG="${XDG_DATA_HOME:-$HOME/.local/share}/mail/queue/msmtpq.log"

alias mq='msmtp-queue' # view queue
alias mqf='msmtp-queue -r' # flush queue (send all)
```

# Realtime Mail Monitoring

Typically setups like this poll the server every x amount of minutes to check
for new mail. This is inefficient. We can use imapnotify to trigger actions
immediately.

## goimapnotify

Here is a sample imapnotify configuration using pass for the user's imap
password

`~/.config/imapnotify/mail.yaml`

```
configurations:
  - host: mail.domain.com
    port: 993
    tls: true
    tlsOptions:
      rejectUnauthorized: true
      starttls: false
    username: user@domain.com
    passwordCMD: "pass domain.com/user"
    xoAuth2: false
    boxes:
      - mailbox: INBOX
        onNewMail: "mbsync user@domain.com:INBOX && notmuch new"
        onNewMailPost: "mbsyncnotify user@domain.com"
```

This will also need a user service enabled to run at login, we'll enable it for our configuration. In this case `mail.yaml`

```
$ systemctl --user enable goimapnotify@mail.service
```

## mbsyncnotify

More than likely desktop notifications are desired. I use a script called
`mbsyncnotify` for this. It's called from `onNewMailPost` in our imapnotify
configuration. In my case:

`~/.local/bin/mbsyncnotify` (or somewhere in your path)

```
#!/usr/bin/bash

new=$(find\
		"$HOME/.local/share/mail/$1/"[Ii][Nn][Bb][Oo][Xx]/new/ \
		-type f 2> /dev/null)
	newcount=$(echo "$new" | sed '/^\s*$/d' | wc -l)
case 1 in
	$((newcount > 5)) )
  	echo "$newcount new mail for $1."
		notify-send "New Mail!" "󰶊  $newcount new mail(s) in \`$1\` account."
		;;
	$((newcount > 0)) )
		echo "$newcount new mail for $1."
		for file in $new; do
		  # Extract and decode subject and sender from mail.
			subject="$(sed -n "/^Subject:/ s|Subject: *|| p" "$file" |
				perl -CS -MEncode -ne 'print decode("MIME-Header", $_)')"
			from="$(sed -n "/^From:/ s|From: *|| p" "$file" |
				perl -CS -MEncode -ne 'print decode("MIME-Header", $_)')"
			from="${from% *}" ; from="${from%\"}" ; from="${from#\"}"
				notify-send "  $from:" "$subject"
		done
		;;
	*) echo "No new mail for $1." ;;
esac
```

## Mailbox Sync at Login

goimapnotify is going to monitor for incoming mail once logged in, however, it
only triggers when new mail is recieved. If mail is recieved while a user was
logged out or a machine was off it needs to be synced manually. Once at login
will suffice.

`~/.config/systemd/user/mbsync.service`

```
[Unit]
Description=Mailbox synchronization service

[Service]
Type=oneshot
ExecStart=/usr/bin/mbsync -Va && notmuch new

[Install]
WantedBy=default.target
```

enabling this user service will sync all mailboxes configured for mbsync and
indexed with notmuch

```
$ systemctl --user enable mbsync.service
```

# GPG key unlocking

I sign all of my emails and attach my public key to them. It can be a real
pain to keep unlocking your key every time you want to read an email someone
sent you with encryption. I utilize pam-gnupg to unlock my key when I log in to
my machine. This has the added benefit of having the key unlocked for pass so
all these components can function without you having to type in a password more
than once.

Here's a few commands that will set that up. Note that this assumes you're
logging in from a TTY on an Arch Linux system. Your configuration details may
need to be different from what I provide here if you're running something else.

```
$ echo "auth     optional  pam_gnupg.so store-only" | sudo tee -a /etc/pam.d/system-local-login
$ echo "session  optional  pam_gnupg.so" | sudo tee -a /etc/pam.d/system-local-login
$ echo "allow-preset-passphrase" >> ~/.gnupg/gpg-agent.conf
$ echo "default-cache-ttl 34560000" >> ~/.gnupg/gpg-agent.conf
$ echo "max-cache-ttl 34560000" >> ~/.gnupg/gpg-agent.conf
```

Don't blind copy and paste this last one, you'll need to fill in your email
that matches your key.

```
$ echo $(shell gpg -K --with-keygrip user@domain.com | tail -n2 | awk '{ print $$3 }') >> ~/.pam-gnupg
```

# aerc Setup

Here's the aerc accounts config that combines everything we just setup.

`~/.config/aerc/accounts.conf`

```
[user@domain.com]
source        = notmuch://~/.local/share/mail # use notmuch as backend
maildir-store = ~/.local/share/mail # maildir mapped in notmuch
maildir-account-path = user@domain.com # subdirectory of maildir for account
multi-file-strategy = act-dir
check-mail-cmd = mbsync user@domain.com && notmuch new
check-mail-timeout = 30s
outgoing      = msmtpq -a user@domain.com # calls msmtpq for sending
default       = INBOX
from          = User Name <user@domain.com>
copy-to       = Sent
cache-headers = true
pgp-auto-sign = true
pgp-attach-key = true
pgp-self-encrypt = true
pgp-opportunistic-encrypt=true
```

You can stop here if you don't care about email contacts.

# Local Contacts

aerc doesn't handle contacts on it's own, we'll use khard (command line vcard
utility) for this.

I also like to combine address book providers to use notmuch and khard. I wrote about that [here](https://man.sr.ht/~rjarry/aerc/integrations/combine-address-books.md) on the aerc wiki. If you'd like to do that it's up to you but from here I'll go over the configuration for khard.

## khard

I didn't change anything in the default config except adding my path to where I
wanted my contacts to be stored, it's right at the top.

`~/.config/khard/khard.conf`

```
[addressbooks]
[[addressbook]]
path = ~/.local/share/contacts/addressbook/
```

You'll need to tell aerc to use khard as your address book provider.

edit the `address-book-cmd` line in `~/.config/aerc/aerc.conf`

```
address-book-cmd=khard email -a addressbook --parsable --remove-first-line "%s"
```

You can stop here if you don't care about contact sync.

## vdirsyncer

After setting all that up I wanted a way to sync my contacts to my phone, for
this I setup a carddav server (xandikos) and sync to it using vdirsyncer. This also uses
pass to authenticate.

I won't go over to how to set up the server because there's a number of ways
you can deploy it. You could also use [syncthing](https://syncthing.net/) if
you wanted to for this and forgo vdirsyncer all together.

I will however provide a sample configuration for vdirsyncer in case you do
want to go that route.

`~/.config/vdirsyncer/config`

```
[general]
status_path = "~/.local/share/vdirsyncer/status/"

[pair my_contacts]
a = "my_contacts_local"
b = "my_contacts_remote"
collections = ["from a", "from b"]
conflict_resolution = "b wins"

[storage my_contacts_local]
type = "filesystem"
path = "~/.local/share/contacts"
fileext = ".vcf"

[storage my_contacts_remote]
type = "carddav"

url = "https://dav.domain.com"
username = "user"
password.fetch = ["command", "pass", "dav.domain.com/user"]
```

After configuring vdirsyncer a we run a few commands to sync initially

```
$ vdirsyncer discover
```

```
$ vdirsyncer sync
```

Then we can enable the systemd user timer to auto sync via `vdirsyncer.service`

```
$ systemctl --user enable vdirsyncer.timer
```

# The End

That's pretty much it. Email management following the Unix Philosophy. I'm sure
that there are improvements I can make but I'll cross those bridges when I get
there.
