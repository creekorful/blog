+++
title = "Debian maintainer from zero to hero"
date = "2020-10-03"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Debian"]
keywords = ["", ""]
description = "What it feels like to become a Debian maintainer in 2020"
showFullContent = false
+++

I have been using [Debian](https://www.debian.org/) intensively for more than 4years, 
mainly on the servers I administrate. It's a really powerful & stable (one of the most) OS, with a *TONS* of package available
trough [APT](https://en.wikipedia.org/wiki/APT_(software)).

Debian has helped me in a lot of ways, and I wanted to bring something in return, even the little help I could offer.
So, one year ago I have emailed a friend who is also a [DD](https://wiki.debian.org/DebianDeveloper), to ask him if he's
willing to become my mentor to help me contribute to Debian. One year ago, 
I've started, with his help, the [new maintainer process](https://www.debian.org/doc/manuals/maint-guide/).

Please note that this isn't a technical guide at ALL, it's more my opinions & impressions on the Debian maintainer
experience.

# 1. What kind of contributions?

You can contribute to Debian in a lot of ways: from translating, submitting bug reports / patches (bugfixes), 
help newcomers on forums, help package & maintain software, etc ([full list of wanted contributions](https://www.debian.org/intro/help)). It only depends on
your skills, your free time & what you like to do.

I personally wanted to get involved into packaging (TL;DR: help get software into the Debian archive, 
so that end-users can install them with apt).

# 2. Packaging

## 2.1. Learn how to make a package from scratch

The first step is to learn how to make a package from scratch, so that you can understand exactly how it works,
and therefore be able to investigate future errors. I have used the [following documentation](https://wiki.debian.org/Packaging/Intro?action=show&redirect=IntroDebianPackaging).

I personally have spent 2 weeks reading the documentation, setting up a VM, and trial by errors. I have learned a lot of 
things that would help me a lot in the future.

For me, learning to make a package from scratch is a *really* important step, since it will be the core of your work
as a maintainer. Nowadays, a lot of the package process is automated, especially with the *dh-make* tools. Therefore,
a lot of stuff is going on under the hood, and not knowing how packaging work will make you struggle when encountering
errors / fixing packaging related bugs.

## 2.2. Learn what to package

Once you know how to package stuff, you have to choose what you're gonna do with this new skill.
Once again, several options are available:

### 2.2.1. Create a new package

This option may be the most interesting, especially if you already have a
software in mind, but you're gonna ask you a some questions first: 

- does this package will fit in the archive? (copyright, usefulness, etc...)
- Do I have enough skills to package this software?
- Do I have enough free time?
- Doesn't this package already exist?

In case of doubt, always ask somewhere for help. You can use [IRC](https://wiki.debian.org/IRC), [mailing list](https://www.debian.org/MailingLists/), etc. 

### 2.2.2. Help maintain existing package

There's a lot of package in [RFH](https://www.debian.org/devel/wnpp/rfh) (request for help). These packages already have
a maintainer, but seeking for help, meaning that he may be willing to help you update / fix bugs on the package, as
well as upload it on the archive for you.

RFH packages are really great since you'll help an existing maintainer, learn from him, be able to ask questions,
and the maintainer may be willing to advocate you to become a Debian maintainer in the future (see below for more details).

### 2.2.3. Take over abandoned package

Sometimes, maintainer may not have enough time for their packages, and will mark a package [RFA](https://www.debian.org/devel/wnpp/rfa) (request for adoption). 
These package are basically abandoned, looking for adoption. The maintainer may help you in taking over the maintenance.

# 3. Becoming a Sponsored maintainer

When your package is finished (successfully built + tested) you can't just upload it on the Debian archive. Only DD & [DM](https://wiki.debian.org/DebianMaintainer)
can do that.

You'll have to ask someone with upload rights to review it & upload it for you (this is called sponsoring).

To ease package reviewing you can use [mentors](https://mentors.debian.net/) and/or email the [debian-mentors](https://lists.debian.org/debian-mentors/) mailing list.

Once your first package is on [unstable](https://wiki.debian.org/DebianUnstable), you will be a [Sponsored maintainer](https://wiki.debian.org/SponsoredMaintainer)!

# 4. Becoming a Debian maintainer

Once you start mastering the art of packaging, you may candidate to become a Debian Maintainer (DM).

DM are people with a restricted upload rights on the archive. DD can grant them upload rights for specific packages,
so that they will be able to upload without sponsoring. 

To become a DM you must go through the [new-member process](https://nm.debian.org/), there's several steps required such
as getting your PGP uid signed by a DD (to enable trust), being advocated by a DD, agreeing to the [SC](https://www.debian.org/social_contract)/[DFSG](https://www.debian.org/social_contract#guidelines)/[DMUP](https://www.debian.org/devel/dmup).

After completing all steps, your application will stay pending for 4days+ (to allow any objections to be made) 
and finally the keyring maintainers will add your key to the Debian keyring, making you a Debian maintainer.

# 5. Conclusion

Contributing to such a big & distributed open source project taught me a lot of things, from technical skills such as packaging,
reviewing / submitting patches, arguing about them, reporting bugs, ... 
To social skills such as asynchronous communication (only email & IRC based) all
around the world, working with a lot of different mindset, culture, **being patient**, humble, etc... 

This experience helped me to be a better software developer & human being.

I'd like to thank the whole Debian community who is always really helpful & especially Alexandre Viau, my mentor,
who has greatly helped me & pushed me to become a Debian maintainer.

Happy hacking!
