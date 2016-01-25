Using DicrectCA.org to Create Certificates for Direct and ETT
=============================================================


Purpose and Audience
--------------------

The purpose of this document is to demonstrate how to create Direct certificates for use with a Direct server or NIST's Edge Testing Tool (ETT) using DirtectCA. DirectCA is an [open source](https://github.com/videntity/vcert/) tool, that was created to support NIST, the ONC, and ATLs in creating a variety of certificates for testing purposes.  These include certificates that could be deemed "bad".

Direct CA is a web-based application where users can create certificates for testing purposes.  Unlike in the past, it is no longer necessary to download and setup the Direct Java RI to use the `certGen` tool.  `certGen`, while useful, does not support revocation or CRL/AIA information publication.  For these scenerios, a functioning "certificate authority" is necessary.  DirectCA performs these functions with two online resources. First the [console](https://console.directca.org) is a secured web application where certificates are manipulated (e.g. created, revoked). Public and private certificates can be downloaded here as well. Secondly the [ca](http://ca.directca.org) is a public website where public certificates and related information such as CRLs may be downloaded.

Although an ATL (or anyone) [could install DirectCA locally](https://github.com/videntity/vcert/blob/master/INSTALL.md), it is instead strongly reccomended that ATLs use the hosted version. If you are an ATL email sales@videntity.com for a free registration code.

If you are not an ATL but want to use the hosted service. contact Videntity by phone(+1888.871.1017) or email sales@videntity.com.  Videntity offers a 1 week free trial.  After that the fee is $200/mo.


Since Direct CA is [free open source software](https://github.com/videntity/vcert/blob/master/LICENSE.txt), another option is to setup your own server. It could be installed [from source](https://github.com/videntity/vcert) or from a [pre-configured AMI](https://github.com/videntity/vcert/blob/master/INSTALL.md).



Overview of Direct Certificates and Trust Verification
------------------------------------------------------

Certificates used in Direct are [x509 certificates](https://en.wikipedia.org/wiki/X.509). A certificate an electronic document used to prove ownership of a public key. The certificate includes information about the key, information about its owner's identity, and the digital signature of an entity that has verified the certificate's contents are correct. [Reference](https://en.wikipedia.org/wiki/Public_key_certificate)


Trust Verification in Direct
----------------------------
As a primer, please read section 4 "Trust Verification" in

http://wiki.directproject.org/file/view/Applicability%20Statement%20for%20Secure%20Health%20Transport%20v1.1.pdf

As you see, there are 5 conditions to satisfy for Trust Verification.

1. Has not expired
2. Has a valid signature
3. Has not been revoked
4. Binding to the expected entity
5. Has a trusted certificate path


Let's briefly explain each of these items with some negative examples.

1. **Has not expired** - An expired certificate is when today is after the expiration date indicated in the certificate.
2. **Has a valid signature** - A certificate with an invalid signature is essentially a invalid/malformed certificate that is likely to throw errors by any software expecting a valid x509 certificate as input. A file containing a random string could be used for negative testing in this case, therefore no special tool is needed to create an example of a malformed certificate.
3. **Has not been revoked** - A revoked certificate is one that has been revoked by the CA or its owner. In order to check for revocation, the certificate must include the URL(s) that identify revoked certificates.  This can be either a URL to a CRL or an OCSP server. One or more of the URLs must be live and contain up to date information about the certificate information. DirectCA pusblishes CRLs and does not support OCSP at this time.
4. **Binding to the expected entity** - A certificate that is bound to the wrong entitiy is when that which is fetched (expected) does not match the contents of the returned certificate. In Direct, what is "expected" is the value of the DNS or LDAP certificate query.  Specificaly, the `SubjectAltName` should match. Keep in mind that the @ symbol is converted to '.' when quering for an email-bound certificates in DNS. For example, quering `bob@example.com` in DNS is formatted as `bob.example.com`, and the resulting certificate should containg an `SubjectAltNAme` of `email: bob@example.com`. If quering for `bob.example.com` returned a `SubjectAltName` of `email: alice@example.com`, then Bob's certificate is bound to the wrong entitiy, Alice's public certificate.
5. **Has a trusted certificate path** - A certificate without a trusted certificate path (also refered to as a "chain") cannot be validated to a root certificate.[Reference](https://en.wikipedia.org/wiki/Certification_path_validation_algorithm). In Direct, however, the path is only validated up to the Trust Anchor which is usually not a root certificate authority. Let's use the following scenerio as a setup for broken chain exaples.  The Direct endpoint certificate `alice@direct.example.com` has an intermediate anchor of `intermediate.example.com` and a trust anchor of `example.com`.The full path would look like this: _alice@direct.example.com-->intermediate.example.com --> example.com->Root CA._ The certificate path is untrusted if:
    * `alice@direct.example.com` is missing the AIA (i.e. URL pointer to) `intermediate.example.com` and the `intermediate.example.com` is not otherwise avialable (such as stored locally).
    * The `AuthorityKeyIdentifier` in `alice@direct.example.com` does not match the `SubjectKeyIdentifier` of `intermediate.example.com`. This is an "I am NOT your father" / potential imposter situation.
    * `alice@direct.example.com` contains the AIA (i.e. URL pointer to) `intermediate.example.com` but the URL (or list of URLs) do not yield the `intermediate.example.com` certificate. For example, the AIA server could be offline.
    * `alice@direct.example.com` contains the AIA (i.e. URL pointer to) `intermediate.example.com` and the pointer does yield the certificate, but when checking `intermediate.example.com`'s AIA for `example.com`, it is either missing or offline.


Conditions Supported by DirectCA
--------------------------------

The following table outlines Trust Verification Tersts made possible with DirectCA.


    Conditions                          Supported  Certificate to Create       Test Type
    ==========                          =========  =====================       =========
    1. Has not expired                  Y          Expired                     Negative
    2. Has a valid signature            Y          Good                        Positive
    3. Has not been revoked             Y+         Good then revoke            Negative
    4. Binding to the expected entity   Y++        Good w/ misconfig DNS/LDAP  Negative
    5. Has a trusted certificate path   Y+++       Missing AIA+++              Negative

\+ **Revocation**: *Revocation is only supported via via CRL.*

++ **Binding to the expected entity**: *Installing the incorrect certificate in DNS or LDAP is a way to test for this. For example, when we query for Alice's certificate, we get a certificate for Bob instead. PLease note that the DNS server bundled with Java Direct RI does not allow this misconfigurration since it inspect the certificate to determine the value placed in DNS.  Use BIND or another DNS server for this sort of test.



+++ **Trusted Certificate Path**: AIA could be missing from the endpoint certificate itself or any intermediate certificate between the endpoint and the Trust Anchor.*The certificate chain is verified all the
way up to the Trust Anchor. ( Version 1.1 of the Direct Applicability Statement
for Secure Health Transport Working does not require checking of the chain back
to the Root CA, but rather just to the Trust Anchor.



Creating Certificates
=====================

Depending on what you are doing, you may need to create some or all of the certificates listed here. One reccomended practice is to use one domain-bound certificate and one email-bound certificates for "happy paths" and create email-bound certificates on the same domain for negative test cases.  For example you could create "direct.example.com" and "happy@direct.example.com" as a "happy certificates" and then create individual negative certificates such as "revoked@direct.example.com", "expired@direct.example.com", "missing-aia@direct.example.com", missing-crl@direct.example.com", etc.




The domain "example.com" used is just for illustration. Use your own domains.

Good Certificates
------------------

Good certificates will test your "happy path". Creating the certificates as described below creatse one intermedioate "hop" between the trust anchor and the endpoint. This makes testing certificate path possible. 

   1. A Top Level trust anchor, we will name "example.com". You will need the file "example.com.der".
   2. An intermediate anchor built from the aforementioned "example.com", for example
   "intermediate.example.com".
   3. A domain-bound endpoint certificate "direct.example.com" built from the aformentioned "intermediate.example.com".  You will need the file "direct.example.com.p12" and  possibly "direct.example.com.der".
   4. An email-bound endpoint certificate "happ@direct.example.com" built from the aformentioned "intermediate.example.com".  You will need the file "happ@direct.example.com.p12" and  possibly "happy@direct.example.com.der".

Good Certificates in DirectCA
---------------------------------

1. **Login** to the [console](https://console.directca.org)

2. **Create the Top-Level Trust Anchor:** Click the Create Top Level Anchor Button and complete the form with your hostname. Ensure the DNS and email address both have the value of your top-level trust anchor ("example.com" in the above explanation.). Click submit and then verfy on the next screen.

3.**Create an Intermediate Anchor:** Now that a Top-level trust anchor exists, an entry will exist under the Root CA.  Click on Create an Intermediate Anchor under "example.com". Complete the form with your desired hostname. As before, ensure the DNS and email address fields both contain your desired hostname value ("intermediate.example.com" in the above explanation).

3.**Create a Domain-Bound Endpoint:** Now that an intermediate certificate exists you can fan out the tree to see the intermediate anchor below the top-level anchor.  Under the intermediate anchor you just created, click "Create Endpoint". Complete the form with your desired hostname. As before, ensure the DNS and email address fields both contain your desired hostname value ("direct.example.com" in the above explanation).

3.**Create an Email-Bound Endpoint:** Fan out the tree to see the intermediate anchor below the top-level anchor.  Under the intermediate anchor you just created, click "Create Endpoint". Complete the form with your desired email. Ensure the DNS and email address fields both contain your desired email value ("happy@direct.example.com" in the above explanation).





"Not so Good" Certificates for Negative Testing
-----------------------------------------------

Alertative certificates can be created for the purposes of negative testing.  





   1. An expired certificate.
   2. 
   3. An invalid trust relationship. We will call this
   "invalid-trust-relationship" in these instructions. This anchor is valid,
   but the children node were not created by this trust anchor.Use this in conjunction with
   "ttt.your-domain.com" from #2.  These node Certificates then it would be
   invalid because the domain-bound certificate was not created with this trust
   anchor.

Other Negative Tests
--------------------

ATLs are currently not testing 3, 4, and are partially testing 5.


3. Revocation -  The Direct Applicbility Statement version 1.1 does not specify
how revocation is to be performed.  The Java RI implements certificate revocation
lists (CRL), but other Direct implementation do it some other way.The web based
certificate generation tool, https://DirectCA.org  , provides experimental
support for this test, but this is not officially supported.

4. Binding to the expected entity - It is not possible for the "certGen" tool
to generate a certificate with The subjectAltName extension is present, a
dNSName is included, and it DOES NOT matches the Direct Address' Health
Internet Domain.  Other tools attempt to prevent this mistake as well.
The web based certificate generation tool, https://DirectCA.org  , provides
experimental support for this test, but this is not officially supported.

5. The test procedure needs to be updated to require the building of a multi-node
chain (2 nodes or more away from the Trust Anchor).


Running the Canned version of CertGen using VMWare
==================================================

1. Download the file [http://certgen.s3.amazonaws.com/certGen-Ubuntu-12-LTS-64bit.zip]
(http://certgen.s3.amazonaws.com/certGen-Ubuntu-12-LTS-64bit.zip)
2. Unzip the file "certGen-Ubuntu-12-LTS-64bit.zip" to a folder such as "certgen-vm".
3. Download VMWare Player from [http://www.vmware.com/products/player/]
(http://www.vmware.com/products/player/)
4. Start VMWare Player.
5. Press Ctrl-O to open a VMWare Image
6. Navigate to folder extracted in step 1 and select the file
"Direct certGen - Ubuntu64-bit.vmx" and click "Open".
7. When the virtual VM is completely booted you will see a Desktop.
There is no password or login required, but the user and password are:
ubuntu/adm1nD1r3ct
8. Press Ctrl-Alt-T to open up a terminal window.
9. Start the certGen tool.

Type this into the terminal:

```
    cd direct/tools
    ./certGen.sh
```
Details for creating a each type of certificate is detailed  in the following
sections.


Create a Good Root CA / Trust Anchor (# 2)
==========================================

This certificate is for testing "2. Has a valid signature ".  It is used in
conjunction with the domain-bound endpoint certificate decribed in the next
section of this document.

1. Enter the appropriate values for the CA you are creating in the certGen tool
following the example shown in the figure below. You only need to create one CA.

```
    CN:                             [Name for your CA - ex. "example.com"]
    Country:                        [Your Country] # Use two letter ISO code, e.g. US.
    State:                          [Your State]
    Location:                       [Your City]
    Org:                            [Your Organization Name]
    Email:                          [example.com]
    Expiration Days:                365
    Key Strength:                   1024
    Password:                       [Your password]
    Add Email to Alt Subject Names: Checked
```
![Screen shot of certGen used to create a root certificate authority.]
(http://certgen.s3.amazonaws.com/CA1.png "Create a Root CA")


2. Click the Create button to create the CA. This will create the files
"example.com.der" and "example.comKey.der" in the /home/ubuntu/direct/tools directory.
If you need to create more leaf certificates from this CA later, You will need
these two files along with your password to reload the CA.



Create a Good Domain Bound Certificate (#2)
===========================================

This certificate is for testing "2. Has a valid signature ".  It is used in
conjunction with the trust anchor certificate decribed in the previous
section of this document.


1. After the CA is created (or loaded), click "Create Leaf Cert" button. Enter
the required values. Be sure to click the “Add Email to Alt Subject Names”.
DO NOT add a password.

```
    CN:                                 [ttt.example.com]
    Country:                            [Your Country] # Use two letter ISO code, e.g. US.
    State:                              [Your State]
    Location:                           [Your City]
    Org:                                [Your Organization Name]
    Email:                              [ttt.example.com] #Notice this is not actually an email.
    Expiration Days:                    365
    Key Strength:                       1024
    Password:                           LEAVE BLANK
    Add Email to Alt Subject Names:     Checked
```


![Screen shot of certGen used to create a domain-bound certificate]
(http://certgen.s3.amazonaws.com/domain-bound-cert.png
"Create a Domain-Bound Certificate")


2. Click the "Create" button.This will create the files "ttt.example.com.der",
and "ttt.example.com.p12" in the /home/ubuntu/direct/tools/ directory.

3. Copy the trust anchor and domain-bound endpoint certificates to files to
their own directory called "good".  This is necessary because the next steps will
overwrite these files because they will have the same file names.
```
    mkdir good
    mv example.com* good
    mv ttt.example.com* good
```


Expired (#2)
============


1. Using the Good Trust Anchor from above, create an expired domain-bound
 endpoint certificate.

```
    CN:                             [Name for your CA - ex. "ttt.example.com"]
    Country:                        [Your Country] # Use two letter ISO code, e.g. US.
    State:                          [Your State]
    Location:                       [Your City]
    Org:                            [Your Organization Name]
    Email:                          [ttt.example.com]
    Expiration Days:                0
    Key Strength:                   1024
    Password:                       LEAVE BLANK
    Add Email to Alt Subject Names: Checked
```
![Screen shot of certGen used to create an expired domain-bound certificate]
(http://certgen.s3.amazonaws.com/expired.png
"Create an Expired Domain-Bound Certificate")


2. Copy this domain-bound endpoint's certificates to their own directory
called "expired".  This is necessary because the next steps will
overwrite these files because they will have the same file names.

```
    mkdir expired
    mv ttt.example* expired
```


Create Another Root CA / Trust Anchor to be Used in the Invalid Trust Relationship Test (#5)
============================================================================================

1. Restart certGen.sh.

2. Enter the appropriate values for the CA you are creating in the certGen tool
following the example shown in the figure below. You only need to create one CA.

```
    CN:                             [Name for your CA - ex. "example.com"]
    Country:                        [Your Country] # Use two letter ISO code, e.g. US.
    State:                          [Your State]
    Location:                       [Your City]
    Org:                            [Your Organization Name]
    Email:                          [example.com]
    Expiration Days:                365
    Key Strength:                   1024
    Password:                       [Your password]
    Add Email to Alt Subject Names: Checked
```
![Screen shot of certGen used to create a root certificate authority.]
(http://certgen.s3.amazonaws.com/CA2.png
"Create a Root CA")


3. Click the Create button to create the CA. This will create the files
"example.com.der" and "example.comKey.der" in the
/home/ubuntu/direct/tools directory.  Place these in their own directory called,
invalid-trust-relationship" to avoid confusing this with your other sets of certificates.

```
    mkdir invalid-trust-relationship
    cp example.com* invalid-trust-relationship
    cp good/ttt.example.com* invalid-trust-relationship
```
The above step will result in a folder containing a trust anchor and endpoint that
do not match.


4. Install this example.com.der into the Direct implmentation.  Then attempt to
send a Direct message using the certificate from the "Good Domain Bound Certificate"
step.  Since this endpoint certificate was not created by this trust anchor, despite
the fact the DNS, Common Name (CN), and Email, and Subject are the same. 

Please refer to ttt-configuration.md for more explanation on installation of these
certificates.
