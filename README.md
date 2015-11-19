Using the Direct Certificate Authority Console for Generating Test Certificates (DRAFT)
======================================================================================


The Direct Certificate Authority Console can be used to create
certificates with a number of variations including what might be
called good and bad certificates..

The priimary motiovation for developing this software was create test
certificates for the purpose of testing Direct. Direct is an email-based
health information exhange standard protocol that uses email protected with
x509 certificates. See
http://wiki.directproject.org/file/view/Applicability+Statement+for+Secure+Health+Transport+v1.2.pdf
 for more information.

Both "good" and "bad" certificates with chains are needed to 
fully test Trust Verification. for example eching for: for items such as:

* Revocation
* Missing AIA Information
* Binding to the expected (when a certificate is pulled from DNS or LDAP)


The following instrustions outline how to create the certificates. 
After they are created, you can download them from the website and install them in your 
Direct system you are testing.



Logon
-----

The URL to logon to the console is: https://console.directca.org, The root CA page, which also hosts the the AIA and CRL information is http://ca.directca.org.


Create a Top Level Anchor
=========================

This is the first step and may only need to be done once.

Click the "Create a Top Level Anchor" and fill in the form. The DNS entry is the most important.
Enter a valid DNS name for your anchor. This is one and the same with the common name (CN).
For example, "example.com". Please note you can omit CRL or AIA information at any point in the chain by unchecking their checkboxes.
When creating a "good" certificate chain, leave the defauls checked.  Click Save at the bottom of the form.


If you have self-verify permission you can verify the certificate creation request.
Otherwise, you must wait for an administrator to verify it for you. After you recive notifcaion the 
anchor is active, refresh the screen to see the new anchor for download and further menu options.
Click on its name, such as "example.com" to see the full submenu. From here, intermediate anchors and endpoints can be created. The file you most likey need to is the public ".der" file.


Create an Intermediate Anchor
=============================

This is an optional step but necessary to check AIA chain verification.

Click the button "Create Intermediate Anchor" and follow the steps from above.


Create a Domain-Bound Endpoint
==============================

Click the button "Create Endpoint certificate" and
complete the form.  A domain v/s an email-bound certificate is determined by an "@"" sign in the
"Email or Domain" and "DNS" fields. Note these two fields should match exactly when making a
"good" certificate.

To create a certificate for "direct.example.com" enter that value for both fields.


Click Continue to generate the certificate.
After your certificaste is verified, refesh to see the enpoint certificate menu.
The file you most likely need is the ".p12" file whch contains both the pulic and private
certificates in one file.



Create and Email-Bound Endpoint
===============================


Same as above but swap `direct.example.com` for `john@direct.example.com`


Creating Bad Certificates
=========================

The following sections outline creating so called "bad" certificates where information might be missing.

Note: Tt is reccomended that most negative tests be attached to email-bound 
certificates because it makes the testing process easier to manage because 
it avoids a lot of revocation and re-issuing.

Although the instrustions below are for email-bound endpoints the steps are identical for anchors or domain-bound enpoints.


Create a revoked Endpoint.
=========================


Same as *Create and Email-Bound Endpoint* step but use a something like `revoked@direct.example.com`.
After downloawding the needed certificates, click on the Revoke button.  You can re-fresh the CRL instantly by clicking recreate CRL.


Create an Endpoint without AIA Info.
====================================

Same as *Create and Email-Bound Endpoint*  but use a name such as `no-aia@direct.example.com` and uncheck the"Include AIA" box.


Create an Endpoint without CRL Info.
====================================


Same as *Create and Email-Bound Endpoint* but use a name such as `no-crl@direct.example.com` and uncheck the "Include CRL" box.

Create an Expired Endpoint
===========================

Same as *Create and Email-Bound Endpoint* but use a name such as `no-crl@direct.example.com` and  set he expire days to 1 day. the certificate will be "expired" the next day.



Source Code Resources:
=====================

Direct Certificate authority Console
------------------------------------

Django project power the tool described here.

https://github.com/videntity/vcert


GetDC
-----
A Python library and command line tool for fetching parsing certificates


https://github.com/videntity/getdc

Can be installed with `pip`.


django-direct
--------------
A reusable django app that provides a RESTFul API for fetching parsing
certificates. Depends on `getdc`.



https://github.com/videntity/django-direct

Can be installed with `pip`,




All software is open source under the GPLv2 licese.  Other licenses available.



