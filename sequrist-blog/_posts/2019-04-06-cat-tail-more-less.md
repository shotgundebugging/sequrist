---
layout: post
title: "Cat tail! More or less"
date: 2019-05-10 23:50:25 +0300
categories: blog
---

I've been using `cat` for ages to display contents of files. I used it very rarely to actually concatenate files together.
A few years ago, [it was revealed](http://www.openwall.com/lists/oss-security/2015/09/17/5) that `cat` interprets escaped sequences. If you've been doing any kind of security work, you know how that can turn bad.
I've already been using `tail -f` to follow logs, since I started doing web development. I always had the problem that I wanted to scroll back but, as new items were added, it would automatically scroll me back to the end of the file.
At one point I stumbled upon [a good alternative](https://www.linux.com/community/blogs/129-servers/30056), namely using `less +F`. I loved that you can switch between examine (`^C`) and follow mode (⇧`F`).

<!--more-->

`head`, `tail` & `more` also interpret escaped sequences. Good news is that you can ditch `head` and use `less` in combination with it's LINES option.

`env LINES=10 less file.txt`

Here's a cool use case for `less` let's say you have a directory of text files and you quickly want to skim through them.

`ls | xargs less --prompt=%x --squeeze-blank-lines`

You can navigate with `:n` to the next file and use `--prompt` to see the name of the next file which `less` will open. Prompts are cool, you can use them to display line number, percentage of how much you read from the file, percentage of how many of the files you went through.

`man` also uses `less` so learning it's tricks is a great way to speed up navigating the UNIX documentation.

When using `less` to read documentation to can jump to a certain chapter by using the `--pattern` option.

`less --help | less --pattern=MOVING`

The above will display the`less` docs starting with the section on "MOVING."

And finally, as I found out from [Gary Bernhardt](https://www.destroyallsoftware.com/screencasts/catalog/pretty-git-logs), using `less` is also great for doing pretty pagination.

`echo long_file.txt | less -FXRS`

Let's break this apart:
- `-F` tells `less` not to page if it doesn't have more than one page.
- `-X` doesn't clear the screen after exiting.
- `-R` is for passing color codes through.
- And finally `-S` which chops long lines.
