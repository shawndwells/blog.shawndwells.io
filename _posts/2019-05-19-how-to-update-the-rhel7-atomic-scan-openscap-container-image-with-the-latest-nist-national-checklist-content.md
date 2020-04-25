---
layout: post
title:  "How to Update the RHEL 7 Atomic Scan / OpenSCAP Container Image with the Latest NIST National Checklist Content"
author: shawn
categories: [ work, scap, how-to ]
image: assets/images/nist-ncp-logo.png
tags: featured
---
Red Hat delivers configuration assessment content natively in Red Hat Enterprise Linux. Unfortunately, this content is generally updated every 4–6 months, causing the RHEL-provided content to be several months behind official baselines like the NIST National Checklist. This article steps through updating Red Hat’s native Atomic Scan tooling to use the latest [NIST National Checklist Program](https://nvd.nist.gov/ncp/repository) content.

## Step 1: Identify Desired Content

Identify which baseline(s) you would like to incorporate into your custom scanner. For this blog we’ll use the [NIST National Checklist for Red Hat Enterprise Linux 7.x](https://nvd.nist.gov/ncp/checklist/811).

The NIST National Checklist catalog for Red Hat technologies does cover more than RHEL 7. A full listing is available from NIST:

[https://nvd.nist.gov/ncp/repository?authority=Red+Hat&startIndex=0](https://nvd.nist.gov/ncp/repository?authority=Red+Hat&startIndex=0)


Direct links to commonly saught content include:
* [NIST National Checklist for Red Hat Enterprise Linux 7.x](https://nvd.nist.gov/ncp/checklist/811)
* [NIST National Checklist for Red Hat Enterprise Linux 8.x](https://nvd.nist.gov/ncp/checklist/909)
* [NIST National Checklist for Red Hat OpenShift Container Platform 3.x](https://nvd.nist.gov/ncp/checklist/866)
* [NIST National Checklist for Red Hat Virtualization Host 4.x](https://nvd.nist.gov/ncp/checklist/908)

To ensure interoperability with the widest range of configuration scanners, Red Hat provides content in both the SCAP 1.2 and SCAP 1.3 specifications. The primary difference being SCAP 1.3 added support for evaluating systemd configuration settings. Because of this, *the SCAP 1.3 data stream is recommended for hosts based on RHEL 7 or RHEL 8.*

## Step 2: Login to the Red Hat Container Registry and Download Latest OpenSCAP Container

Because you will build a custom container image, which pulls from the Red Hat Container Registry, ensure you're logged in:

```shell
$ docker login registry.redhat.io

Username: << your username >>
Password: << your password >>
Login Succeeded!
```

## Step 3: Create and Build Customer Container Image
Create a temporary directory to house your Dockerfile. This could be under `/tmp`, or even your user directory.
Because this file is meant to be ephemeral, `/tmp/openscap-ncp/` is used for this blog.

```shell
$ mkdir /tmp/openscap-ncp
```

Now create your Dockerfile. At the time of publication, SCAP Security Guide v0.1.44 was the latest available and is reflected in the template below.
Place this text into your chosen directory (e.g. `/tmp/openscap-ncp/Dockerfile`). Replace URLs and file names as needed:

```shell
FROM registry.redhat.io/rhel7/openscap
MAINTAINER Your Name (YourEMail@domain.com)
#
# Download updated NIST National Checklist
#
# NOTE: This URL will need to be updated as future versions
# are released! NIST NCP content for Red Hat is kept at
# https://nvd.nist.gov/ncp/repository?authority=Red+Hat&startIndex=0
RUN wget https://github.com/ComplianceAsCode/content/releases/download/v0.1.44/scap-security-guide-0.1.44-redhat-SCAP-1.3.zip -P /tmp/
RUN unzip /tmp/scap-security-guide-0.1.44-redhat-SCAP-1.3.zip -d /tmp/
#
# Unpack the content into /usr/share/xml/scap/ssg/content/
#
# Technically this could be any directory as defined by the
# 'ssg' variable in the [Content] section of /etc/oscapd/config.ini
# on the container host.
#
RUN cp /tmp/scap-security-guide-0.1.44/ssg-* /usr/share/xml/scap/ssg/content/
#
# Cleanup the downloaded zip file and extracted
# directory.
#
RUN rm -Rf /tmp/scap-security-guide-0.1.44{/,-redhat-SCAP-1.3.zip}
```

Next, docker build the container image:
```shell
$ sudo docker build -t openscap-ncp:v0.1.44 /tmp/openscap-ncp/
.....
.....
.....
Successfully built b5285a42068f
```

Make special note of your container ID. In the example above, the resultant container was `b5285a42068f`.
Some deployments are accustomed to using the `:latest` tag. Should that be the case, alias your new image to `:latest` by issuing the docker tag command:

```shell
$ sudo docker tag <<Your Image ID>> openscap-ncp:latest
```

For example:
```shell
$ sudo docker tag b5285a42068f openscap-ncp:latest
```

## Step 4: Update Atomic Scan to use "openscap-ncp" Container Image
Configuration files under `/etc/atomic.d/` ensure Atomic Scan can modularly support multiple scanning engines. These config files define the engine (such as name, and what commands to run) and also provide variables, APIs, or CLI arguments as needed.

The Dockerfile used in Step 2 downloaded the standard OpenSCAP container image, inserted the updated NIST National Checklist content into it, and saved the resultant image in your local registry. At this point, you now configure Atomic Scan to use this new image.

The following commands take the default Atomic Scan configuration file for OpenSCAP, copies it to a new scanner called `openscap-ncp`, and tell this scanner to use the `openscap-ncp:latest` container image when performing scans.

```shell
$ sudo cp /etc/atomic.d/openscap /etc/atomic.d/openscap-ncp

$ sudo sed -i 's/^scanner_name: .*$/scanner_name: openscap-ncp/' /etc/atomic.d/openscap-ncp

$ sudo sed -i 's/^image_name: .*$/image_name: openscap-ncp:latest/' /etc/atomic.d/openscap-ncp
```

## Step 5: Set Default Scanner
If you run the `atomic scan` command, using the syntax provided in the Red Hat documentation, you’ll now get a new error:

```shell
$ sudo atomic scan \
--scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
--verbose \
registry.redhat.io/ubi7/ubi

You must specify a scanner (--scanner) or set a default in /etc/atomic.conf
```

The following command will set the default Atomic Scan engine to `openscap`, which will ensure `atomic scan` invocations will always use the default (RHEL-provided) content for configuration assessments:

```shell
$ sudo sed -i \
's/^default_scanner:.*$/default_scanner: openscap/g' \
/etc/atomic.conf
```

Alternatively, if you’d like to use the updated NIST National Checklist Program content by default, then `openscap-ncp` should be used:

```shell
$ sudo sed -i \
's/^default_scanner:.*$/default_scanner: openscap-ncp/g' \
/etc/atomic.conf
```

System operators can switch between Atomic Scan engines by using the `--scanner` flag.

## Step 6: Test!
The following command runs a configuration assessment of the RHEL 7 UBI image, using the NIST National Checklist for RHEL 7.x:

```shell
$ sudo atomic scan --scanner openscap-ncp \
--scan_type configuration_compliance \
--scanner_args xccdf-id=scap_org.open-scap_cref_ssg-rhel7-xccdf-1.2.xml,profile=xccdf_org.ssgproject.content_profile_ospp,report \
--verbose \
registry.redhat.io/ubi7/ubi
```

The scan will output pass/fail information during the scan. Atomic scan will also dump all scanning data into a dynamically-generated directory, as shown below:

```shell
....

Ensure YUM Removes Previous Package Versions
     Severity: Low
       XCCDF result: fail
Ensure gpgcheck Enabled for Local Packages
     Severity: Important
       XCCDF result: fail
Files associated with this scan are in /var/lib/atomic/openscap-ncp/2019-05-19-18-33-03-177103.
```

Under that directory you will find:
* **arf.xml**: Scan report data generated in the Asset Report Format, or ARF.
* **fix.sh**: Should you opt to remediate the container image, this is the bash script which will be executed.
* **json**: JSON-formatted results of the scan
* **report.html**: HTML formatted results using the OpenSCAP-provided stylesheet.

## More Information
Guidance on performing configuration and known ulnerability (CVE) scans can be found in the [Using OpenSCAP With The Atomic Scan Command](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sect-using_openscap_with_atomic) section of the RHEL 7 Security Guide.
Red Hat’s NIST National Checklist content is developed through the [ComplianceAsCode project on GitHub](https://github.com/ComplianceAsCode/content). A [community mailing list for content development](https://lists.fedorahosted.org/mailman/listinfo/scap-security-guide) and use is also available.