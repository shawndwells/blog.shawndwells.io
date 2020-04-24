---
layout: post
title:  "RNGD and Weak Cryptography for Virtual Machines"
author: shawn
categories: [ work, virtualization, security ]
image: assets/images/rusty-lock.jpeg
---

There was recently a great post on [Red Hat’s gov-sec mailing list](https://www.redhat.com/mailman/listinfo/gov-sec) that asked about random number generation inside virtual machines, what constitutes “weak cryptography,” how to increase performance, and generally make things better while staying inside Federal guidelines. Lets take this opportunity to trace through RHEL’s FIPS certifications to make a risk decision on what tools+techniques can speed things up. This conversation is always a rabbit hole. Here’s an approach to breakdown the conversation for security accreditors/auditors.

The original posting:
<blockquote>I keep hearing that enabling the random number generator daemon (rngd) on VMs “introduces weak cryptography”. It’s useful for installs on VMs when deploying things to speed up installs & can obviously then be disabled but I’ve ran into issues often where entropy runs low and application performance suffers.

Anyway, is there any information on exactly how it “introduces weak cryptography” and more specifically if enabling it violates a NIST 800–53r4 control indirectly or directly or is this just an alternative fact?
</blockquote>

To rephrase: *Customer is building virtual machines and running out of entropy. Various tools can speed this up — but what is allowed for Federal systems?*

tl;dr? **Consider rngd.**

## Understanding the Technical Problem
Cryptography relies on the generation of random numbers which are then inserted into various algorithms to protect your data. The term entropy refers to the randomness collected by an operating system.

In Linux, there are four default ways to request random numbers from the kernel:

#### Method #1: [add_device_randomness](https://github.com/torvalds/linux/blob/master/drivers/char/random.c#L978#L989)
Adds data to the random pool that is likely to differ between two devices (or possibly even per boot). This would be things like MAC addresses or serial numbers, or the read-out of the RTC. This does *not* add any actual entropy to the pool, but it initializes the pool to different values for devices that might otherwise be identical and have very little entropy available to them (particularly common in the embedded world).

#### Method #2: [add_input_randomness](https://github.com/torvalds/linux/blob/master/drivers/char/random.c#L1058#L1072)
Uses the input layer interrupt timing, as well as the event type information from the hardware.

#### Method #3: [add_interrupt_randomness](https://github.com/torvalds/linux/blob/master/drivers/char/random.c#L1108#L1172)
Uses the interrupt timing as random inputs to the entropy pool. Using the cycle counters and the irq source as inputs, it feeds the randomness roughly once a second.

#### Method #4: [add_disk_randomness](https://github.com/torvalds/linux/blob/master/drivers/char/random.c#L1174#L1184)
Uses what amounts to the seek time of block layer request events, on a per-disk_devt basis, as input to the entropy pool. Note that high-speed solid state drives with very low seek times do not make for good sources of entropy because their seek times are usually fairly consistent.

If you’ve ever installed Linux and opted to encrypt your hard drive, or attempted to create SSL certificates, you’ve had to move the mouse around or jam on the keyboard. The location of the mouse cursor and the timing between your keystrokes reflects random data.

That’s fine for a single event (encrypt a hard drive during installation). But what happens at scale? What would happen if 100 requests for random numbers came in per second?

During normal system operation, random numbers can be generated from add_risk_randomness (because everything on Linux is a file, so some disk I/O is always happening). If you’re on server or desktop the interrupt timing from your network card could provide some random data.

And is where the problem originates: when virtual machines are created they exhaust their supply of random numbers. How can we speedup the generation of random numbers?

Before jumping into that question we need to understand where the randomness comes from.

## /dev/random vs /dev/urandom
When talking about entropy performance, it’s inevitable that someone brings up how “*/dev/random is blockin*g” and “*/dev/urandom is non-blocking*.” The best article I’ve ever read on how linux generates randomness is “[Myths about /dev/urandom](https://www.2uo.de/myths-about-urandom/),” published by [Thomas Hühn](https://twitter.com/thomashuehn). Thomas provides an excellent technical breakdown of how randomness is gathered.

A choice quote on /dev/random vs /dev/urandom:

<blockquote>Have you ever waited for /dev/random to give you more random numbers? Generating a PGP key inside a virtual machine maybe? Connecting to a web server that’s waiting for more random numbers to create an ephemeral session key?

That’s the problem. It inherently runs counter to availability. So your system is not working. It’s not doing what you built it to do. Obviously, that’s bad. You wouldn’t have built it if you didn’t need it.
…
It’s easy to disregard availability, usability or other nice properties. Security trumps everything, right? So better be inconvenient, unavailable or unusable than feign security.
–[Thomas Hühn](https://twitter.com/thomashuehn)
</blockquote>

If we’re building ephemeral computing architectures and we can’t instantiate containers, our mission is impacted. That is part of a risk management decision: *Do we use the purest random numbers possible, which could impact my service availability, or do we use something “good enough”?*

Personal opinions aside, the original customer question was posted on [Red Hat’s Government Security mailing list](https://www.redhat.com/mailman/listinfo/gov-sec). The audience is inherently federal, and in the Federal realm, information systems are restricted by law to using cryptography that has been FIPS 140–2 evaluated.

For those not familiar, the [Federal Information Processing Standard 140](https://csrc.nist.gov/publications/detail/fips/140/3/final) requires that any cryptography used to “protect sensitive information” be reviewed and validated by the U.S. Government.

So, lets break apart [RHEL7’s FIPS certification package](https://access.redhat.com/articles/2918071#fips-140-2-2) to understand what has been formally reviewed and deemed acceptable to use.

When RHEL7 hosts are in FIPS mode, Deterministic Random Bits Generator (DRBG) is used. Documentation found on page 10 of the [kernel crypto API FIPS paperwork](http://csrc.nist.gov/groups/STM/cmvp/documents/140-1/140sp/140sp2742.pdf). Specifically in section “3.2 Services.”

The paperwork states DRBG is implemented in accordance with NIST 800–90A. Truly interested parties can [muck through the upstream code](https://github.com/torvalds/linux/blob/master/crypto/drbg.c).

Reading a bit further, on page 14 we encounter section “6.1 Random Number Generation.” This is where the paperwork talks nerdy and provides details about how the approved Deterministic Random Bits Generator (DRBG) is implemented. The section starts on page 14, which reads:

`````
The DRBG is initialized during module initialization. The module loads by default the DRBG using HMAC DRBG with SHA-512, with derivation function, without prediction resistance. The DRBG is seeded during initialization with a seed obtained from /dev/urandom of length 3/2 times the DRBG strength. Please note that /dev/urandom is an NDRNG located within the module’s physical boundary but outside its logical boundary.
`````

Readers are then presented with a table that indicates these ``/dev/urandom`` seeds adhere to [NIST 800–90A](https://csrc.nist.gov/publications/detail/sp/800-90a/rev-1/final). That is the “Recommendation for Random Number Generation Using Deterministic Random Bit Generators.” Note this is a *recommendation*, not a *standard*.

As it turns out, RHEL7’s FIPS default is to use ``/dev/urandom``.

All arguments about using ``/dev/random`` vs ``/dev/urandom`` should have been put to bed when evaluating the tradeoff between system availability vs security (use ``/dev/urandom``!). If there were any more concerns, the official RHEL FIPS evaluations use ``/dev/urandom``.

## Potential Solution: rngd
Earlier in this post we mentioned four sources of randomness available in the kernel:

1. **add_device_randomness()**: Data that is likely to differ between two devices. Most often used in seeding random number generators with machine-specific data, like MAC addresses.

2. **add_input_randomness()**: Hand jamming on keyboards to generate random time samples between keystrokes.

3. **add_interrupt_randomness()**: Cycle counters and irq source as inputs, e.g. timing of network packets on server machines.

4. **add_disk_randomness()**: Seek time of block layer requests. Bad idea with SSDs because low seek times make for consistent I/O timings.

All of the above rely on 3rd parties to generate their randomness — keystrokes, interrupts, disk I/O. At some point the system simply can’t keep up with generating random bits, such as when virtual machines are created at scale.

Our question now morphs: *How do we speed up ``/dev/urandom`` (while following NIST recommendations)?*

## Introducing rngd
Modern Intel-based processors have RDRAND instruction sets. These are cryptographically secure pseudorandom number generators in hardware. [Intel provides additional details on their blog](https://software.intel.com/en-us/blogs/2012/11/17/the-difference-between-rdrand-and-rdseed). It should be especially noted RDRAND complies with NIST 800–90A , meaning we stay within the swim lanes of previously acceptable routines.

The Red Hat-provided Random Number Generator Daemon (``rngd`) can tap into this Intel capability to significantly increase the performance of random number generation.

There should be little doubt over the use of rngd. Shipped and supported by the operating system vendor (in this case, Red Hat) and the underlying instructions comply with NIST 800–90A (which is what the default FIPS certified behavior does).

1. Install rngd via the ``rng-tools`` package:
```shell
$ sudo yum -y install rng-tools
```

2. Configure rngd to seed ``/dev/urandom``:
```shell
$ sed -i '/^ExecStart/ s/$/ -r \/dev\/urandom/' \
/usr/lib/systemd/system/rngd.service
```

3. Before enabling anything, run a quick benchmark for non-rngd:

```shell
$ dd if=/dev/urandom of=/dev/null bs=1024 count=1000000 iflag=fullblock

1000000+0 records in
1000000+0 records out
1024000000 bytes (1.0 GB) copied, 73.8034 s, 13.9 MB/s
```

And for we can use ``rngtest`` to review if the randomness of data meets FIPS 140–2 standards:
```shell
$ cat /dev/urandom | rngtest -c 100000

rngtest 5
Copyright (c) 2004 by Henrique de Moraes Holschuh
This is free software; see the source for copying conditions.  There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
rngtest: starting FIPS tests...
rngtest: bits received from input: 2000000032
rngtest: FIPS 140-2 successes: 99915
rngtest: FIPS 140-2 failures: 85
rngtest: FIPS 140-2(2001-10-10) Monobit: 9
rngtest: FIPS 140-2(2001-10-10) Poker: 14
rngtest: FIPS 140-2(2001-10-10) Runs: 32
rngtest: FIPS 140-2(2001-10-10) Long run: 30
rngtest: FIPS 140-2(2001-10-10) Continuous run: 0
rngtest: input channel speed: (min=1.199; avg=175.412; max=19073.486)Mibits/s
rngtest: FIPS tests speed: (min=1.702; avg=162.586; max=186.995)Mibits/s
rngtest: Program run time: 22628591 microseconds
````

4. Enable rngd:
```shell
$ sudo systemctl start rngd

$ sudo systemctl status rngd

● rngd.service - Hardware RNG Entropy Gatherer Daemon
    Loaded: loaded (/usr/lib/systemd/system/rngd.service; enabled; vendor preset: enabled)
    Active: active (running) since Sat 2017-05-13 05:46:40 EDT; 7s ago
 Main PID: 12379 (rngd)
    CGroup: /system.slice/rngd.service
            └─12379 /sbin/rngd -f -r /dev/urandom
 May 13 05:46:40 devbox systemd[1]: Started Hardware RNG Entropy Gatherer Daemon.
 May 13 05:46:40 devbox systemd[1]: Starting Hardware RNG Entropy Gatherer Daemon...
 ```

 5. Re-run the dd test:
 ```shell
$ dd if=/dev/urandom of=/dev/null bs=1024 count=1000000 iflag=fullblock

1000000+0 records in
1000000+0 records out
1024000000 bytes (1.0 GB) copied, 88.9843 s, 11.5 MB/s
```

… roughly a 17% increase. *Not bad!*

And for rngtest:
```shell
$ cat /dev/urandom | rngtest -c 100000
rngtest 5
Copyright (c) 2004 by Henrique de Moraes Holschuh
This is free software; see the source for copying conditions.  There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
rngtest: starting FIPS tests...
rngtest: bits received from input: 2000000032
rngtest: FIPS 140-2 successes: 99918
rngtest: FIPS 140-2 failures: 82
rngtest: FIPS 140-2(2001-10-10) Monobit: 11
rngtest: FIPS 140-2(2001-10-10) Poker: 15
rngtest: FIPS 140-2(2001-10-10) Runs: 33
rngtest: FIPS 140-2(2001-10-10) Long run: 23
rngtest: FIPS 140-2(2001-10-10) Continuous run: 0
rngtest: input channel speed:(min=1.150; avg=170.846; max=19073.486)Mibits/s
rngtest: FIPS tests speed: (min=2.642; avg=163.223; max=185.179)Mibits/s
rngtest: Program run time: 22873454 microseconds
```

Mostly the same.

6. If you like what you see, make the change persistent:

```shell
$ sudo systemctl enable rngd.service
```

### Time for a Risk Decision
In default configuration your system might not generate enough random numbers, which may affect system availability. Consider installing ``rng-utils`` to take advantage of the cryptographic functions in your hardware to speed things up. The tooling is provided natively in RHEL, and is supported by Red Hat and Intel Corp. 

rngd also happens to follow the same NIST recommendations on random number generation as the default subsystem. In quick testing on a generic x86 server this gave a ~17% performance boost. Your metrics may vary -- but this is a fairly significant increase for such a minor change.