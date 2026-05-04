---
title: The quest for equivalent Exchange
description: "Towards the end of the summer of 2023 my team at MZLA Technologies Corp, the Mozilla Foundation subsidiary that looks after Thunderbird, started discussing a fairly ambitious project: adding support for Microsoft Exchange's APIs. This would turn into an ambitious project that spanned over 2 years and resulted in the first new protocol to be natively supported by Thunderbird in over 20 years. In the first part of this duology, I'll go over the project's big picture, how we defined the project's direction, and the reasoning behind architectural decisions."
tags:
  - thunderbird
  - email
  - exchange
  - netscape
  - ews
  - graph
publishDate: 2026-05-04T00:00:00+00:00
draft: false
thumbnail: /exchange-pt-1/exchange-logo.png
---
Towards the end of the summer of 2023 my team at [MZLA Technologies Corp](https://blog.thunderbird.net/2020/01/thunderbirds-new-home/), the Mozilla Foundation subsidiary that looks after [Thunderbird](https://www.thunderbird.net), started discussing a fairly ambitious project: adding support for Microsoft Exchange's APIs. "Ambitious", because the last time native support for a new protocol/platform was added in Thunderbird was... a while ago. In fact, [Thunderbird 1.0](https://blog.thunderbird.net/2022/08/thunderbird-time-machine-windows-xp-thunderbird-1-0/), which came out towards the end of 2004, already came with most of the email and newsgroups support you know and love, which originated in [Netscape Mail](https://en.wikipedia.org/wiki/Netscape_Mail_%26_Newsgroups) 2.0 almost 10 years earlier.

I was lucky to be involved in the project's leadership to varying degrees, continuously up until the [public release](https://blog.thunderbird.net/2025/11/thunderbird-adds-native-microsoft-exchange-email-support/) of Exchange support for email last November, and I thought I'd reminisce about it here. This blog post overlaps to some extent with [a talk](https://fosdem.org/2026/schedule/event/FYVUB8-a_short_story_of_supporting_microsoft_exchange_in_thunderbird/) I gave at FOSDEM 2026 in February, though it's not meant as a textual transcript of it and will go a bit more in depth on some points. It's also much longer, as you can see, because I have ADHD and if there's one thing people with ADHD like to do is talk about anything at length. 

In fact, partway through writing this post I decided this blog has enough articles that require a full time job to read, so I decided to split it in two. This one, which you're currently reading, will go over the big picture and explain how we defined the project's direction, and the reasoning behind architectural decisions. The second and final part of this duology should come shortly after this one, and will cover the more practical and technical aspects of this project.

# What is Exchange?

Let's start with the basics: what the hell is this "Microsoft Exchange" thing?

Exchange is the name of the back-end service that supports some of Microsoft's productivity services, such as email, calendar management, and a few others. Essentially, if you've used Outlook with such services in a Microsoft-heavy environments (like some workplaces or school/universities can be), odds are you've used Exchange. It powers both Microsoft's SaaS platform (Microsoft 365, which seems to be changing names every other year), as well as on-premise environments through [Microsoft Exchange Server](https://en.wikipedia.org/wiki/Microsoft_Exchange_Server).

Now the next question: why do we even need to support it? In fact, a number of Exchange environments, starting with Microsoft's own free Outlook.com (previously Hotmail) service, can be used over open protocols (at least for email) such as IMAP or SMTP. However, that's far from being the case everywhere, as Exchange particularly thrive in corporate environments that forbid the use of these protocols for "security" reasons. The only alternative is to use one of the few proprietary APIs designed by Microsoft for interacting with their services (which, thankfully, are publicly documented and available).

Among those, we picked an API called [EWS](https://learn.microsoft.com/en-us/exchange/client-developer/exchange-web-services/explore-the-ews-managed-api-ews-and-web-services-in-exchange) (short for "Exchange Web Services"), which uses XML (SOAP) over HTTPS. This might sound like a weird decision to anyone familiar with Exchange, as even at the time EWS was slowly being deprecated in favour of the newer and shinier [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) API. In fact, while we we still in the early exploratory phase for this project, Microsoft announced the [upcoming retirement](https://devblogs.microsoft.com/microsoft365dev/retirement-of-exchange-web-services-in-exchange-online/) of EWS in their SaaS services, urging applications to migrate to Graph. Despite this, we decided to keep moving ahead with EWS for a few reasons:
* EWS's retirement date on Microsoft's SaaS services wasn't for another 3 years, and we bet on our ability to land at least email support for EWS with enough time to do the same for Graph before that deadline.
* EWS is not being retired on on-premises infrastructure, where Graph is unavailable. We want to support as many environments as possible, so we'll need to support EWS regardless.
* EWS is already supported by at least one well-established open-source email application: [Evolution](https://en.wikipedia.org/wiki/GNOME_Evolution), and being able to use it as a reference when Microsoft's documentation was lacking helped us a few times.
* EWS is a more appealing API for a native desktop application, specifically regarding notifications. While Graph does support live notifications for e.g. incoming messages, it's heavily based on push notifications, which require the kind of centralised infrastructure a web application would have. On the other hand, EWS supports pull notifications, where the client holds an open connection to the server and receives updates as they becomes available.

Note that for convenience, I'll refer to these APIs as "protocols" from now on. While they're not really at the same layer of the OSI model as, say, IMAP or SMTP, they sit at the same level from an implementation point of view.
# Open + closed = ... open?

A natural follow-up question to this explanation is: actually, why do we want to support Exchange in Thunderbird? After all, isn't Thunderbird supposed to be a champion of open standards? Why are you trying to taint my open-source with proprietary stuff?

It's true that Thunderbird's main mission is to support and facilitate communication based on open-standards. In fact, its own [mission statement](https://www.thunderbird.net/en-GB/about/mission-statement/) says so:

> Thunderbird facilitates cross platform, decentralised, open-standard communication, which puts the user in control of their data and workflow.

With this in mind, it is fair to question whether supporting a proprietary platform in Thunderbird, and doing so natively (as opposed to building an add-on for it) makes sense.

I think a way to answer this question is to question the often strict dichotomy that exists between open platforms and proprietary ones. I've sometimes observed open-source communities having an almost visceral rejection of anything that isn't also open-source, acting as if it was unnatural to be using those closed alternatives in the first place. Which I don't believe to be a realistic approach.

In an ideal world, I obviously would like to see every platform and every system being powered by at least a majority of open-source projects with total transparency. In the real world, however, many are limited by constraints they cannot control, which heavily influence the options available to them - for example, most Exchange mailboxes exist in corporate environments, where environments and policies are decided by a specific department, and individual employees outside of that department have little power to change things.

In this sense, a consequence of adopting an inflexible, hard-line stance on the "open vs closed" debate is that free and open-source software, which at its core is meant to offer users more freedom and choice regarding how they can interface with digital services, ends up leaving the same users with less choice.

Paradoxically, integrating with closed platforms can also help the project fulfil its mission to champion openness. If someone is looking for a solution that helps them use both their work and personal inboxes on the same application, I would argue that allowing them to use an open-source solution such as Thunderbird or Evolution goes further for the project's mission than pushing them towards a closed solution like Outlook. And although it's a very small sample size, I was glad to hear multiple people at FOSDEM tell me they had come back to Thunderbird because they were now able to use it with their Exchange mailbox, which I feel somewhat confirmed this belief of mine.

In the specific case of supporting Exchange APIs, although they're almost never relevant to personal mailboxes, a very significant part of workplaces or schools/universities use Exchange configurations that allows only those APIs. This represents a large enough amount of users that we decided it was worth embarking in this project.
# Adding some new to the old

Right. Now we've answered these initial questions, let's try to answer the next: how do we even support a new protocol in Thunderbird? As I mentioned in the introduction, the last time support for a new protocol in Thunderbird was over 30 years ago. This means there aren't a lot of people who took part in that early work and are still involved in the project today. And while we did understand our existing protocol implementations enough to maintain and improve them, our understanding of the overarching code architecture was clearly not good enough.

So let's rephrase that question: how were protocols supported in Thunderbird 20-30 years ago? Well, there's only one way to get to an answer: research it. Open the code, pull each thread one by one and figure out what kind of picture they make together. And [document](https://source-docs.thunderbird.net/en/latest/backend/email_protocols.html) as much of it as possible.

But that question lead to another one: do we want to apply the same playbook that was used all that time ago? After all, good development practices, methods and tools have evolved a lot since then, and we decided that this project would be a great opportunity to try to define what a modern protocol implementation looked like in Thunderbird.

This lead, in part, to the introduction of [Rust](https://rust-lang.org/) in the code base. This came up for the usual reasons (memory safety, rich ecosystem, decently friendly low-level language, etc.), but also because supporting Rust was made easier from the fact that Thunderbird is built on top of Firefox, and that Firefox has already been shipping with some Rust inside for a while. Due to this configuration, Thunderbird is able to inherit a few components from Firefox, including its build system which already came with Rust compiler support (although we still had to do [a](https://bugzilla.mozilla.org/show_bug.cgi?id=1860654) [few](https://bugzilla.mozilla.org/show_bug.cgi?id=1864624) [tweaks](https://bugzilla.mozilla.org/show_bug.cgi?id=1869860) to make it work for us).

A lot of our code architecture for protocol support relies heavily on C++ inheritance, though, which meant we couldn't use Rust for all of the new code that would be introduced in this project. Rewriting the existing architecture in Rust would have involved either a lot of code duplication, or a very heavy (and risky) refactoring, which would have blown up the scope of an already complex enough project. Ultimately, we decided to use Rust for the new protocol client, and to keep using C++ for the more Thunderbird-specific logic.

# Fighting technical debt with intentional design

The second prong for our approach to building modern protocol support has been being intentional with how we design and architecture new code (at least, as much as we could without throwing away the existing overarching architecture), specifically on the C++ side of things. A portion of Thunderbird's existing C++ code, like its IMAP implementation, is architectured in a way that makes this code pretty difficult to follow or understand, and we certainly don't want to replicate confusing patterns in this new code.

One example of these confusing patterns is the way asynchronicity is handled. A fair amount of asynchronous operations in Thunderbird are performed by giving an asynchronous function a "listener" object, which is essentially a collection of callbacks. These listeners are designed in interfaces, such that consumers aren't constrained to use a specific implementation, and some of our existing C++ classes implement those listener interfaces *on top of* implementing their own functional logic. This creates a few issues, for example:
* For operations that are stateful (i.e. they require data that is specific to the operation to be kept in scope throughout its duration), it means the class will be carrying state that is specific to the longer-lived consumer, but also to a short-lived operation that's being performed on that listener. This makes individual classes very heavy in state and confusing to reason with (because parts of its state end up having different scopes and lifetimes), and it can quite obviously be the source of many a concurrency issue.
* On top of this, asynchronous operations implemented this way become very difficult to trace, because the same listener object (which is also its own consumer) might be reused for several kind of operations (or several operations of the same kind), making following the thread of specific operations more difficult than it should be.

Let's take the example of the [`nsImapMailFolder`](https://searchfox.org/comm-central/rev/5d544ba521e128e3fbe113541b4bddc904f68bbb/mailnews/imap/src/nsImapMailFolder.h#204) class, which represents a mail folder using the IMAP protocol. Managing a mail folder is no simple task, but when we add into that all of the asynchronous listener implementations we end up with a huge monolithic class [which implementation](https://searchfox.org/comm-central/rev/5d544ba521e128e3fbe113541b4bddc904f68bbb/mailnews/imap/src/nsImapMailFolder.cpp#195-9033) spans over almost 9000 lines of code. This implementation includes methods such as [`OnStartRunningUrl`](https://searchfox.org/comm-central/rev/5d544ba521e128e3fbe113541b4bddc904f68bbb/mailnews/imap/src/nsImapMailFolder.cpp#4892-4903) (which come from [`nsIUrlListener`](https://searchfox.org/comm-central/rev/5d544ba521e128e3fbe113541b4bddc904f68bbb/mailnews/base/public/nsIUrlListener.idl), a listener interface commonly used for all sorts of operations throughout Thunderbird), as well as members such as `m_initialized` or `m_onlineFolderName` which refer to the long-lived folder, but also like `m_junkMessagesToMarkAsRead` or `m_curMsgUid` which refer to the current, short-lived operation. As you can imagine, this gets unwieldy fairly easily.

Other issues include the use the stateful "URL" objects to represent remote operations (while URLs are semantically expected to hold information that refers to a resource and how to operate on it, they shouldn't be really expected to, say, be updated with information regarding how that operation is going, or be used to get references to things like event sinks); or the lack of centralised implementations for common local operations (e.g. the local storage management side of operations like message copy needs to be implemented separately for each protocol, despite doing essentially the same thing regardless of the context).

After observing those issues, we decided on a few principles:
* Reuse as much common code and types as we can. This might sound obvious, but forcing ourselves to e.g. base EWS URLs on a "normal" and semantically correct URL object interface (which we inherit from Firefox) rather than allowing ourselves to create our own custom "EWS URL" type helped create constraints that prevented us from accidentally perpetuating the patterns we wanted to avoid.
* New classes should do the strict minimum required, and defer to other objects for functionality that wasn't semantically fitting with their model. This means, for example, moving away from a "folder" mega-class that worked almost entirely on its own, to a smaller class that only does what it needs to do (e.g. orchestrate operations on an email folder) and defers to even smaller classes for e.g. reacting to asynchronous output.
* New code should be made as generic as possible. I don't mean this in the syntactic/typing sense, but I'm rather referring to generally approaching new code with the idea that if it doesn't need to include logic or a workflow that is specific to a given protocol, then it should live outside of protocol-specific code and be entirely reusable by other protocols that might need similar functionality.

# That's all for now

Like I mentioned at the beginning of this post, this is not the end of the story. As always, I would like to thank [Thibaut](https://toot.cat/@CromFr) for helping proofread this first part. A second and (for now) final part should appear on this blog fairly soon (hopefully), and it will cover the more technical side of the project - now that we've defined its direction, it's about time we get our hands dirty!

Stay tuned, and see you again soon!


