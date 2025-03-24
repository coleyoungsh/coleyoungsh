+++
title = "Offline Email With Aerc"
date = "2025-03-23T21:53:40-04:00"

#
# description is optional
#

description = "Aerc mail client setup for offline email, indexing with notmuch,and contact sync"

tags = ["cli", "email", "tutorial"]
+++

Build an offline first email system the unix way

## Applications

- [aerc](https://aerc-mail.org/) - mail client
- [notmuch](https://notmuchmail.org/) - index + search mail
- [mbsync](https://isync.sourceforge.io/mbsync.html) - download mail
- [msmtp](https://github.com/marlam/msmtp) - send mail + send queue
- [goimapnotify](https://gitlab.com/shackra/goimapnotify) - mailbox watcher
- [khard](https://github.com/lucc/khard) - contact management
- [vdirsyncer](https://github.com/pimutils/vdirsyncer) - contact sync
- [xandikos](https://github.com/jelmer/xandikos) - contact server (optional)

# Storing Mail Offline

## mbsync

Sample configuration for mbsync using an external command to obtain the users
imap password using pass

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
faster, especially with large amounts of mail

## notmuch

```~/.notmuch-config
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

## Mailbox Sync at Login

goimapnotify is going to monitor for incoming mail once logged in, however, it
only triggers when new mail is recieved. If mail is recieved while a user was
logged out or a machine was off it needs to be synced manually. Once at login
will suffice.

```~/.config/systemd/user/mbsync.service
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

# Sending Mail Offline

aerc itself can send mail via smtp. To be able to reply to mail and queue it
for sending without a network connection we'll setup msmtp.

## msmtp

msmtp configuration example using pass for smtp password

```~/.config/msmtp/config
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
some aliases to interact with the queue

```~/.zshrc
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

Simple imapnotify configuration using pass for imap password

```yaml ~/.config/imapnotify/mail.yaml
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

```bash ~/.local/bin/mbsyncnotify
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

# aerc Setup

Here's the config that combines everything we just setup.

```~/.config/aerc/accounts.conf
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
pgp-opportunistic-encrypt=true
```

# Local Contacts

aerc doesn't handle contacts on it's own, we use khard (command line vcard
utility) for this.

## khard

I didn't change anything in the default config except adding my path to where I
wanted my contacts to be stored

```~/.config/khard/khard.conf
[addressbooks]
[[addressbook]]
path = ~/.local/share/contacts/addressbook/

[general]
debug = no
default_action = list
# These are either strings or comma seperated lists
editor = nvim, -i, NONE
merge_editor = vimdiff

[contact table]
# display names by first or last name: first_name / last_name / formatted_name
display = first_name
# group by address book: yes / no
group_by_addressbook = no
# reverse table ordering: yes / no
reverse = no
# append nicknames to name column: yes / no
show_nicknames = no
# show uid table column: yes / no
show_uids = yes
# show kind table column: yes / no
show_kinds = no
# sort by first or last name: first_name / last_name / formatted_name
sort = last_name
# localize dates: yes / no
localize_dates = yes
# set a comma separated list of preferred phone number types in descending priority
# or nothing for non-filtered alphabetical order
preferred_phone_number_type = pref, cell, home
# set a comma separated list of preferred email address types in descending priority
# or nothing for non-filtered alphabetical order
preferred_email_address_type = pref, work, personal

[vcard]
# extend contacts with your own private objects
# these objects are stored with a leading "X-" before the object name in the vcard files
# every object label may only contain letters, digits and the - character
# example:
#   private_objects = Jabber, Skype, Twitter
# default: ,  (the empty list)
# private_objects = Jabber, Skype, Twitter
# preferred vcard version: 3.0 / 4.0
# preferred_version = 3.0
# Look into source vcf files to speed up search queries: yes / no
search_in_source_files = no
# skip unparsable vcard files: yes / no
skip_unparsable = no
```

## vdirsyncer

After setting all that up I wanted a way to sync my contacts to my phone, for
this I setup a carddav server and sync to it using vdirsyncer. This also uses
pass to authenticate.

```~/.config/vdirsyncer/config
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

```bash
$ systemctl --user enable vdirsyncer.timer
```

