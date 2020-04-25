---
layout: post
title:  "Do RHEL Containers Inherit Security Compliance from the Host?"
author: shawn
categories: [ work, security, scap, nist, containers, compliance ]
image: assets/images/containers-host-blog-logo.png
tags: featured
---

Recently an OpenShift deployment within the U.S. Department of Defense asked if container images would inherit security compliance settings from their container host. There is documentation — through blogs and vendor content — eluding security control inheritance being possible. Unfortunately, details seem scarce, and content is frequently overly generalized.

Using Red Hat Enterprise Linux 7.6 and the latest configuration guidance from the [NIST National Checklist for Red Hat Enterprise Linux 7.x](https://nvd.nist.gov/ncp/checklist/811), this post:
* Documents security control inheritance between a container host and container image (when using the NIST National Checklist);
* Demonstrates native tooling which attests container configuration;
* Shows how to remediate container configuration drifts using atomic scan

*tl;dr:* There are 363 configuration settings applicable to RHEL 7-based container hosts. **93 controls are not inherited** and applicable to RHEL as a container image. **85 of the 93 controls are resolvable through automation**. The remainder requires manual review.

## Background
While the original conversation was for a U.S. Department of Defense customer, this post uses the [NIST National Checklist for Red Hat Enterprise Linux 7.x](https://nvd.nist.gov/ncp/checklist/811). The national checklist program delivers many compliance baselines — such as *HIPAA* and *FBI Criminal Justice Information Systems (FBI CJIS)* — however, one of them, the ‘*United States Government Configuration Baseline (USGCB)*’ implements a superset of configuration requirements from:

* Committee on National Security Systems Instruction No 1253 (CNSSI 1253)
* NIST Controlled Unclassified Information (NIST 800–171)
* NIST 800–53 control selections for MODERATE impact systems
* NIAP Protection Profile for General Purpose Operating Systems v4.0 (OSPP v4.0)
* DISA Operating System Security Requirements Guide (OS SRG)

This post uses the USGCB baseline to ensure relevance across Civilian, Defense, and Intelligence communities. At the time of writing (March 2019) the current NIST content version is 0.1.43.

NIST publishes the NIST National Checklist content in the [Security Content Automation Protocol (SCAP)](https://csrc.nist.gov/projects/security-content-automation-protocol). The name is slightly misleading — SCAP is not a protocol, such as TCP/IP, but rather a “synthesis of interoperable specifications” from NIST. Essentially a cross-platform automation language that can evaluate for misconfigurations and known vulnerabilities.

An interpreter is needed to parse SCAP content. On Red Hat-based systems, a command line utility called OpenSCAP is delivered natively in RHEL. In 2017, [OpenSCAP was certified by NIST](https://www.redhat.com/en/about/press-releases/red-hat-adds-new-nist-certification-openscap-expands-footprint-open-it-security-standards) as a SCAP configuration and vulnerability scanner.

Using these two pieces — the NIST National Checklist content and OpenSCAP, we can scan container hosts and container images.

## Evaluating a Container Host
Configuration scanning of the container host can be performed with OpenSCAP as shown below (note: on the command line, OpenSCAP is called `oscap`). The `--remediate` flag resolves any failed configuration checks by executing bash-based remediation snippets.

```shell
$ sudo oscap xccdf eval \
--profile xccdf_org.ssgproject.content_profile_ospp \
--oval-results \
--remediate \
--report ~/remediation-report.html \
/usr/share/xml/ssg/content/ssg-rhel7-ds.xml
```

The result was the container host passing 356 configuration checks, failing 4, and having 3 notchecked results for a 99.73% conformance rating. A copy of the full scan report can be found here, with the ‘*Compliance and Scoring*’ section shown below:

![image](/assets/images/inherit-security-container-host.png)


Three of the fails related to not setting up CAC/Smart Card authentication which will are ignored for this quick demo. The fourth failure was caused by not having antivirus installed.

The three `notchecked` results referred to manual evaluations regarding (1) encrypting underlying disk partitions; (2) ensuring software patches are installed; (3) verifying no rogue IPSec tunnel connections.
All in all, the container host is reasonably aligned with how the U.S. Government and U.S. Department of Defense systems are fielded.

## Scanning the Container Image
On Red Hat Enterprise Linux, `atomic scan` is a wrapper utility that enables OpenSCAP to evaluate containers for known CVE vulnerabilities and configuration compliance. Additionally, `atomic scan` can remediate images to a specified policy. [Documentation is available in the RHEL 7 Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sect-using_openscap_with_atomic#sect-Scanning_and_Remediating_Configuration_Compliance_of_Docker_Images_and_Containers_Using_Atomic).

To download the latest RHEL 7 image:
```shell
$ sudo docker pull registry.access.redhat.com/rhel7:latest

Trying to pull repository registry.access.redhat.com/rhel7 ...
latest: Pulling from registry.access.redhat.com/rhel7
2cb1196a3b27: Pull complete
c9c433594a59: Pull complete
Digest: sha256:a77b79c53c1477404e00786ca57b634ab2f0cafdadfb5f0db51452c0354bd298
Status: Downloaded newer image for registry.access.redhat.com/rhel7:latest
```

With `rhel7:latest` pulled locally, you can now evaluate its configuration compliance. Pass/fail information will quickly scroll across the terminal once the `atomic scan` command is run:

```shell
$ sudo atomic scan --scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
registry.access.redhat.com/rhel7:latest

.........

Ensure gpgcheck Enabled for Local Packages
     Severity: Important
       XCCDF result: fail
Ensure gpgcheck Enabled for Repository Metadata
     Severity: Important
       XCCDF result: fail
Ensure Red Hat GPG Key Installed
     Severity: Important
       XCCDF result: fail
Files associated with this scan are in /var/lib/atomic/openscap/2019-03-03-20-36-43-979133.
```

Scan artifacts will be placed into a dynamically generated directory on the container host (based on timestamp). In this case, `/var/lib/atomic/openscap/2019–03–03–20–36–43–979133`.
An HTML report, `report.html`, can be found under that directory. 

## Reviewing the Container's Compliance Report
The review begins again with the ‘Compliance and Scoring’ section. To make comparison easier, the **top image is from the container host and the bottom image is from the `rhel7:latest` image**:

Summary of the container host operating system:

![image](/assets/images/inherit-security-container-host.png)

Summary of the ``rhel7:latest`` container image:

![image](/assets/images/inherit-security-container-itself.png)

While both systems evaluated against the same compliance profile, the container host reflects 363 configuration checks (356 passes + 4 fails + 3 unknowns), whereas `rhel7:latest` has 93 configuration checks (45 passes + 46 fails + 2 unknown).

The number of configuration checks varies because the NIST National Checklist for Red Hat Enterprise Linux 7.x is platform aware. The underlying automation detects if the platform is bare metal, virtual machine, or container image, and adjusts the scans accordingly. **Rules that do not apply to a container image are ignored.**

Consider taking a minute to compare the hardened container host report against the `rhel7:latest` scan report. The report stylesheet, by default, only shows applicable rules. To expand them, or sort the rules by NIST 800–53 or DISA SRG identifiers, use the “Group Rules By” drop-down as shown below:

![image](/assets/images/inherit-security-results-animation.gif)

Only a single configuration check remains applicable once the `rhel7:latest` report is sorted by NIST 800–53 identifiers. This is an example of the security automation content detecting the deployment pattern (bare metal, VM, container) and determining check applicability.

![image](/assets/images/inherit-security-container-ac3.png)

## Remediating the Container
The `rhel7:latest` image passed 45 configuration checks, failed 46, and 2 `notchecked` (indicating manual inspection needed). Before deploying `rhel7:latest` you may want to remediate the image configuration to follow security policy.

`atomic scan` can be used for this purpose. Earlier the following command was used to scan the container images:

```shell
$ sudo atomic scan --scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
registry.access.redhat.com/rhel7:latest
```

To harden an image you’ll need to add the `--remediate` flag.

Every configuration check embeds Bash (and Ansible) snippets. The `--remediate` flag tells `atomic scan` to perform a “dry run” and record all failed results, then for each failed result, execute the associated remediation snippet. Once all remediations are applied a new container image is saved into your local registry. This entire process generally takes less than two minutes.

Add the `--remediate` argument to kickoff remediation:

```shell
 $ sudo atomic scan --remediate \
--scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
registry.access.redhat.com/rhel7:latest
```

Status information will scroll on the screen indicating which rule is currently remediated. Once the process finishes you’ll see output similar to the following:

```shell
Cleaning repos: rhel-7-server-rpms

Other repos take up 2.1 k of disk space (use --verbose for details)
 ---> 39551febfacc

Removing intermediate container 17c2fd176cc7

Successfully built 39551febfacc

Successfully built remediated image 39551febfacc from fddf74700759df45f04a5dc07ed256fd091326455d534761232b04c00507aa1b.
```

## Saving the Remediated Container
A new container image, `39551febfacc`, was created from `rhel7:latest`. This image should be present when running docker images:

```shell
$ sudo docker images \
--format "table {{.Repository}}\t{{.Tag}}\t{{.ID}}"
REPOSITORY          TAG           IMAGE ID
<none>              <none>        39551febfacc
```

You can then tag this image and give it a human-readable name:

```shell
$ sudo docker tag 39551febfacc rhel7-hardened:latest
```

## Performing a Final Scan
As a final step, you may wish to quickly validate the `rhel7-hardened:latest` image against the NIST National Checklist. To do so, re-run `atomic scan` and change the last argument from `registry.access.redhat.com/rhel7:latest` to `rhel7-hardened:latest`:

```shell
$ sudo atomic scan \
--scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
rhel7-hardened-latest
```

The location to the HTML scan report will be output to the screen. To easily follow along, a copy from creating this post is shown below:

![image](/assets/images/inherit-security-container-container-scan-results.png)

In this final evaluation there were 6 failed rules and 2 `notchecked`:

![image](/assets/images/inherit-security-container-container-scan-results-specifics.png)

### These "failures" are expected.
*In regards to the Smart Card/CAC card:*
We’re caught in a situation where government policy hasn’t caught up with technology. Most government agencies, and all of the U.S. Department of Defense, requires “endpoints” to have PIV/CAC card authentication. Consider collaboration with your security auditors to remove these checks from your container baseline.

*In regards to file permission checks:*
If you click on the “Ensure All Files are Owned by a User” link, the specific reason(s) why the rule failed are shown. When creating this blog post, my results looked like this:

![image](/assets/images/inherit-security-container-container-scan-results-fileperms.png)

Inside the container it appears files are owned by UID 0 (root) and GUID 0 (also root). This is currently a false positive, as Linux containers utilize user namespaces to map user IDs. Dan Walsh, who used the lead the SELinux project, [authored an opensource.com article about how this works](https://opensource.com/article/18/3/just-say-no-root-containers).

*In regards to IPSec Tunnels:*
DISA imposes this configuration check and requires manual review of the IPSec configuration files. Their intent is attestation that no cryptographic tunnels configured that are not expected to be present.

*In regards to Software Patches:*
Another DISA imposed configuration check has system administrators and auditors ensure that third party software is appropriately updated. While automation exists for Red Hat-provided software, there’s no known method to verify third party software has all security patches applied.

## In Closing
At this point, you’ve learned how to evaluate and remediate a RHEL-based container host, scan a container image, and remediate any findings. Hopefully, this post balanced readability with thoroughness.

If you’re in the government vertical there are a few resources which may be useful to you:
* [gov-sec@redhat.com Mailing List](https://www.redhat.com/mailman/listinfo/gov-sec)
The gov-sec community mailing list brings together Red Hat customers across the government. It’s a place to ask questions amongst the government user community, share ideas and stories, and keep a pulse of what’s happing.

* Public Workshops
Occasionally Red Hat public sector hosts workshops around Container Security and DevSecOps. Workshop materials can be found online at https://redhatgov.io. Ask your Red Hat reps about this workshops — they’re generally free and scheduled across North America.

* ComplianceAsCode Project
The ComplianceAsCode project at https://github.com/ComplianceAsCode/content is the open source and upstream community of Red Hat’s NIST National Checklists. If you’re interested in developing baselines or enhancing them, the ComplianceAsCode community is a great place to start!