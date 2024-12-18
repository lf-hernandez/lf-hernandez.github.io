---
title: "Patch Workflow"
date: 2024-12-17T21:06:23-05:00
---
This post will serve as a reference for me to revisit and improve upon as I continue learning*

I joined the LFX Mentorship program for the Linux kernel not that long ago, and I've been working on finding an effective workflow for creating and managing patches. Initially, I created all my patches using `git format-patch` and relied on `git send-email` to mail them out. However, during my time in the Linux Kernel Bug Fixing Summer 2024 program (LKMP), I was introduced to `b4` and `git worktree` through interactions with mentors, Ricardo B. Marliere and Javier Carrasco. And while it's evident these two tools aim to make the kernel developer's life easier, I've had a bit of learning curve adapting to them.

Nonetheless, I want document my current workflow for the sake of posterity:

```bash
$ git clone --bare git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git ~/Linux/worktrees/.bare && cd ~/Linux/worktrees/.bare

$ git remote add gregkh/tty git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/tty.git
$ git remote add kselftest git://git.kernel.org/pub/scm/linux/kernel/git/shuah/linux-kselftest.git
$ git remote add mainline git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
$ git remote add next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
$ git remote add stable git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
$ git remote add staging git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/staging.git
$ git remote add github git@github.com:lf-hernandez/linux.git
$ git fetch --all

$ git worktree add -b mac_hid_patch ../<related_patches>/mac_hid_patch next/master
$ b4 prep -n <some-descriptive-name>
$ vim <file-to-change>.c
$ git commit -a -s

$ b4 -c
$ b4 prep --check
$ b4 send -o $PATCH_DIR
$ b4 send --reflect
$ b4 send
```

### Breakdown

I start by cloning a bare repository, specifically the mainline kernel tree. This is a special type of repository that only contains the `.git/` directory and no working directory with sources. This will help manage my worktrees and acts as a central hub for tracking repository history and data.

Next, I add remotes I'm interested in working with as well as my fork of the kernel to aid in collaborative patches. These will serve as the base for my worktrees.

`git worktree add` is what actually creates the working directory for a specific branch. I pass the path and branch name. In this case I'm using an intermediate directory to house related patches. This is an organizational choice on my end, but may change as my workflow evolves.

Now that I have my worktree in place, I prepare my patch with `b4`. I've yet had the need to create a patch series. All my patches thus far have dealt with single file changesets so I'm leaving the cover letter empty for now. In most cases, you would use this as an opportunity to explain what the patch series is aiming to achieve and why. Once I've created the b4 topical branch and made and tested my changes (compiling, testing potential fixes via qemu/gdb, running kunit, etc.), I stage my changes, create the commit and sign it.

Normally here is where I would manually run `get_maintainers` (and in some cases `git blame`) to get the list of recipients for the patch and manually add them with `git format-patch`, now I use `b4` to grab the appropriate recipients and add them to the patch automatically. Similarly I would also manually invoke `checkpatch` against a recently created patch and fix any issues before proceeding, instead I now use `b4 prep --check` which does so under the hood. Assuming everything checks out, I go ahead and write a copy of the patch to my patches directory (this has already saved me a few times, it's perfect for looking back on previous work or in my most recent case running `b4 shazam` on it to apply an existing patch). From there I send out a test email to myself via `--reflect` and confirm that things look correct before sending.

This is where I’m at for now. I’ve only sent a few patches using this approach, and, as expected, I’ve made mistakes along the way. But that’s part of the learning process, and I’m gaining more experience with every step.

For further reading, I highly recommend checking out Ricardo's and Javi's blogs:

[A better workflow for kernel debugging](https://marliere.net/posts/2/) by Ricardo B. Marliere

[b4 for Linux kernel contributors](https://hackerbikepacker.com/b4-for-kernel-contributors) by Javier Carrasco

And of course, the docs:

[Git Worktree](https://git-scm.com/docs/git-worktree)

[b4](https://b4.docs.kernel.org/en/latest/index.html)

