---
title: Upgrade Your Packages!
date: 2021-03-05 07:30:05
tags:
  - security
  - war stories
---

It's Saturday night, March 7th 2015. When checking my old photos this selfie is the only one I found from the night - obviously I'm having a blast with my friends in Uppsala, Sweden. I'm 19 at the time so partying was not unusual, although I don't really remember going this hard.

<img src="situation-fun-night.png" width="350" />

Anyways, a colleague from the web agency I work for calls me during the night - we didn't really hang out during spare time so I was surprised. I pick up, and he tells me to go somewhere quiet. Also he tells me that one of our clients' websites, more specifically an airport, has been hacked - by Islamic State. 2015 was kind of the peak of the European migration crisis, with Sweden being heavily affected. With IS being part of the cause they were at the top of many people's minds. I think we both realised quickly this was quite bad.

<img src="situation-hacked.png" width="500" />

The website was built on top of WordPress (so obviously running with PHP), and co-located on a server running Apache with many other of our clients, using chroot and vhosts for isolation (backed by Plesk). We decided to shut the website as quickly as possible, checked some of the other clients' websites to know they weren't targeted, and decided to continue later when we were both in a better state.

On Monday morning we found out that this was not a targeted attack but rather a [vulnerability in the WordPress plugin FancyBox](https://blog.sucuri.net/2015/02/zero-day-in-the-fancybox-for-wordpress-plugin.html), where the attackers were able to automatically scan possible vulnerable targets and perform XSS on them. As this vulnerability had already been disclosed and [patched a month before](https://plugins.trac.wordpress.org/timeline?from=2015-02-04T22%3A13%3A27Z&precision=second), a simple upgrade of the packages would've prevented this from happening.

As you understand, the consequences for an airport getting hacked by IS (even though it was not a targeted attack), are quite severe. Several newspapers wrote about this incident, security had to be elevated for a while etc. All of it avoidable by upgrading the packages.

This was some years before tools like [Dependabot](https://dependabot.com/) and [Snyk](https://snyk.io/) started popping up, but nowadays there's just absolutely no excuses for leaving vulnerable packages. Especially not with GitHub prompting you every git push in case Dependabot has discovered a vulnerability. Despite all this tooling, [Using Components with Known Vulnerabilities](https://owasp.org/www-project-top-ten/2017/A9_2017-Using_Components_with_Known_Vulnerabilities) is still part of the OWASP top-10.

It's super easy to prevent this - just upgrade your packages. Let me repeat - UPGRADE YOUR PACKAGES!

Cheers