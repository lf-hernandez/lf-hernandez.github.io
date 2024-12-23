---
title: "LKMP Bug Fixing Summer 2024"
date: 2024-12-23T03:41:38-05:00
---
# My Linux Kernel Mentorship Journey

Engaging with the Linux Kernel community has been a transformative experience. It allowed me to take the leap and explore my interests in low-level programming, Linux, and open-source development. During the Linux Kernel Mentorship Program Bug Fixing Summer 2024, I not only had the opportunity to pick up new technical skills but also gained a deeper understanding of the various processes that go into Linux kernel development. I want to use this post to outline key aspects of my journey, including my contributions, tools I used, speaking engagements, and resources that helped me along the way. I couldn't be happier with my decision to apply and participate in this program.

## Contributions

Throughout the program, I submitted 10 patches, of which 5 made it upstream, that dealt with a diverse range of issues, from improving documentation to writing new kernel test suites:

### Unit Testing the Kernel Math Library

lib/math: Add int_sqrt test suite: This patch implemented a test suite for the integer square root function, ensuring reliability across diverse inputs.

lib/math: Add int_pow test suite: Expanded the math library’s test coverage by unit testing the the integer power function.

lib/math: Move kunit tests into tests/ subdir: Organized all math lib tests in their own tests/ dir in order to better organize directory structure and follow conventions found else where in the kernel. KUnit is relatively new (officially introduced in 5.5) and as tests have been added, maintainer David Gow gave me valuable feedback with regards to conventions (nomenclautre and organization) which prompted me to follow similar efforts carried out by Kees Cook accross the lib/ subsystem. 

### Code Refinement and Structural Improvements

Addressed grammatical errors and inconsistencies in subsystems such as hid and platform/x86, enhancing code readability.

usb: dwc3: remove unused sg struct member: Simplified driver structure by eliminating deprecated element.

### Documentation Updates

Enhanced documentation by correcting typographical errors and refining phrasing, ensuring clarity and consistency.

---

Each contribution provided me with lessons in adhering to kernel patch standards, engaging with maintainers, and responding to feedback.

## Tools and Techniques

Git Utilities: git format-patch, git send-email, and git worktree for patch preparation, submission, and patch/repo workflow.

Debugging: gdb, and QEMU to analyze and troubleshoot syzbot bugs.

Exploration: cscope, menuconfig, and vim/nvim (yes, prior to this I relied on modern code editors. I purposely used this program as an opportunity to finally use CLI-based workflow with vim/nvim and I can say I'm happily using it at work as well).

KUnit Framework: Writing unit tests with KUnit helped make me understand some kernel internals by allowing me to write test code against functions in isolation.

Mailing List Engagement: Collaborating on the Linux Kernel Mailing List (LKML) forced me to be more deliberate with crafting commit messages and integrate feedback.

## Resources

### Essential Books:

* The C Programming Language by K and R — an essential guide to learning C and one of my favorite technical books.
* Linux Kernel Programming (2nd Edition) by Kaiwan N Billimoria — excellent introduction to kernel development.
* Linux Kernel Development by Robert Love — another great book covering Linux internals.

### Mentor Insights

Office hours with Shuah Khan were immensely helpful as I was able to ask questions regarding my patches and learn from those worked on by my peers, as well as all the invaluable advice regarding career and speaking.

Ricardo B. Marliere and Javier Carrasco, both have done an amazing job providing support during the program. I appreciate all the time they provide mentees and how patient they are with all of us. I also will say their blogs were invaluable resources early on, check them out:

[Ricardo's Blog](https://marliere.net/)

[Javi's Blog](https://javiercarrascocruz.github.io/)

## Talks and Community Engagement

### Open Source Saturday Workshop: Orlando Dev Ops [Event Link](https://www.meetup.com/orlando-devops/events/301661699/)

Held on June 22, 2024, this was a 4-hour workshop where I, alongside a fellow Orlando DevOps member and open source developer, had the opportunity to share our experience contributing to open source and assisted every member in attendance to choose one open source project and attempt to submit a contribution. This was my first time sharing my experience with LKMP and delivering a technical walkthrough on how to pull a kernel tree, configure, build, and install it from source.

### Tampa Bay DevOps Days: "In The Deep End: My Experience as a Linux Kernel Mentee" [Event Link](https://devopsdays.org/events/2024-tampa/speakers)

On September 19, 2024, I delivered a presentation at Tampa Bay DevOps Days, sharing insights from my kernel development journey as well as sharing with the audience the various programs offered by LFX and how to apply to them.

### Orlando Devs Ignite: "Unit Testing the Linux Kernel" [Event Link](https://orlandodevs.com/ignite/)

ODevs Ignite The Holidays held on December 18, 2024, I delivered an ignite talk titled "Unit Testing the Linux Kernel." This presentation covered the role of KUnit in kernel development for ensuring code reliability, as a means to find potential bugs in existing code, and form of technical documentation for various kernel APIs and functions. I provided a practical guide to creating and executing KUnit tests through my own patches/test suites I worked on during the program.

## Looking Ahead

Reflecting on my experiences, I am excited to continue my journey in the Linux kernel. Future goals include:

1. Specializing in a subsystem (or two).

2. Contributing to higher impact development/bug fixing efforts.

3. Continuing to learn and improve my technical and collaborative skills within the kernel community.

Contributing to the Linux Kernel has been an immensely rewarding. Even though professionally I work higher up the stack (cloud, IoT, infra, and web), I look forward to furthering my impact and becoming a good member of the community as all the maintainers, reviewers, peers, and mentors have been with me.
