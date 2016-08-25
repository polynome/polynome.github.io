---
layout: post
title:  "HackedThat: Breaking in to a hardened server via the back door"
date:   2016-08-16 15:39:10 -0700
categories: infosec inversoft elasticsearch linode penetration-testing
---

Earlier this summer, the team at
[Inversoft](https://www.inversoft.com/) published a comprehensive and
sophisticated
[guide to user data security](https://www.inversoft.com/guides/2016-guide-to-user-data-security). The
guide spans from hardening servers from provisioning, up through the
IP and SSH layers, and all the way to application-level techniques for
password hashing, SQL injection protection, and intrusion
detection. As proof that they stood behind their advice, the Inversoft
team provisioned a pair of [Linode](https://www.linode.com/) hosts, a
web server and database server, and gave them the hardening
treatment. Inversoft offered up a fully-loaded MacBook to anyone who
could break in, taunting all comers by naming the hardened web server
[hackthis.inversoft.com](https://hackthis.inversoft.com).


<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/game-on.gif">
</p>

Game on.

False Attempts
--------------

So one morning I started poking around. It only took a few minutes to
verify that the target servers were hardened just as described in the
whitepaper; SSH access via public keys only, no additional ports open
other than HTTP/HTTPS. Fingerprinting the web-facing host with
[nmap](https://nmap.org/) showed that it was running the latest Ubuntu
version, and revealed nothing about the HTTP server running (though I
knew from the whitepaper that it was a recent version of
[Express](https://expressjs.com/)).

Darkness washed over the Dude, as I realized that there would be no
easy way in. I didn't have high hopes for a
[SQL injection](https://www.owasp.org/index.php/SQL_Injection) attack,
so instead I spent some time trying various
[XSS payloads](https://github.com/zsitro/XSS-test-dump/blob/master/xss.txt)
on their basic web app, all to no avail. I didn't spend too much time
on this vector; even if I were able to get an XSS working, it would at
best allow me to read data from other users; a real security issue to
be sure, but insufficient to win the coveted MacBook.

<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/darkness-washed-over-the-dude.gif">
</p>

The user database under protection was Inversoft's
[Passport](https://www.inversoft.com/products/user-database-sso)
product, so as my next approach I decided to read a little bit more
about it, particularly its
[API docs](http://docs.inversoft.com/display/P10/Introduction). What's
that, hiding behind that last link? The API docs (as well as the
documentation for Inversoft's other products) live in a Confluence
wiki. If Inversoft uses Confluence so heavily for customer-facing
docs, I wondered, maybe they keep important internal information there
as well?

New Avenues
-----------

I spent a few minutes gathering usernames from the public-facing
Confluence pages and guessing at passwords to no avail. Knowing that
Confluence is complicated self-hosted software that often goes
un-updated, I looked at the version in a CVE database. There's
something!
[CVE-2015-8399](https://jira.atlassian.com/browse/CONF-39704) allows
unauthenticated users to browse and read files from disk that are
accessible to the Confluence user, and docs.inversoft.com was
vulnerable to it. I spent quite some time looking at basically every
file accessible to Confluence, but the team had managed to keep any
secrets out of the various Confluence configuration files. I was
hoping to snag database credentials or an administrator password, but
neither were to be found.

Speaking of database credentials, where was Confluence storing its
wiki data? I went back to nmap and took a look at
docs.inversoft.com. This was **much** more interesting; several HTTP
servers (mostly Java / Tomcat), Postgres and MySQL databases,
Elasticsearch, and some unknown services, as well as SSH, were listening
for connections on this host. I determined with a little more digging
that this machine was also the host for www.inversoft.com, and a
number of internal and external services. I had a lot more to explore
now.

I started trying to fingerprint and otherwise gather version
information for the services on this host. The server was running an
old version of Ubuntu (12.04), but it seemed to be fully-patched (I
verified later that it was). The Elasticsearch version running on the
server, however, was old enough to be vulnerable to
[CVE-2015-1427](http://jordan-wright.com/blog/2015/03/08/elasticsearch-rce-vulnerability-cve-2015-1427/). Go
take a look at that link, as it contains any security researcher's
three favorite letters:

R. C. E.

<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/jackpot.gif">
</p>

Elasticsearch, it seems, allows API users to specify custom scoring
functions that can be used to rank results. Those scoring functions
can be written as Groovy code. Elasticsearch implements a sandbox that
attempts to prevent any malicious code, but the sandboxing has
multiple flaws, and it's relatively trivial to send a "scoring
function" that in fact calls Runtime.getRuntime().exec() with
arbitrary shell commands. Given that the Elasticsearch port was open
to the world, and no authentication was required to run one of these
custom-scored searches, I had all the ingredients I needed to run
shell commands. Actually putting together a working PoC and a
[working reverse shell](https://highon.coffee/blog/reverse-shell-cheat-sheet/)
(connecting out to a new EC2 box I provisioned) took some grunt work,
but I ultimately succeeded and found myself staring at a command
prompt.

whoami
------

What could I do with my new shell? First, I determined that I was not
running as root, but instead as an application user, conveniently
named `inversoft`. This user could of course read anything that
Elasticsearch could, but it turned out there was no useful information
in the Elasticsearch cache. I turned my attention back to Confluence;
conveniently, it was also running under the `inversoft` account. With
this information, it didn't take long to find the database credentials
used by Confluence to talk to Postgres, and start pulling a full
database dump of the Confluence DB for offline perusal. There was a
massive amount of data, and I called in my colleagues to help me sift
through and look for anything useful.

Of course, I also had to think about detection here. I wasn't yet to
my goal, nor did I have a clear path to it, but I was disrupting two
services (Elasticsearch and Postgres) with my activity, and probably
starting to leave fingerprints. I determined that, as far as I could
tell, I wasn't harming any production resources or over-taxing their
servers with my exploration, and I continued to proceed with caution.

The Confluence database seemed, for a minute, to be the promised
land. It was used for internal documentation; not just docs, but also
various shared secrets, passwords, and other keys! Best of all, I
found a username and password for a Linode account! I knew from the
original whitepaper that the "HackThis" machines had been provisioned
with Linode, so I signed in and prepared to claim my reward.

It was the wrong Linode account. A real one, with real servers, but
not the servers that would win me my prize.

<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/oh-my-god-i-was-wrong.gif">
</p>

A Plan Emerges
--------------

Could there be multiple linked Linode accounts? Might the other
account use the same password, or a variant? No, and no. I was able to
recover the username of the Linode account I was targeting from the
original whitepaper's screen shots, so I tried several other passwords
found in the wiki, and none worked.

I hadn't yet succeeded, but I could see the way forward. Gain access
to the Linode account, and use the web-based console to get root on
the target servers. I just needed a password.

How would I get it? I spent some time looking at the postfix server
also running on the machine I had reached; could I intercept a
password reset email from Linode? Nope, the postfix server wasn't in
use; the team used Google Apps for email. Could I fashion a convincing
phishing attack using my privileged position? I couldn't think of a
clever way to do it, and I knew the Inversoft team would be on high
alert given the challenge they had issued. I spent some time trying to
elevate my privileges to root at the shell (for no good reason) but
found that the team had religiously applied Ubuntu's LTS patches and
none of the Linux elevation tricks I could find would work on their
kernel.

I continued to browse around the filesystem, searching for any lead. I
finally found the way forward in the first place I should have looked;
the `inversoft` user's home directory. It turns out that not only was
this account used to run several services on the box, but it was also
used by humans as a shared account for various projects and
one-offs. And one of those projects was provisioning the HackThis
machines. There, in `~inversoft/.linodecli/config`? A Linode API key.

<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/happy.gif">
</p>

A call to the "list hosts" API in Linode revealed the exact two hosts
I had in my sights, and confirmed that now I had the correct key. Time
to get a root console, right? Nope. You can do all sorts of things
with the [Linode API](https://www.linode.com/api), but getting a
console is not one of them; you can only do that on the web with a
regular username and password, and I still didn't have one of
those. The next thought was to try and export the disk image from one
of the Linode machines, but the API does not provide an "export"
function.

Smash and Grab
--------------

After lots of messing around with APIs, my colleague Anton had an
idea; what if we spin up a new, "intruder" machine in the Linode
account with a root password that we know and then connect the target
application server's disk to our intruder server instead? Looking
through the API docs, this seemed like it would be a working plan, but
it would also be obvious and destructive; our intruder machine would
appear on the Linode dashboard, and as soon as we unmounted the
volume, the machines we were targeting would start failing.

To pull this off, we'd have to grab the MySQL credentials and
exfiltrate the DB as quickly as possible, before the Inversoft
operators could detect us and shut us down. Since we were so close to
the prize, and since the machines weren't "real" production machines,
rather honeypots designed to be hacked, we decided this was a
reasonable and ethical course of action. At this point it was about
8pm local time at the Inversoft offices in Denver, so we hoped nobody
would be at their desks, potentially buying us a few extra minutes.

<p align="center">
<img width="75%" src="{{ site.baseurl }}/assets/hack-that-inversoft/mission-impossible.gif">
</p>

With me at the shell of the intruder host, Anton used the API to
attach the application server disk image to my machine, from which I
quickly retrieved the Passport API keys and the MySQL credentials to
the other host. Anton started enumerating all the data he could from
Passport using our API key while I triggered a `mysqldump` to geth the
DB - but I couldn't connect! We should have seen this coming; the
MySQL server had a firewall rule that permitted access *only* from the
application server; this was one of the hardening measures from the
whitepaper.

For most attackers, this bit of defense-in-depth would have been a
dead-end, but thinking quickly and using our superpowers (Linode API
keys), we performed a private IP swap between the application server
and our intruder server. Rebooted the intruder server to get the new
IP and it was all over: `mysqldump` connected, the data was SCPed off
to my machine, and we had beaten the challenge. Our haste was
warranted; once we reported the attack to Inversoft, they let us know
that they had received notification emails for every Linode action we
had taken (create server, connect disk, swap IP, etc), and had already
started investigating just as we had finished downloading the
database dump.

The Recap
---------

After discovering an unpatched, unfirewalled Elasticsearch instance
using `nmap`, we gained shell access on a utility server used for
various functions at Inversoft. On there, we found API keys for Linode
left behind by a human operator. Those keys allowed us to detach disks
from running servers and attach them to servers we controlled,
stealing sensitive user data (all to win a prize).

What could Inversoft have done differently to prevent this? Their
hardening guide was and remains correct; there was no way we were
getting through the front door of their servers (SSH or HTTPS). The
course we took was a common one in targeted attacks; gain access to
secrets used by humans, sometimes in ancillary systems, and use that
access to bypass security via operator consoles or other magic. The
most frequently seen version of this in the wild is to steal access to
an email inbox that can be used to reset a password, and although this
attack was slightly different, it's a great reminder that attackers
are far more likely to go around your defenses than through them.

The other weakness was the "jack-of-all-trades" Elasticsearch server
that we discovered and exploited. It's an example of a utility box 
that runs various random services - maybe acts as a bastion host or
testing ground - and nobody quite manages it or knows what it is used
for. This server is as weak as its weakest service; and because it is
not purpose-managed, it can be difficult to keep track of what is
running on it and ensure all services are patched and secured. If you
have one of these servers floating around somewhere, you might want to
think twice about keeping it - it may very well be the chink in your
armor.

Thanks to Inversoft for the effort they put in to writing their
security guide and sponsoring the "HackThis" challenge, and of course
the prize of a MacBook (which they quickly delivered). Their
[security guide](https://www.inversoft.com/guides/2016-guide-to-user-data-security)
remains an excellent resource, and we hope the practical lessons
learned from this post will help your organization identify less
obvious risks risks and secure your infrastructure.
