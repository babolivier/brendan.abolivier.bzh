---
title: Manage your passwords with pass
description: Let's talk about passwords. Basically, that's the things you're supposed to keep different for each account you have on the Internet. Either you don't do it, do it partially, or have a password manager do it for you. This week, I'm writing about pass, a simple and minimal password manager mainly consisting in a 699-line long bash script, which I've been using for some months.
tags: [ 'password', 'manager', 'pass', 'passwordstore', 'store' ]
publishDate: 2018-05-21T00:00:00+02:00
draft: false
thumbnail:
---

Let's talk about passwords. Basically, that's the things you're supposed to keep different for each account you have on the Internet. Either you don't do it, do it partially (like a mix between a leet-speak version of the service's name and a fix part, with an uppercase letter and a character that's neither a letter nor a number at some place, such as `mySup3rw3bs!t3MyUsualPassword`), or have a password manager do it for you.

I've had quite a hard time finding a password manager that fits my needs. During the past few years, I've tried quite a few of them, and eventually stopped using them one after the other. LastPass because of its poor UX on points that mattered to me, and I couldn't feel safe trusting that much into such a centralised and closed service. Keepass because it was a pain to synchronise my database between all devices. Passbolt because it focuses on a team use case and I want something designed for individuals. You name it.

After a while, I started trying to get a description of what I wanted. To me, the ideal password manager must be:

* free software
* security audited
* synchronisable across devices
* self-hostable
* easy to set up
* easy/quick to use

I realised that was quite an idealistic description, and thought I was done with password managers. To be fair, to this day, I still haven't found one that match all of my criteria, though the one I'll be talking about in this post gets quite close.

Also, let me get things straight first: the last two points in the list above are using the relative definition of "easy", i.e. what's easy to set up/use to me, as someone who has some technical knowledge and background. Specifically, the solution I'll be writing about in this post would be labelled as quite painful to use by somebody who isn't used to bash, git et al.

## It's all about simplicity

[Pass](https://www.passwordstore.org/) is a minimal and very simple password manager which consists in a 699 line long bash script (including comments). It stores your password as files in a given directory (the "store"), and encrypt them using [GnuPG](https://gnupg.org/). That way, you can organise your passwords as you want, in as many sub-directories as you wish, and they will be stored, possibly along with some metadata, in a somewhat-secure fashion.

Notice here that I made a compromise on my criteria of an ideal password manager, here, because, as far as my knowledge goes, pass hasn't got a security audit yet (only GnuPG did). I consider it safe enough for my personal use, though.

Pass also has both CLI and GUI clients for most platforms, including OS X, Android, iOS and Windows, and also some browser extensions, but I'll only cover the basic command-line use of the bash script here. All clients and extensions can be found [here](https://www.passwordstore.org/#other), though.

## Creating the store

I won't cover installation, which is already covered on [pass's website](https://www.passwordstore.org/#download) and should be quite easy on most systems.

You'll also need to generate a GPG key, which is the pass equivalent of the store's master key/passphrase, if you haven't got one, which I also won't cover here since there's already great resources for that on the Internet.

Once pass is installed, let's initialise a store with

```
pass init GPG-ID
```

Here, `GPG-ID` is the identifier of the key you'll use to encrypt your passwords. It can be the key's fingerprint (in the case of my own key, `E1D4B7457A829D771FBA8CACE860157274A28D7E`) or one of it's associated email addresses (which, in my case, can be `hello@brendanabolivier.com`).

It will then initialise a store in a directory, which path is `~/.password-store` and is created if it doesn't exist. This directory is the one in which pass will work in every call you'll make in the future. This value can be overriden by setting the environment variable `PASSWORD_STORE_DIR`.

## Adding an existing password

Because you had accounts on the Internet before starting using pass, you might want to store their passwords in your brand new password store.

To insert a password into your password store, just run

```
pass insert PASSWORD-NAME
```

Where `PASSWORD-NAME` is the name you'll give to this entry. If you want to manage your entries with sub-directories, the entry name can also be a relative path to the password store (e.g. `pass insert hostProviders/ovh` will create an entry in the sub-directory "hostProviders"). If a sub-directory doesn't exist, pass will create it for you.

It will then prompt you for your password, which you can just paste and validate, and an encrypted copy of it will be stored in the password store. For example, if the entry name is `hostProviders/ovh`, it will store an encrypted copy of my password in `~/.password-store/hostProviders/ovh.gpg`.

You might also want to add metadata to your password, such as the account's login, or the service's URL, which some pass clients can use. You can do that by appending the `-m` flag to your `pass insert` call (before the `PASSWORD-NAME`), which allows you to write your entry using more than just one single line and save it using `Ctrl+D`.

In case of multiline entries, it's usually better to start an entry with the password as the first line's only content, and then add your metadata on the following lines. The reason for that is because, to pass, a non-multiline entry is just a one-line long file with the password as the only content. Having the first line only including the password will help pass handle multiline entries the same way as a single line entry.

In the end, your multiline entry would look like this:

```
mySup3rw3bs!t3P4ssw0rd
login: me@me.tld
url: mysuperwebsite.com
```

It might be worth noting that if you come from another password manager, there might be a migration script aiming at migrating all of your entries to pass instead of doing it manually, one at a time. Migration scripts for most password managers can be found [here](https://www.passwordstore.org/#migration).

## Creating passwords

Of course, one of the good things with having a password manager is having it generate different strong passwords for each service you have an account on. Generating a password with pass is as easy as calling:

```
pass generate PASSWORD-NAME
```

As with `pass insert`, this will create a `.gpg` file at the desired location, and will this time fill it with a 25-character long password. If you want the password length to be something else than 25, you tell pass by appending the desired length after the `PASSWORD-NAME`.

Once the password is generated, pass will print it into the terminal, so you can copy it. If you don't want it to appear on your screen, you can also append the `-c` flag to your call, right before the `PASSWORD-NAME`. Pass will then copy it into your clipboard, which it will clear after 45 seconds (the delay can be changed by setting the environment variable `PASSWORD_STORE_CLIP_TIME` to the number of seconds you want).

Another useful trick is appending metadata to the newly generated password, like we've seen before. It's obviously possible to edit an existing password (using `pass edit PASSWORD-NAME`, which will open an unencrypted copy of the password entry in vim), but I personally prefer to never have pass printing my password on a screen.

To achieve that, we can first call `pass insert -m PASSWORD-NAME`, which will prompt for the password and its metadata, leave the first line blank and fill the following one with metadata before hitting `Ctrl+D`. We can then call `pass generate -ci PASSWORD-NAME`. Note the `-i` flag (which stands for "in place"), which means that the entry we want to generate a password for already exists, in which case pass will replace the entry's first line with the newly generated password, and leave the rest of the file as it was.

You now have your newly generated strong password copied to your clipboard, and the desired metadata in its file.

## Retrieving passwords

It would be quite useless to have all your passwords stored in your store without being able to retrieve them and use them. As everything with pass, this is quite easy:

```
pass show PASSWORD-NAME
```

Which you can even shorten as:

```
pass PASSWORD-NAME
```

Pass will then print out the corresponding password, along with its metadata (if it has any) in the terminal. If you don't want the password to be printed out, but rather to be copied to your clipboard, just append `-c` before the `PASSWORD-NAME`, just like `pass generate` (and just like `pass generate`, it will clear the clipboard after 45 seconds (again, this delay can be overriden using the `PASSWORD_STORE_CLIP_TIME` environment variable)).

You might also prefer not having to fire a terminal and type a command line in order to get a password you'll then copy to the website. In that case, you might be interested in using one of the few browser extensions available, such as [PassFF for Firefox](https://github.com/jvenant/passff#readme) or [Browserpass for Chrome](https://github.com/browserpass/browserpass#readme), which you can use to automatically fill in login forms using passwords from your store and their metadata. For what it's worth, I've been using PassFF for quite a while now, and it works pretty well.

## Synchronising passwords

Because I always have more than one device, one thing I'm really looking for in a password manager is its ability to synchronise with other devices easily. This is the reason I stopped using Keepass, because having to manually copy your database across all of your devices each time you add/remove/change an entry was really painful.

Where I become really picky is that I don't want to be stuck with a proprietary service's hosting such as LastPass's or Dashlane's. I want to control where I send my passwords, who can access them, etc.

Once again, pass choses simplicity, by implementing a great compatibility with git, letting it do all of the versionning and networking, which is, obviously, optional.

If you want to synchronise your own password store with a git repository, create an empty one somewhere (I personally did that on one of my own servers, but a GitHub/GitLab/Gitea/etc. repository will, of course, work as well), grab its URL and run

```
pass git init
pass git remote add origin REMOTE-URL
```

Where `REMOTE-URL` is the repo's URL.

This will initialise a local git repository at the root of you password store, and also create a commit containing all your store's content. Note that the `pass git` commands' syntax follow the standard git commands'. That is because `pass git` will actually run every git command you give it in the store, whatever your current working directory is. This means that you can basically use every git command you want, as long as you prefix them with `pass`, the commands will affect your password store and nothing else.

Now that the git repository is initialised in the password store, each time you'll create, remove or edit a password, pass will automatically create a commit for that, so you only have to run `pass git push` now and then to synchronise your local password store with your remote copy.

In my case, I like to have a copy of my password store on my phone, and to manage it using [the Password Store Android app](https://github.com/zeapo/Android-Password-Store) (available on [F-Droid](https://f-droid.org/repository/browse/?fdid=com.zeapo.pwdstore) and [Google Play](https://play.google.com/store/apps/details?id=com.zeapo.pwdstore)), to which I just have to give the URL and credentials required to clone the repository, and the GPG key to use when trying to decrypt passwords, and I can instantly use my passwords on my smartphone.

Of course, since pass manages your passwords files and directories, you can have multiple sub-directories in your password store, each one of them having a different git remote. For example, most of my passwords are pushed to a remote repository on a server I own, except for one folder containing internal passwords we use at [CozyCloud](https://cozy.io), which are synchronised with an internal repository we have.

## To infinity and beyond

Of course, I haven't described all the features pass has. This post only describes the few I personally use, along with some setup instructions, and doesn't really cover the various ways in which one can use it. Now it's yours to play with it! 😉

Thanks for reading through this post, and huge thanks to [the amazing feedback and attention](https://twitter.com/BrenAbolivier/status/995973756240777217) you gave following my [latest post on Matrix](/enter-the-matrix), that's hugely appreciated.

The length and complexity of the said last post brought some fatigue with it, which explains this one's lateness. Taking that into account, and given the fact that I'm working really had on [Trancendances's immersion{s}](http://www.immersions.bzh) party in Brest that's taking place in less than two weeks, I don't think I'll be publishing any more post in the next couple of weeks (except maybe a very small one on a couple tools I discovered recently, but that's far from sure).

I'll see you after that, most likely in a bit less than three weeks, for a brand new blog post (of which I already know the topic, and it'll be a completely non-tech one, for a change!). See you then!
