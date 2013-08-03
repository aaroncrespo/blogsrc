---
layout: post
title: "Clojure and Leiningen Coexisting With inputrc"
date: 2013-08-03 09:43
comments: true
categories:
---

It is common to remap bash history keys with an `inputrc ` file.
For example the following:

```bash
"\e[B": history-search-forward
"\e[A": history-search-backward
```

Will allow you to search your bash history with arrow keys forward and back based on the already entered string. If there is no string it will behave as 'usual'. Similar to zsh's history search.

However if you have ever tried to toy with a clojure REPL with these settings you probably have, or will, notice that your history doesn't work how you would expect it to:

``` clojure
user=> (zero? 2)
false
;; Press up
(reverse-i-search)`':
```

A [bug](https://github.com/jline/jline2/issues/51) in jline (a command parsing library leiningen uses) causes it to mis-interpret a user's .inputrc. Needless to say having no history in a REPL is a significant impediment. So lets fix it:

We can either create a local leiningen repository and point it to a newer version of jline with the bug fixed. Or we can temporarily overwrite the self configuring behavior. Jline is [configured](https://github.com/jline/jline2/wiki/Configuration-Properties) via its own `.jline.rc`. The default behavior looks for your `.inputrc`, it will then try and automagically configure itself to match your environment.

The location of your input.rc is configurable within jline so that a jline specific inputrc configuration settings can be used. The default setting unsurprisingly points to your inputrc:
```bash
jline.inputrc=$HOME/.inputrc
```
To set a custom configuration you can make a new inputrc configured to your preference (for now without history key re-maps). Or, if you are okay with the default bash like behavior point it to a blank file.

```bash
# in a .jline.rc file in $HOME
jline.inputrc=$HOME/.none
```
Restart your REPL or terminal and your history will work how you expect again.