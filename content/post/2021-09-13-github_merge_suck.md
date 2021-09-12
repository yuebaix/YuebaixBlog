---
title:       "Github Merge Suck"
subtitle:    "Github的合并很糟糕"
description: ""
author:      "月白"
date:        2021-09-13T03:11:59+08:00
URL:         ""
image:       ""
categories:  ["TECH"]
tags:        ["传奇"]
---

[邮件原文链接](https://lore.kernel.org/lkml/CAHk-=wjbtip559HcMG9VQLGPmkurh5Kc50y5BceL8Q8=aL0H3Q@mail.gmail.com/)

## 邮件正文

```
From: Linus Torvalds <torvalds@linux-foundation.org>
To: Konstantin Komarov <almaz.alexandrovich@paragon-software.com>
Cc: ntfs3@lists.linux.dev,
	linux-fsdevel <linux-fsdevel@vger.kernel.org>,
	Linux Kernel Mailing List <linux-kernel@vger.kernel.org>
Subject: Re: [GIT PULL] ntfs3: new NTFS driver for 5.15
Date: Sat, 4 Sep 2021 11:08:58 -0700	[thread overview]
Message-ID: <CAHk-=wjbtip559HcMG9VQLGPmkurh5Kc50y5BceL8Q8=aL0H3Q@mail.gmail.com> (raw)
In-Reply-To: <CAHk-=whFAkqwGSNXqeN4KfNwXeCzp9-uoy69_mLExEydTajvGw@mail.gmail.com>

On Sat, Sep 4, 2021 at 10:34 AM Linus Torvalds
<torvalds@linux-foundation.org> wrote:
>
> For github accounts (or really, anything but kernel.org where I can
> just trust the account management), I really want the pull request to
> be a signed tag, not just a plain branch.

Ok, to expedite this all and not cause any further pointless churn and
jumping through hoops, I'll let it slide this time, but I'll ask that
you sort out your pgp key for the future and use signed tags.

Also, I notice that you have a github merge commit in there.

That's another of those things that I *really* don't want to see -
github creates absolutely useless garbage merges, and you should never
ever use the github interfaces to merge anything.

This is the complete commit message of that merge:

    Merge branch 'torvalds:master' into master

Yeah, that's not an acceptable message. Not to mention that it has a
bogus "github.com" committer etc.

github is a perfectly fine hosting site, and it does a number of other
things well too, but merges is not one of those things.

Linux kernel merges need to be done *properly*. That means proper
commit messages with information about what is being merged and *why*
you merge something. But it also means proper authorship and committer
information etc. All of which github entirely screws up.

We had this same issue with the ksmbd pull request, and my response is
the same: the initial pull often has a few oddities and I'll accept
them now, but for continued development you need to do things
properly. That means doing merges from the command line, not using the
entirely broken github web interface.

(Sadly, it looks like that ksmbd discussion was not on any mailing
lists, so I can't link to it).

                    Linus
```
