---
title: "Random tools #1"
description: Over time, I came to encounter a few tools, addressing different use cases and/or issues. Recently, I started listing these tools in order to share some of them with you, whether you already know them or not, in smaller posts like this one, without going too much in depth with them. Welcome to the first episode of "Random tools"!
tags: [ 'tool', 'tilix', 'terminal', 'tabsearch', 'firefox' ]
publishDate: 2018-07-08T00:00:00+02:00
draft: false
thumbnail: /random-tools-1/tilix-thumbnail.png
---

Over time, I came to encounter a few tools, addressing different use cases and/or issues. Recently, I started listing these tools in order to share some of them with you, whether you already know them or not, in smaller posts like this one, without going too much in depth with them.

I'm giving posts like this one two objectives: help people discover tools that often help be with small or bigger tasks, sometimes on a daily basis, and allow me to effectively share some of my knowledge with everyone (which is the main goal for this blog) while requiring less work and efforts than the other posts you can usually find around here, which are mainly focus on one specific topic.

So let's get things started with this first episode of "Random tools"!

## Tilix

Let's start with one tool that a lot of people already know: [Tilix](https://gnunn1.github.io/tilix-web/). It's a terminal for GNU/Linux systems using GTK+3, which you might also know as its older name: Terminix.

Tilix is currently my main terminal on my computer, and almost the only one I use period (which, given my current job being systems engineer, makes it one of my main work tools). Among the amazing features it offers is the ability to split the screen in panes as much as you want, making it super easy to work with a task requiring data from several hosts at the same time, or checking in real time the effects of a patch on a system while directly applying it without losing sight of each, for example.

Add to that Tilix's "Quake mode", allowing you to make Tilix appear on the top of your screen and disappear from it without losing the current session (just like the Quake console minus the animation), and that makes the perfect tool for me to work with, because I never have to waste loads of time trying to bring it my terminal back at my desktop's foreground while jungling with a few other windows.

![Example GIF](/images/random-tools-1/tilix.gif)

Keyboard shortcuts for splitting are `Ctrl+Alt+R` for a vertical split, and `Ctrl+Alt+S` for a horizontal one. Quake mode can be enabled by editing the keyboard shortcut bound to Tilix in your system settings (or adding one if there's none yet) so it runs `tilix --quake` (or `tilix -q`) instead of `tilix`.

## TabSearch

This tool is a Firefox extension that will help you save a lot of time if you always carry loads of tabs open at once, sometimes even spread through multiple windows. In this kind of configuration, it's usually a frequent waste of time to remeber where a specific tab is located on a tab bar, and what window this tab bar belongs to.

If added to Firefox's top bar, [TabSearch](https://addons.mozilla.org/en-US/firefox/addon/tab_search/) will provide you with a small graphical user interface that will allow you to browse through every tab that is open in the current window, open in another window, or has been recently closed. This interface also provides a search feature, and can be toggled by hitting `Ctrl+Shift+F`. You can then browse through the tabs using the arrow keys or search within them, and hit `Enter` when you find the tab you want to switch to.

![TabSearch's interface](/images/random-tools-1/tabsearch.png)

*This screenshot isn't mine and comes from [the extension's page on AMO](https://addons.mozilla.org/en-US/firefox/addon/tab_search/)*

The extension also provides you with other ways to interact with tabs, which are listed in [its documentation](https://github.com/reblws/tab-search/#shortcuts). It will also add to Firefox's top bar a count of the tabs that are currently opened in the window.

## In the next episode

In order to make these posts quick to make, I've decided to only cover two tools per episode, so that's the end of this first one! I'm not sure about when the next one will get its release, but I've already the tools to talk about in mind.

If you like that one or want to share some feedback on it with me, feel free to hit me up on [Twitter](https://twitter.com/BrenAbolivier), [Mastodon](https://mastodon.social/@babolivier) or [Matrix](https://matrix.to/#/@brendan:abolivier.bzh)!

I'll see you before then, next week in fact, for what should be a technical walkthrough. Until then, have fun, and see you next week! 😄
