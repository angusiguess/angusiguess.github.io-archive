---
layout: post
title: "Programming with a broken wrist"
date: 2017-05-15 15:00:00
---

# The Problem

I broke my wrist. I'm in a cast for probably 2 months and still have plenty of
programming to do. I just cant use my left hand, which is my main hand. I don't
want to buy new hardware if I can help it, so the solution has to be software.

Given that I'm a qwerty typist, I want to lean on that as much as possible.

## My home setup

At home I run a 2015 MacBook Pro running El Capitan, so I've done the following:

* Installed [Karabiner](https://pqrs.org/osx/karabiner/), which is a powerful key remapping app.
* Taken [this](https://gist.github.com/emory/4c0aa3b41958f8960c95) half-qwerty config as a starting point.

## Half qwerty

The half qwerty layout is, as far as I know, based on the [Mathias](http://half-qwerty.com/)
keyboard, which remaps each side of the keyboard to the opposite side when you hold
down the space bar. It's useful because if you're a touch typist you have key positions
symmetrically in your fingers.

Typing is still slower, but I feel like I'm learning fast.

## Tweaks

### Default key hold time

By default, Karabiner remaps keys after 200ms, which is enough to feel a bit frustrating.

In Karabiner's parameters, we can adjust the "Holding Key To Key Holding Threshold".

![setting the holding threshold](/hold-key.png)

### Where's my tab key?

Since I'm on the right side of the keyboard, I need a tab key, so I remapped \.

Given that I don't have a control key and I use emacs, I've also remapped alt on the right side to control.

## Config

My config so far is available at [this repository](https://github.com/angusiguess/righty)
