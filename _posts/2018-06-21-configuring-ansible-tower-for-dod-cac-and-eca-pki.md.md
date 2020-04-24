---
layout: post
title:  "Configuring Ansible Tower for DoD CAC and ECA PKI"
author: shawn
categories: [ work, ansible, how-to ]
image: assets/images/ansible-tower-cac.jpeg
tags: [sticky]
---

For this writeup we'll configure Ansible Tower to require DoD PKI or ECA PKI certificates for authentication. Many thanks to my colleagues [Stuart Bain](https://www.linkedin.com/in/stbain/) and [Jamie Duncan](https://www.linkedin.com/in/jamieeduncan/) for pointers on how to get all this set up!


## Step 1: Download DoD Certification Authority (CA) Certificates
The DoD Root CAs can be [downloaded directly from DISA](). DISA offers bundled Zip files for DoD, ECA, JITC, and SIPR PKI. These certificates are updated (roughly) annually, so the links below may not work after January 2019. *Visit DISA's website directly for the latest download URLs!*

First, download the DoD PKI bundle:

`````bash
$ wget -O \
/tmp/Certificates_PKCS7_v5.0u1_DoD.zip \
http://iasecontent.disa.mil/pki-pke/Certificates_PKCS7_v5.0u1_DoD.zip
`````

Then download the ECA PKI bundle:
`````bash
$ wget -O \
/tmp/Certificates_PKCS7_v5.0.1_ECA.zip \
http://iasecontent.disa.mil/pki-pke/Certificates_PKCS7_v5.0.1_ECA.zip
`````

And unzip them into ``/tmp``:
`````bash
$ unzip /tmp/Certificates_PKCS7_v5.0.1_ECA.zip -d /tmp/
$ unzip /tmp/Certificates_PKCS7_v5.0u1_DoD.zip -d /tmp/
`````

## Step 2: Export CA certificates to PEM files
The Ansible Tower UI is powered by an embedded NGINX server. The DISA certificates need to be converted into a format that includes just the public certificate, of which NGINX will then use to validate client certificates.

Use OpenSSL to perform the conversion:
`````bash
$ openssl pkcs7 -in /tmp/Certificates_PKCS7_v5.0u1_DoD/Certificates_PKCS7_v5.0u1_DoD.pem.p7b \
-print_certs -out /tmp/DoD_PKI_CAs.pem

$ openssl pkcs7 -in /tmp/Certificates_PKCS7_v5.0.1_ECA/Certificates_PKCS7_v5.0.1_ECA.pem.p7b \
-print_certs -out /tmp/DoD_ECA_CAs.pem
`````

To accept both DoD PKI and DoD ECA certificates, concatenate the two PEMs:
`````bash
$ cat /tmp/DoD_PKI_CAs.pem /tmp/DoD_ECA_CAs.pem \
> /etc/ssl/certs/DoD_CAs.pem’
`````

And cleanup the temp files:
`````bash
$ rm -Rf /tmp/Certificates_PKCS7_v5.0* /tmp/DoD_*.pem
`````

## Step 3: Configure NGINX, restart Ansible Tower
Ansible Tower deploys the NGINX configuration file to ``/etc/nginx/nginx.conf``.

The server stanza must be modified to tell NGINX to recognize the DoD CAs for client certificates. Add the text below to ``/etc/nginx/nginx.conf``:

`````bash
http {
    . . .
    server {
    . . .
    # Add Support for DoD CAC/ECA Certs
    ssl_verify_depth 2;
    ssl_client_certificate /etc/ssl/certs/DoD_CAs.pem;
`````

Restart Ansible Tower for changes to take effect:
`````bash
$ sudo ansible-tower-service restart
`````

## Step 4: Test!
Attempt to access your Ansible Tower instance through a non-CAC/ECA enabled browser. Ansible Tower should return a 400 error:

![image](/assets/images/ansible-tower-cac-error.png)


With a CAC or ECA-enabled browser, Ansible will first prompt for the user certificate:

![image](/assets/images/ansible-tower-user-certificate.png)

Once a valid client certificate is presented, Ansible Tower will expose the standard logon screen:

![image](/assets/images/ansible-tower-logon-screen.png)

Congrats -- you're done!

## Continuing the Conversation
Consider joining the [gov-sec@redhat.com mailing list](https://www.redhat.com/mailman/listinfo/gov-sec). It’s a moderated forum where those in Government IT can ask questions about Red Hat products. The list includes people from across DoD, Intelligence, State, Civilian, and Red Hat engineering. While not a formal support mechanism, gov-sec is a great place to bounce questions and ideas to others using Red Hat in Government!