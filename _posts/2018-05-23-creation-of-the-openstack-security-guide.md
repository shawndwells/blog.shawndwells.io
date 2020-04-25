---
layout: post
title:  "Creation of the OpenStack Security Guide"
author: shawn
categories: [ work, security, openstack ]
image: assets/images/openstack-security-guide-cover.png
---
Factoring all moving components of OpenStack, the rapid release cycles, and the sheer complexity of large deployments, OpenStack security information was decentralized and obsoleted every one or two releases (case in point: Nova networking vs Quantum Neutron). To aid the community and provide practical hardening guidance, the [OpenStack Security Group](https://launchpad.net/~openstack-ossg) aspired to create a book that would assist system administrators in hardening their installations. Here’s how we did it.

Through a humbling offer from [Keith Basil](https://twitter.com/noslzzp), I joined the group and extended an invitation to NSA’s [Nate Burton](https://twitter.com/mathrock). Over the course of 5 days we were to write the first edition of an OpenStack Security Guide. From scratch. With many group members never having met before. **And then open source the entire thing!**

Having no idea how to write a book, the group brought in [Adam Hyde](https://twitter.com/booksprint) of FLOSS Manuals to act as a facilitator. This was one of the best decisions; Adam brought years of experience regarding the process of content authoring and digital media. Adam injected an element of engineering rigor into the authorship process, and without him, there’s simply no chance that the book would be authored in the 5-day window.

![image](/assets/images/openstack-book-sprint-wall.jpg)

To put thoughts together, and to assist scoping the book, the first day involved writing down elements we wanted to include on a post-it note, slapped to the wall, and discussed….. I can only imagine how much the hotel staff *loved* us when they walked in and saw their walls decorated like this :)

After the first day over 100 pages were written. By the end of the week, somewhere near 300. The book was assigned primary chapter owners, such as [Ben De Bont](https://twitter.com/bendebont) (Chief Security Officer for HP Cloud Services) handling Compliance.

![image](/assets/images/openstack-book-sprint-table.jpg)

The ability to sit next to [Bryan Payne](https://twitter.com/bdpsecurity) (of Nebula fame), spitball ideas with [Cody Bunch](https://twitter.com/cody_bunch) (Rackspace), and hear implementation stories directly from [Nate Burton](https://twitter.com/mathrock) (NSA) was foundational to our success. Every attendee had unique backgrounds, ranging from developer to consumer, which allowed us to incorporate a wide range of use cases and perspectives into the book.

![image](/assets/images/openstack-security-guide-group.png)

## Where to Download

![image](/assets/images/openstack-security-guide-cover.png)

To download the book, head over to the [OpenStack Foundation’s blog post](http://www.openstack.org/blog/2013/07/openstack-security-guide-now-available/), which includes a [direct link to the ePub](http://aa4698cc2bf4ab7e5907-ed3df21bb39de4e57eec9a20aa0b8711.r41.cf2.rackcdn.com/OpenStackSecurityGuide.epub) file. Over the upcoming weeks the source code will be released as a project on Github, allowing the broad OpenStack community to collaborate and improve the base content. Additionally, you’ll be able to purchase the book off Amazon (profits benefit the OpenStack foundation)!

## What's Next?
The OpenStack Security Guide provides a great starting point for hardening OpenStack installations. Using the themes in the book, combined with input from the NSA and DISA FSO, Red Hat will be submitting for an OpenStack STIG later this month. This work will occur through the [SCAP Security Guide](https://fedorahosted.org/scap-security-guide/), so if you’re interested in such things, [join the mailing list](https://lists.fedorahosted.org/mailman/listinfo/scap-security-guide)!

Tentatively, the OpenStack STIG will be written to a frankenstein mashup of the Application SRG, Operating System SRG, and Network SRG. As with all STIGs, guidance will be written in detail (e.g., what specific variables to change in what specific file).

Watch out for announcements to the SCAP Security Guide mailing list as work progresses on this!

## And in Closing . . .
Cheers to all those who participated, and to the success of the OpenStack Security Guide project! With the pending release of the book's source, it should be an incredible opportunity for the community to build a centralized body of security guidance for future OpenStack releases.

![image](/assets/images/openstack-book-sprint-afterparty.jpg)