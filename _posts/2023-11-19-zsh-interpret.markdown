---
layout: post
title: zsh interprets a standalone redirect as a `cat` command
date: '2023-11-19 21:21:52'
tags:
- shell
---

I use `zsh` and yesterday I was testing a program with `myprogram 2>/dev/null`. In one iteration, I accidentally copied a newline character after the `myprogram` command and the zsh prompt looked something like

<!--kg-card-begin: markdown-->

    $ myprogram
    2>/dev/null

<!--kg-card-end: markdown-->

The program appeared to hang after printing out the expected output. I removed the extra newline character and the issue went away. That's weird.

I reproduced the issue with `echo` just fine.

<!--kg-card-begin: markdown-->

    $ echo hello
    2>/dev/null

<!--kg-card-end: markdown-->

My `zsh` has [bracketed paste mode](https://cirw.in/blog/bracketed-paste), which can be verified by the checking the verbatim input of the command by pressing ctrl-v first.

<!--kg-card-begin: markdown-->

    $ ^[[200~echo hello

<!--kg-card-end: markdown-->

`^[[200~` is the escape code for bracketed paste mode. So zsh should just execute the paste (stored in a buffer) as two commands `echo hello` and `2>/dev/null`.

And sure enough, I could reproduce the shell hanging by just running `2>/dev/null`.

What does a standalone `2>/dev/null` even do?

<!--kg-card-begin: markdown-->

    ~ ❯❯❯ pstree -p 2303
    zsh(2303)─┬─cat(14128)
              └─zsh(2321)

<!--kg-card-end: markdown-->

Turns out for some reason, zsh inteprets a standalone [redirect](https://zsh.sourceforge.io/Doc/Release/Redirection.html) as a cat command with redirect(s). And `cat` by default is waiting on input from stdin - that's why the program appears to hang. `bash` doesn't have this behavior.

