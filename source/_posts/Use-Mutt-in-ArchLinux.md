---
title: Use Mutt in ArchLinux
tags:
  - 技术
categories:
  - 工具
date: 2024-05-12 23:00:17
---


I successfully configured the Mutt email client a long time ago. Many open-source communities continue to use mailing lists as their primary communication tool, making a plain text email client essential. Mutt is an excellent choice for this purpose. I wrote this blog post to deepen my understanding of Mutt's processes and to share why it remains relevant and useful in today's digital communication landscape.

The following contents are cited from Mutt ArchWiki:

> Mutt focuses primarily on being a Mail User Agent (MUA), and was originally written to view mail. Later implementations (added for retrieval, sending, and filtering mail) are simplistic compared to other mail applications and, as such, users may wish to use external applications to extend Mutt's capabilities.
>
> Nevertheless, the Arch Linux mutt package is compiled with IMAP, POP3 and SMTP support, removing the necessity for external applications.

Although the ArchLinux mutt package includes SMTP protocol functionality, which could simplify email management, I have decided to stick with the traditional way of using Mutt. This approach allows me to fully understand and control the email handling process, ensuring that I can customize and optimize it according to my specific needs.

I decide to use the following softwares in Arch Linux (It should work in any UNIX-like environment).

+ [fetchmail](https://www.fetchmail.info/). It is a mail-retrieval and forwarding utility; it fetches mail from remote mail servers and forwards it to your local machine's delivery system.
+ [procmail](https://github.com/BuGlessRB/procmail). A program for filtering, sorting and storing email.
+ [Mutt](https://github.com/muttmua/mutt). The MUA.
+ [msmtp](https://marlam.de/msmtp/). A simple SMTP client.

## The Usage of fetchmail

It's not hard to use `fetchmail`. You should carefully read its [manual](https://www.fetchmail.info/fetchmail-man.html). It will explain everything you need.

```config
#  QQMail (FoxMail)
poll imap.qq.com protocol IMAP user "392499367@qq.com" password "" ssl keep
mimedecode
mda "/usr/bin/procmail -f %F -d %T"

# GMail
poll imap.gmail.com protocol IMAP user "shejialuo@gmail.com" password "" ssl keep
mimedecode
mda "/usr/bin/procmail -f %F -d %T"
```

The `fetchmail` will eventually call `procmail` for directing the emails.

## The Usage of procmail

The functionality of `procmail` is very powerful. However, power sometimes can cause a lot of trouble. So, I only use `procmail` to put the received email into the corresponding inbox.

```config
MAILDIR=$HOME
VERBOSE=off
LOGFILE=/tmp/procmaillog

:0
* ^TO392499367@qq\.com
Mail/QQMail/inbox/

:0
* ^TOshejialuo@foxmail\.com
Mail/QQMail/inbox/

:0
* ^(To|Cc):.*git@vger\.kernel\.org
* ^TOshejialuo@gmail\.com
Mail/GMail/mailing_list/git/

:0
* ^From:.*@google\.com
* ^TOshejialuo@gmail\.com
Mail/GMail/information/google/

:0
* ^From:.*@apple\.com
* ^TOshejialuo@gmail\.com
Mail/GMail/information/apple/

:0
* ^From:.*@github\.com
* ^TOshejialuo@gmail\.com
Mail/GMail/information/github/

:0
* ^From:.*@wakatime\.com
* ^TOshejialuo@gmail\.com
Mail/GMail/information/wakatime/

:0
* ^From:.*@wakatime\.com
* ^TOshejialuo@gmail\.com
Mail/GMail/information/steam/

:0
* ^TO*shejialuo@gmail.com*
Mail/GMail/inbox/

:0：
Mail/spam
```

The configuration described above is straightforward. It efficiently examines both the To and Cc fields of an email and directs the message to the appropriate folder accordingly.

## The Usage of mstmp

It's still not so hard to configure the `mstmp`.

```config
# QQMail (FoxMail)
account QQMail
    auth login
    host smtp.qq.com
    port 587
    from 392499367@qq.com
    user 392499367@qq.com
    password ""
    tls on
    tls_certcheck off

# GMail
account GMail
    auth login
    host smtp.gmail.com
    port 587
    from shejialuo@gmail.com
    user shejialuo@gmail.com
    password ""
    tls on
    tls_certcheck on

logfile /tmp/msmtp.log
```

## The Usage of Mutt

### Theme

I use the [dracula theme](https://github.com/dracula/mutt).

```config
# Dracular color theme
source ~/.config/mutt/dracula.muttrc
```

### Basic Configuration

The configuration could be very different for different people. I simply put my configuration below. You'd better use the `man muttrc` to change the configuration for whatever you like.

```config
# Use directory by default instead of a single file
set mbox_type=Maildir

# The default directory to save attach_save_dirachments from the "attachment" menu.
set attach_save_dir = ~/.config/mutt/attachments

# When this variable is set, mutt will beep when an error occurs.
set beep = no

# Character set your terminal uses to display and enter textual data
set charset = "utf-8"

# This option allows you to edit the header of your outgoing messages along with
# the body of your message.
set edit_headers=yes

# This variable specifies which editor is used by mutt.
set editor="vim"

# Controls whether or not a copy of the messages you are replying to is
# included in your reply.
set include = yes

# When set, mutt will set the envelope sender of the message.
set use_envelope_from = yes

# This variable specifies what real name should be used when
# sending messages.
set realname="shejialuo"

# Specifies how to sort messages in the "index" menu
set sort = threads

# Controls whether or not Mutt will move read messages from your spool mailbox to your
# $mbox mailbox
set move = yes
```

### Use w3m To View HTML Content

I don't want to use external browser to view HTML content. I use [Reading html email with mutt](https://fiasko.io/projects/htmail-view.html.en) as the reference to setup. It's not hard.

We need to first change the `mutt` configuration like the following:

```config
# Set w3m to view html content
auto_view text/html
alternative_order text/plain text/enriched text/html
```

And the add the following content to the `~/.mailcap`:

```config
text/html; w3m -I %{charset} -T text/html; copiousoutput;
```

### Use Zathura to View PDF File

Append the following content to the `~/.mailcap`:

```config
application/pdf; zathura %s;
```

### Vim Bindings

Please refer to this wonderful blog [Mutt, the Vim Way](https://ryanlue.com/posts/2017-05-21-mutt-the-vim-way).

### Multiple Accounts Support

We could use Mutt hook functionality to support multiple accounts. For multiple accounts, you must use *account-hooks* and *folder-hooks*.

+ *Folder-hooks* will run a command before switching folders. This is mostly useful to set the appropriate SMTP parameters when you are in a specific folder. For instance when you are in your work mailbox and you send an e-mail, it will automatically use your work account as sender.
+ *Account-hooks* will run a command every time Mutt calls a function related to an account, like IMAP syncing. It does not require you to switch to any folder.

I use folder-hooks because there is no account configuration for mutt. I use Gmail as the master configuration in `muttrc` like the following:

```config
# GMail as the master
source ~/.config/mutt/accounts/GMail

# Multiple Accounts support
folder-hook ~/Mail/GMail/* source ~/.config/mutt/accounts/GMail
folder-hook ~/Mail/QQMail/* source ~/.config/mutt/accounts/QQMail
```

Below is a full example for GMail and QQMail.

```config
# ~/.config/mutt/accounts/GMail

set from = "shejialuo@gmail.com"
set sendmail = "/usr/bin/msmtp -a GMail"
set spoolfile="$HOME/Mail/GMail/inbox"
set mbox="$HOME/Mail/GMail/seen"
set record="$HOME/Mail/GMail/sent"
set postponed="$HOME/Mail/GMail/draft"

# vim: ft=muttrc:

---

# ~/.config/mutt/accounts/QQMail
set from = "392499367@qq.com"
set sendmail = "/usr/bin/msmtp -a QQMail"
set spoolfile = "$HOME/Mail/QQMail/inbox"
set mbox="$HOME/Mail/QQMail/seen"
set record="$HOME/Mail/QQMail/sent"
set postponed="$HOME/Mail/QQMail/draft"

# vim: ft=muttrc:
```

## Create Mail Directories

```sh
# spam
mkdir -p $HOME/Mail/spam/{cur,new,tmp}

# QQMail
mkdir -p $HOME/Mail/QQMail/inbox/{cur,new,tmp}
mkdir -p $HOME/Mail/QQMail/draft/{cur,new,tmp}
mkdir -p $HOME/Mail/QQMail/seen/{cur,new,tmp}
mkdir -p $HOME/Mail/QQMail/sent/{cur,new,tmp}

# GMail
mkdir -p $HOME/Mail/GMail/inbox/{cur,new,tmp}
mkdir -p $HOME/Mail/GMail/draft/{cur,new,tmp}
mkdir -p $HOME/Mail/GMail/seen/{cur,new,tmp}
mkdir -p $HOME/Mail/GMail/sent/{cur,new,tmp}
mkdir -p $HOME/Mail/GMail/information
mkdir -p $HOME/Mail/GMail/mailing_list
```

## Common Operations

### Read Raw Mailbox

```sh
muut -f /tmp/a
```

### Send the Patches

```sh
for p in *.patch; do proxychains4 mutt -H "$p"; done
```

## Reference

+ [Mutt ArchWiki](https://wiki.archlinux.org/title/mutt)
+ [Fetchmail Manual](https://www.fetchmail.info/fetchmail-man.html)
+ [Reading html email with mutt](https://fiasko.io/projects/htmail-view.html.en)
