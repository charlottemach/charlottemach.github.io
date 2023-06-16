---
layout: post
title: "Google Foobar Challenge"
date: 2023-03-12
tags: code challenge google
---

A few weeks ago I was googling random things (more specifically how to rename a remote branch as my naming had impacted a colleagues' local environment), when I got a pop-up.

![Google search with secret pop-up](/assets/images/foobar1.png)

A few years ago Google created this secret challenge as a hiring tool, I just didn't know it was still around! The URL is [https://foobar.withgoogle.com/](https://foobar.withgoogle.com/), but without an invite you can't access the content. It no longer seems to be used for hiring, but it's still a fun challenge, especially if you enjoy sites like [adventofcode](https://adventofcodecom/) or [Project Euler](https://projecteuler.net/).

## How does it work?

After getting an invite pop-up (or a code from a friend), you get the option to request a challenge. There are five levels to complete and each level has a number of challenges. 

Your overall goal is to defeat the evil Commander Lambda who is trying to destroy Bunny Planet. Each challenge helps you infiltrate the organization the Commander leads and rise in the ranks until you've defeated a doomsday device and saved all the bunnies. Nobody said the backstories for code challenges need to make sense.

![foobar help](/assets/images/foobar2.png)

## The setup

Each level has up to three challenges, each of which comes with a time limit. There's plenty of time for each challenge. Most of the ones I had allowed for multiple days to solve it.

![foobar levels](/assets/images/foobar5.png)
Once you request a challenge you get an extra folder with a text file that introduces the challenge problem.

You're restricted to using Python or Java. There are two stubs for each challenge, `Solution.java` and `solution.py`, which you can edit, run against a number of test cases and finally submit.

## The challenges

Even though the challenges work based on some backstory in the overall "help destroy the evil corp", there are underlying algorithmic concepts.

Knowing what these are and how they work is very helpful for solving the challenges. Some I came across needed dynamic programming, graphs (shortest path, negative cycles, [DFS](https://en.wikipedia.org/wiki/Depth-first_search)), [cellular automata](https://en.wikipedia.org/wiki/Cellular_automaton).

Sometime during the process Foobar does ask for your details, but you don't have to submit them, or can choose to do so later.
![foobar recruit](/assets/images/foobar3.png)
Keep in mind that (according to other blogs) the challenge hasn't been used for hiring since 2020. 

## Success! 

Finishing all challenges gives you an encrypted message.
![foobar finished](/assets/images/foobar4.png)
Decrypting that returns
```
b"{'success' : 'great', 'colleague' : 'esteemed', 'efforts' : 'incredible', 'achievement' : 'unlocked', 'rabbits' : 'safe', 'foo' : 'win!'}"
```

It's definitely a fun one, and I learned a few new concepts in the process!

Unfortunately it's quite easy to come across people posting solutions when researching the topics, so if you don't want to see any solutions during your first attempt keep an eye out for spoilers on StackOverflow.
