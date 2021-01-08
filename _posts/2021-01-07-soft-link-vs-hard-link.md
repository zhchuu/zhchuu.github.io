---
layout: post
title: "Soft link vs. Hard link"
date: 2021-01-07
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## Soft link (Symbolic link)
A soft link is an actual link to the original file. If you delete the original file, the soft link has no value. Because it points to a non-existent file. It has the characteristic below:

> - Can cross the file system.
> - Allows you to link between **directory**.
> - Has **different inode number** and file permissions than original file.
> - Has only the path of the original file, not the contents.


## Hard link
A hard link is a mirror copy of the original file. If you delete the original file, the hard link will still has the data of the original file. Because hard link acts as a mirror copy of the original file.

> - Can only work on the same filesystem.
> - Can't link between directories.
> - Has the **same inode number** and permissions of orginal file.
> - Permissions will updated if we change the permissions of source file.
> - Has the actual contents of original file.


## Hard link vs. Copy
If you copy a file, it will just duplicate the content. The variation in the original file will not affect the copied one. Also, hard link will not cover extra space.

## Ref
[Explaining Soft Link And Hard Link In Linux With Examples](https://ostechnix.com/explaining-soft-link-and-hard-link-in-linux-with-examples)
