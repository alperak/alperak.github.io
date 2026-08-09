---
title: "Open-Source in Action: OrangePi RV2 (RISC-V) Now Supported in the Yocto Project"
date: 2025-07-01 11:36:59 +0300
categories: [Embedded Linux, RISC-V, Yocto, Open-Source, Contribution]
tags: [embedded-linux, riscv, yocto, open-source, contribution]
---

Over the last weeks, I dedicated myself to adding support for the RISC-V based OrangePi RV2 SBC (Single Board Computer) to the `meta-riscv` layer in the Yocto Project.

> Curious about the implementation? Check out the pull request: [riscv/meta-riscv#535](https://github.com/riscv/meta-riscv/pull/535)

What made this journey truly rewarding wasn't just getting the board up and running, but diving into the world of RISC-V for the first time. Through Yocto porting, I learned how OpenSBI (Open Supervisor Binary Interface) works, its purpose, and its role in the system boot process.

Alongside gaining insight into OpenSBI, here are some highlights from my hands-on experience.

#### Vendor U-Boot & OpenSBI

Noticed that OpenSBI was integrated in-tree with U-Boot, which goes against Yocto's modular approach and complicates independent updates and maintenance. To overcome this, created a patch for Yocto that drops the in-tree OpenSBI structure and adapted the Yocto build to handle OpenSBI and U-Boot as separate recipes, which are combined only as needed for the final image.

#### Kernel Deadlock Debugging

Initially suspected an ABBA deadlock because the kernel appeared to hang after a certain log message. I used lockdep to trace possible lock dependencies, but it turned out there was no real deadlock. The real deadlock was just my own mistake :). In the end, while there wasn't an actual locking issue, this process allowed me to get hands-on experience with lockdep and kernel debugging tools.

#### Vendor Drivers

Analyzed and debugged vendor specific kernel drivers to understand their behavior and address several compatibility problems.

#### License

While working with firmware integration, I researched closed-source licensing scenarios. For open-source firmware, as long as it is released under a recognized open-source license (such as MIT, GPL or Apache), redistribution and integration are generally permitted by default. However, for closed-source firmware (like proprietary blobs), it is crucial to ensure that there is at least an explicit license that clearly allows redistribution. Without such a license, including or sharing closed-source firmware in an open-source project isn't legally possible. This process helped me better understand the legal and compliance requirements for safely integrating firmware components.

---

If you are working on embedded Linux or passionate about open-source, feel free to reach out. I'm always open to new ideas and collaborations.
