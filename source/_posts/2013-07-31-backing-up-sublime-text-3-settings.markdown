---
layout: post
title: "Backing up Sublime Text 3 settings"
date: 2013-07-31 09:45
comments: true
categories:
---

Setting up Sublime Text 3 works a little different than Sublime Text 2, and I like to keep my settings [backed up](https://github.com/aaroncrespo/dotfiles) but since some ST3 packages and ST3 itself are under active development and varying stages of release my github backups tend to get out of date.

With that I decided to back up to DropBox (similar steps work work with Google Drive). There is an added side effect of this backup method that makes sharing and syncing the backups extremely easy accross multiple computers.

1. Use [package control](http://wbond.net/sublime_packages/package_control). This allows you to easily install packaged and keep them up to date.
1. Close Sublime Text
1. Move all your packages and settings to a drop box folder:

```c
# Make your backup Folder.
$ mkdir ~/Dropbox/ST3/
$ cd  ~/Library/Application Support/Sublime Text 3

# Move the files.
$ mv Packages/ ~/Dropbox/ST3/
$ mv Installed\ Packages/ ~/Dropbox/ST3/

# Make symlinks
$ ln -s ~/Dropbox/ST3/Packages/ Packages
$ ln -s ~/Dropbox/ST3/Installed\ Packages/ "Installed Packaes"
```

Now packages and Sublime Text update themselves and my backups always remain current.

__Bonus:__ syncing another system with my new backups.

```c
$ cd  ~/Library/Application Support/Sublime Text 3
# Remove packages and settings
$ rm -rf Packages/ ~/Dropbox/ST3/
$ rm -rf Installed\ Packages/ ~/Dropbox/ST3/

# Make symlinks
$ ln -s ~/Dropbox/ST3/Packages/ Packages
$ ln -s ~/Dropbox/ST3/Installed\ Packages/ "Installed Packaes"
```
