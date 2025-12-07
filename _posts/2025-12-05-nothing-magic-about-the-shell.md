---
title: "CMD vs ENTRYPOINT: There is nothing magic about the shell"
layout: post
category: shell
tags:
    - tech
    - programming
    - docker
    - shell
date: 2025-12-05
published: false
---

I am currently reading the book [Kubernetes in Action][1] by Marko Lukša. He gave such a great explanation of the difference between CMD and ENTRYPOINT in Docker (and Kubernetes) that it made me want to write a post on it. It taught me something about the shell I've never really wrapped my head around before.

## CMD vs ENTRYPOINT

Before I had a better understanding of the shell I had to look up the distinction between these two numerous times. I always seemed to end back up at [this Stack Overflow answer][2] to refresh my memory. I usually defaulted to CMD as I ran a few multipurpose images and understood that CMD could be easily overridden. I also understood that the default Docker entrypoint was running the shell (`/bin/sh -c`) and figured I shouldn't mess with that. Don't I need the shell to run my command?

Lukša defines the CMD and ENTRYPOINT as follows:

- ENTRYPOINT defines the executable invoked when the container is started
- CMD specifies the arguments that get passed to the ENTRYPOINT


[1]: https://www.manning.com/books/kubernetes-in-action
[2]: https://stackoverflow.com/a/34245657/13242100

# TODO

- entrypoint exec mode
    - does read env vars
- exec using execve to replace shell

