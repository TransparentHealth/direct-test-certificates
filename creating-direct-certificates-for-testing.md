Using DicrectCA.org to Create Certificates for Direct and ETT
=============================================================

**DISCLAIMER:** The Direct Certificate Authority (DirectCA) should not be used in production purposes and is for research and testing purposes only.

**DISCLAIMER:** The DirectCA tool supports CRL. It does not support OSCP at this time.

Purpose and Audience
--------------------

The purpose of this document is to demonstrate how to create Direct certificates for use with a Direct server or NIST's Edge Testing Tool (ETT) using DirtectCA. DirectCA is an [open source](https://github.com/videntity/vcert/) tool, that was created to support NIST, the ONC, and ATLs in creating a variety of certificates for testing purposes.  These include certificates that could be deemed "bad".

Direct CA is a web-based application where users can create certificates for testing purposes.  Unlike in the past, it is no longer necessary to download and setup the Direct Java RI to use the `certGen` tool.  `certGen`, while useful, does not support revocation or CRL/AIA information publication.  For these scenarios, a functioning "certificate authority" is necessary.  DirectCA performs these functions with two online resources. First the [console](https://console.directca.org) is a secured web application where certificates are manipulated (e.g. created, revoked). Public and private certificates can be downloaded here as well. Secondly the [ca](http://ca.directca.org) is a public website where public certificates and related information such as CRLs may be downloaded.

Although an ATL (or anyone) [could install DirectCA locally](https://github.com/videntity/vcert/blob/master/INSTALL.md), it is instead strongly recommended that ATLs use the hosted version. If you are an ATL email sales@videntity.com for a free registration code.

If you are not an ATL but want to use the hosted service. contact Videntity by phone(+1888.871.1017) or email sales@videntity.com.  Videntity offers a 1 week free trial.  After that the fee is $200/mo.


Since Direct CA is [free open source software](https://github.com/videntity/vcert/blob/master/LICENSE.txt), another option is to setup your own server. It could be installed [from source](https://github.com/videntity/vcert) or from a [pre-configured AMI](https://github.com/videntity/vcert/blob/master/INSTALL.md).



Overview of Direct Certificates and Trust Verification
------------------------------------------------------

Certificates used in Direct are [x509 certificates](https://en.wikipedia.org/wiki/X.509). A certificate an is electronic document used to prove ownership of a public key. The certificate includes information about the key, information about its owner's identity, and the digital signature of an entity that has verified the certificate's contents are correct. [Reference](https://en.wikipedia.org/wiki/Public_key_certificate)


Trust Verification in Direct
----------------------------
As a primer, please read section 4, "Trust Verification", in the the [Direct Applicability  Statement](http://wiki.directproject.org/file/view/Applicability%20Statement%20for%20Secure%20Health%20Transport%20v1.1.pdf) 

As you see, there are 5 conditions to satisfy for Trust Verification.

1. Has not expired
2. Has a valid signature
3. Has not been revoked
4. Binding to the expected entity
5. Has a trusted certificate path


Let's briefly explain each of these items with some negative examples.

1. **Has not expired** - An expired certificate is when today is after the expiration date indicated in the certificate.
2. **Has a valid signature** - A certificate with an invalid signature is essentially a invalid/malformed certificate that is likely to throw errors by any software expecting a valid x509 certificate as input. A file containing a random string could be used for negative testing in this case, therefore no special tool is needed to create an example of a malformed certificate.
3. **Has not been revoked** - A revoked certificate is one that has been revoked by the CA or its owner. In order to check for revocation, the certificate must include the URL(s) that identify revoked certificates.  This can be either a URL to a CRL or an OCSP server. One or more of the URLs must be live and contain up to date information about the certificate information. DirectCA publishes CRLs and does not support OCSP at this time.
4. **Binding to the expected entity** - A certificate that is bound to the wrong entity is when that which is fetched (expected) does not match the contents of the returned certificate. In Direct, what is "expected" is the value of the DNS or LDAP certificate query.  Specifically, the `SubjectAltName` should match. Keep in mind that the @ symbol is converted to '.' when querying for an email-bound certificates in DNS. For example, querying `bob@example.com` in DNS is formatted as `bob.example.com`, and the resulting certificate should contain an `SubjectAltNAme` of `email: bob@example.com`. If querying for `bob.example.com` returned a `SubjectAltName` of `email: alice@example.com`, then Bob's certificate is bound to the wrong entitiy, Alice's public certificate.
5. **Has a trusted certificate path** - A certificate without a trusted certificate path (also referred to as a "chain") cannot be validated to a root certificate.[Reference](https://en.wikipedia.org/wiki/Certification_path_validation_algorithm). In Direct, however, the path is only validated up to the Trust Anchor which is usually not a root certificate authority. Let's use the following scenario as a setup for broken chain examples.  The Direct endpoint certificate `alice@direct.example.com` has an intermediate anchor of `intermediate.example.com` and a trust anchor of `example.com`. The full path would look like this: _alice@direct.example.com-->intermediate.example.com --> example.com->Root CA._ The certificate path is untrusted if:
    * `alice@direct.example.com` is missing the AIA (i.e. URL pointer to) `intermediate.example.com` and the `intermediate.example.com` is not otherwise available (such as stored locally).
    * The `AuthorityKeyIdentifier` in `alice@direct.example.com` does not match the `SubjectKeyIdentifier` of `intermediate.example.com`. This is an "I am NOT your father" / potential imposter situation.
    * `alice@direct.example.com` contains the AIA (i.e. URL pointer to) `intermediate.example.com` but the URL (or list of URLs) do not yield the `intermediate.example.com` certificate. For example, the AIA server could be offline.
    * `alice@direct.example.com` contains the AIA (i.e. URL pointer to) `intermediate.example.com` and the pointer does yield the certificate, but when checking `intermediate.example.com`'s AIA for `example.com`, it is either missing or offline.


Conditions Supported by DirectCA
--------------------------------

The following table outlines Trust Verification Tests made possible with DirectCA.


    Conditions                          Supported  Certificate to Create       Test Type
    ==========                          =========  =====================       =========
    1. Has not expired                  Y          Expired                     Negative
    2. Has a valid signature            Y          Good                        Positive
    3. Has not been revoked             Y+         Good then revoke            Negative
    4. Binding to the expected entity   Y++        Good w/ misconfig DNS/LDAP  Negative
    5. Has a trusted certificate path   Y+++       Missing AIA+++              Negative

\+ **Revocation**: *Revocation is only supported via via CRL.*

++ **Binding to the expected entity**: *Installing the incorrect certificate in DNS or LDAP is a way to test for this. For example, when we query for Alice's certificate, we get a certificate for Bob instead. Please note that the DNS server bundled with Java Direct RI does not allow this misconfiguration since it inspects the certificate to determine the value placed in DNS.  Use BIND or another DNS server for this sort of test.



+++ **Trusted Certificate Path**: AIA could be missing from the endpoint certificate itself or any intermediate certificate between the endpoint and the Trust Anchor.*The certificate chain is verified all the
way up to the Trust Anchor. (Version 1.1 of the Direct Applicability Statement
for Secure Health Transport Working does not require checking of the chain back
to the Root CA, but rather just to the Trust Anchor.



Creating Certificates
=====================

Depending on what you are doing, you may need to create some or all of the certificates listed here. One recommended practice is to use one domain-bound certificate and one email-bound certificates for "happy paths" and create email-bound certificates on the same domain for negative test cases.  For example, you could create "direct.example.com" and "happy@direct.example.com" as a "happy certificates" and then create individual negative certificates such as "revoked@direct.example.com", "expired@direct.example.com", "missing-aia@direct.example.com", missing-crl@direct.example.com", etc.




The domain "example.com" used is just for illustration. Use your own domains.

Good Certificates
------------------

Good certificates will test your "happy path". Creating the certificates as described below creates one intermediate "hop" between the trust anchor and the endpoint. This makes testing certificate path possible. 

   1. A Top Level trust anchor, we will name "example.com". You will need the file "example.com.der".
   2. An intermediate anchor built from the aforementioned "example.com", for example
   "intermediate.example.com".
   3. A domain-bound endpoint certificate "direct.example.com" built from the aforementioned "intermediate.example.com".  You will need the file "direct.example.com.p12" and  possibly "direct.example.com.der".
   4. An email-bound endpoint certificate "happ@direct.example.com" built from the aforementioned "intermediate.example.com".  You will need the file "happ@direct.example.com.p12" and possibly "happy@direct.example.com.der".

Creating Good Certificates with DirectCA
-----------------------------------------

1. **Login** to the [console](https://console.directca.org)

2. **Create the Top-Level Trust Anchor:** Click the Create Top Level Anchor Button and complete the form with your hostname. Ensure the DNS and email address both have the value of your top-level trust anchor ("example.com" in the above explanation.). Click submit and then verify on the next screen.

3.**Create an Intermediate Anchor:** Now that a Top-level trust anchor exists, an entry will exist under the Root CA.  Click on Create an Intermediate Anchor under "example.com". Complete the form with your desired hostname. As before, ensure the DNS and email address fields both contain your desired hostname value ("intermediate.example.com" in the above explanation).

3.**Create a Domain-Bound Endpoint:** Now that an intermediate certificate exists you can fan out the tree to see the intermediate anchor below the top-level anchor.  Under the intermediate anchor you just created, click "Create Endpoint". Complete the form with your desired hostname. As before, ensure the DNS and email address fields both contain your desired hostname value ("direct.example.com" in the above explanation).

3.**Create an Email-Bound Endpoint:** Fan out the tree to see the intermediate anchor below the top-level anchor.  Under the intermediate anchor you just created, click "Create Endpoint". Complete the form with your desired email. Ensure the DNS and email address fields both contain your desired email value ("happy@direct.example.com" in the above explanation).



Check out this how-to video.



"Bad" Certificates for Negative Testing
---------------------------------------

The following two sections outline creating so called "bad" certificates where 
information might be incorrect or missing, or invalid

**Tip:** It is recommended that most negative tests be attached to email-bound 
certificates because it makes the testing process easier to manage because 
it avoids a lot of revocation and re-issuing for the same domain. Although the instructions below are for email-bound endpoints the steps are identical for anchors or domain-bound endpoints.

Note that when performing a negative test for "bound to the expected entity" use a good certificate because because the bad part lies in the misconfiguration in DNS or LDAP.


Creating Bad Certificates with DirectCA

Here is an list of negative certificates test examples covered in the next sections.

* Expired
* Revoked
* Missing AIA
* Missing CRL


Create an Expired Endpoint
--------------------------

This step is the same as *Create and Email-Bound Endpoint* except use a name such as `no-crl@direct.example.com` and set he expire days to 1 day. the certificate will be "expired" the next day.



Create a Revoked Endpoint
-------------------------


This step is the same as *Create an Email-Bound Endpoint* step except use a something like `revoked@direct.example.com`.
After downloading the needed certificates, click on the Revoke button.  You can re-fresh the CRL instantly by clicking recreate CRL.  Please be sure to download the revoked certificate first, otherwise you will no longer have access to it.


Create an Endpoint with missing AIA
-----------------------------------

This step is the same as *Create and Email-Bound Endpoint* except use a name such as `no-aia@direct.example.com` and un-check the "Include AIA" box.



Other Resources
===============

Direct Certificate Authority Console Source Code
------------------------------------------------

A Django-powered Certificate Authority This is what runs Directca.org.

https://github.com/videntity/vcert



GetDC - Get Direct Certificate
------------------------------

A Python library and command line tool for fetching and parsing Direct certificates

The source codee can be found at https://github.com/videntity/getdc

Can be installed with `pip`.


django-direct
--------------
A reusable django app that provides a RESTFul API for fetching parsing
certificates. Depends on `getdc`.

https://github.com/videntity/django-direct

Can be installed with `pip`,



Licenses
--------
The software packages mentioned above copyright of Videntity Syetare open source under the GPLv2 license.
Commercial licenses are available. Contact sales@videntity.com for more information. 


  (c)Videntity Systems, Inc. 2013-2016.






