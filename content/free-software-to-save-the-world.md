---
title: Free software to save the world
#date:
#description:
tags: [ 'free', 'software', 'foss', 'ethics' ]
draft: true
---

During the past few years, I've been focused on free software a lot. At first, it
seemed to me like a weird thing for hippies and hipsters (which it still is for
most people, let's not deny it).

A couple of years later (which is around now), I've became quite involved in free
software communities. I have a few diverse contributions to my counter, and I'm
currently working at [CozyCloud](https://cozy.io), after a quick (but intense)
internship at [Matrix.org](https://matrix.org) (as you might have guessed, both
work on free software projects). And this world doesn't cease to amaze me.

**Disclaimer**: In this post, I'll share my opinion and experience on free
software. I'm not stating it as an absolute truth, and you're absolutely free
to disagree with it. I'd even be glad to discuss it if that's the case, either
on [Twitter](https://twitter.com/BrenAbolivier) or by email at
<blog@brendanabolivier.com>.

Now I guess some readers don't know what free software is, or might not understand
some expressions I'll be using in this post (plus I'm really stubborn in my way
to use them, ask my flatmate). So here's a quick recap. Please note that I'll be
talking about free **software** in this post, but most of my points also applies
to resources (images, videos, documentation, etc.) published in the same terms as
free software.

## Terminology

* **Free software**: The "free" in "free software" is the same one as in "freedom".
Free software is software distributed under a *free license*, which is a license
allowing the software's user to run, copy, distribute, study, change and improve
it.
* **Open-source software**: There's a lot of discussion on the meaning of this
expression. For some it's the same as free software, for others it's not. I call "open-source software" all software that isn't distributed a free license but
allows the public access to its source code. Also called "OSS".
* **FOSS**: An acronym meaning "Free and Open-Source Software".
* **Proprietary software**: Usually refers to software that isn't free.
<!-- Maybe add some stuff here as writing the post goes -->

## A short history of free software

There was a time, at the dawn of programming, where programmers and hackers,
researchers and curious people, were all living and working in harmony (kind of).
Everyone was discovering the powers of a computer and sharing their discoveries
and source codes with the others.

In the early 80s, however, this [hacker
culture](https://en.wikipedia.org/wiki/Hacker_culture) was in decline, as
programmers and manufacturers progressively stopped distributing the source code of
their programs and started using copyright and restrictive software licenses.

Meanwhile, in a MIT lab, a grumpy hippie named
[Richard Matthew Stallman](https://en.wikipedia.org/wiki/Richard_Stallman), still
found of [hacker ethic](https://en.wikipedia.org/wiki/Hacker_ethic), struggles
with the lab's printer. It has paper jam issues, and lacks some cool features
Stallman hacked into the previous one. So he emails the printer's manufacturer,
Xerox, asking for the source code so he could add his changes to it, which Xerox
denied.

This made Stallman realise the hacker culture was disappearing, and made him
realise he had to take actions before it was too late.

In 1983, Stallman creates the [GNU](https://www.gnu.org/) project which aims at
replacing the (mostly) proprietary [Unix](https://en.wikipedia.org/wiki/Unix).
Shortly after that, he even quits from the MIT to work full time on it. A couple
of years later, he creates the [Free Software Foundation](https://www.fsf.org/)
with the mission to create a legal structure for free software.

These two projects will serve as the base of what free software is today, by
providing the [GNU licenses](https://www.gnu.org/licenses/), which are a set of
free licenses, and by creating the GNU/Linux operating system (which is often
[abbreviated](https://www.gnu.org/gnu/gnu-linux-faq.html#why) as only "Linux"),
built on top of the Linux kernel, and which is currently the most used operating
system in the world.

Back to the present, free software are widely used all around the world, both by
individuals and big corporations. For instance, I'm currently writing this post
using [Atom](https://github.com/atom/atom), while listening to some music in
[Rhythmbox](https://en.wikipedia.org/wiki/Rhythmbox) or
[VLC](https://www.videolan.org/vlc/) and browsing the Web using
[Firefox](https://www.mozilla.org/en-US/firefox/) and chatting with friends over
[Riot](https://github.com/vector-im/riot-web). On the other side of the screen,
most websites I usually browse are using free software as their Web server,
operating system, sometimes even as their [content
manager](https://fr.wikipedia.org/wiki/MediaWiki). Even this blog is [powered by
free software](https://github.com/gohugoio/hugo).

## The culture of freedom

"But what exactly is free software", you might ask. As I mentioned above, a
software is free when it gives freedom to its user. More precisely, it refers
to four kind of freedom, as stated on [the FSF website](https://www.gnu.org/philosophy/free-sw.html.en):

* The freedom to run the program as you wish, for any purpose (freedom 0).
* The freedom to study how the program works, and change it so it does your
computing as you wish (freedom 1). Access to the source code is a precondition
for this.
* The freedom to redistribute copies so you can help your neighbor (freedom 2).
* The freedom to distribute copies of your modified versions to others (freedom 3).
By doing this you can give the whole community a chance to benefit from your changes.
Access to the source code is a precondition for this.

On top of framing a legal setting for free software by being enforced by the
software's license, these four freedoms also set the ethical dimension of the
free software culture. Really close to the hacker culture, it promotes both
transparency and respect of the software's user.

And that's where free software differs from proprietary software: instead of
forcing the user to only be a passive party to the software's life, it allows
them to take an active part in it. The user can now know exactly what the software
does and hack it, instead of enduring it as a closed and opaque box that only
partly fits their needs.

This ethical dimension is really important to free software communities. Most
even use them in their project's design and management. That's how you usually
end up using free software when looking for avoiding [mass
surveillance](https://github.com/EFForg/privacybadger) or
[censorship](https://github.com/NInfolab/website-mirror-by-proxy), or why discussion
around most of free software projects can be found on public mailing lists or
IRC/Matrix/XMPP/etc. channels.

This second point also creates a unique relationship between the developer of a
free software and its users. Instead of having to go through multiple layers of
support/management/communication, a user can get in touch directly with the
software's developer, which usually makes the software fit better with the people
using it.

Another benefit of such a relationship is the feedback you get from your work.
You're not getting congratulations from managers happy because you helped make
some money come in, but you're getting thanks from users because your hard work
allowed them access to service they didn't have before, or with better conditions.
And both when I'm working on free software as my paid job and when I'm doing it
on my free time, reading this kind of messages always warm my heart at a point
I can't describe:

![](/images/matrix-dendrite-feeback.png)

And while this culture of freedom, respect and transparency towards the user can
be a constraint to some projects, some others are built from it. Having these
obligations towards the product's end user is essential in projects orbiting
around privacy or security: users don't **have to** trust the developpers because
they are told to, but users **can** trust the developpers because they see exactly
what the software does. "We are not evil" is replaced with "We can't be evil".
Because if the developpers drift away from their promises, users will be able to
notice it and use something else instead, which would kill the project.

## The professionals of free software

One very common idea about free software is that it's a somehow unstable thing
developed by some hippies in a basement during their free time. But although this
might have been true at some point of history, things have changed a bit since
then.

As I mentioned earlier, free software is getting a bigger and bigger place on our
computers or servers. This also means that the allocated ressources to the
development of such software has also gone bigger and bigger, because the people
doing it usually want to turn it into a paid job, and because the companies using
it usually want to ensure the software will keep getting updates.

However, this goal can seem hard to achieve. How would you make money out of
something anyone can access and use for free?

Several kind of structures and scenarios of people turning free software
development into a paid job already exists. The most obvious case is using a
non-profit foundation structure, which will employ people to work on free software.
This is the case, for example, of the [Mozilla
Foundation](https://www.mozilla.org/en-US/foundation/), which develops
[Firefox](https://www.mozilla.org/en-US/firefox/) and
[Thunderbird](https://www.mozilla.org/en-US/thunderbird/), or the [Wikimedia
Foundation](https://wikimediafoundation.org/wiki/Home), developping the software
behind [Wikipedia](https://www.wikipedia.org/). These foundation usually live off
donations from users or corporations, and promote their software as a solution
to an ethical issue. To continue with the previous examples Firefox is introduced
as a solution to mass surveillance and respectful browsing on the Web, and
the Wikimedia Foundation works, (partly) by developping their software, towards
providing free and reliable knowledge to the world.
