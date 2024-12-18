---
title: "Patch Workflow"
date: 2024-12-17T21:06:23-05:00
---
I will use this post as a reference to look back to and iteratively improve as I continue to learn. 

My first few patches were all created using `git format-patch` and `git send-email`. But over my time in the LKMP Summer 2024 program, I've been swayed to use b4 and git worktrees through my interactions with my mentor Ricardo B. Marliere and fellow mentor Javier Carrasco. Interestingly enough, I'm going through a bit of a learning curve. Even in my short time cloning entire git trees and creating branches for each patch and using `git format-patch`, I grew accustomed to that workflow and have struggled to leverage b4 and git worktrees effectively.

This is more of a consequence of my lack of familiarity, and I expect with time and thorough re-reading of the docs I'll get up to speed.

Nonetheless, I'm going to go over my current workflow for Linux:

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

$ git worktree add ../sysctl_patches/mac_hid next/master
$ b4 prep -n <some-descriptive-name>
$ vim <file-to-change>.c
$ git commit -a -s

$ b4 -c
$ b4 prep --check
$ b4 send -o $PATCH_DIR
$ b4 send --reflect
$ b4 send
```

A couple of things to note for context:
1) I have all my Linux-related work in my home directory ~/Linux
2) I have all my patches conveniently stored in a directory which I reference via $PATCH_DIR env var
3) I threw in some remotes I've been working with recently but this would change based on the subsystem the changes I'm doing relate to and whether there is a git repository for it

So I start off cloning a bare kernel tree repository, specifically mainline. This is a special type of clone that only contains the .git/, no working directory with its files. This is a prereq for creating worktrees. From my understanding, it is a central hub of sorts for my worktrees. All the history and repository data will be tracked there and the worktree directories of the checked out branches will hold the actual files. 

I then proceed to add remotes I'm interested in. These will serve as bases to branch off for my worktrees.

worktree add is what actually creates the working directory, I pass the path ending the name of the new branch I want to create and which remote I want to track from. Although in this case I'm using an intermediate directory to house a few related patches. This is more of an organizational preference on my end. This may change, but it's something I'm trying out.

Now that I have my worktree in place, I proceed to setup my patch with b4. I've yet to create a patch series so in this case I'm leaving the cover letter empty for the most part. In most cases, you would use this as an opportunity to explain what the patch series is aiming to achieve and why. So I create the b4 topical branch and make my changes. Once I've tested the changes (compiling, testing change fixes an issue via qemu/gdb, running kunit, etc.) I create the commit and sign it.

Normally here is where I would manually invoke checkpatch against a recently created patch from that commit. Instead I use b4 to grab the appropriate recipients and add them to the patch. Instead of manually running checkpatch I use `b4 prep --check` which does so under the hood with some extra flags I wasn't using before `--no-summary --mailback --showfile` and assuming everything checks out I go ahead and write a copy of the patch to my patches directory (this has already saved me a few times, it's perfect for looking back on previous work or in my most recent case running `b4 shazam` on it to apply to a fresh patch because I had to start from scratch). From there I send out a test email to myself via `--reflect` and check that things look ok before doing the actual send. 

This is where I'm at the moment, I've only sent out a couple of patches using this approach and have made my share of mistakes on both. It happens, and I'm learning more and more from these experiences.

For solid references on these tools check out Ricardo's and Javi's blogs:
[A better workflow for kernel debugging](https://marliere.net/posts/2/)
[b4 for Linux kernel contributors](https://hackerbikepacker.com/b4-for-kernel-contributors)

And ofc the docs:
[Git Worktree](https://git-scm.com/docs/git-worktree)
[b4](https://b4.docs.kernel.org/en/latest/index.html)

Suggestions are welcomed!
