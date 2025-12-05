---
layout: post
title:  "Let's Encrypt and VCD"
summary: "Using Let's Encrypt to automate SSL certificates for VCD"
author: samui
date: '2025-12-05'
category: [daily, vcd]
thumbnail: 
keywords: github, jeykyll
permalink: /blog/lets-encrypt-vcd/
usemathjax: true
---

Several years ago, my friend Tomas Fojta wrote a <a href="https://fojta.wordpress.com/2019/12/21/automate-lets-encrypt-certificates-part-2/">blog article</a> on how to automate updating SSL Certificates for VCD. I appreciate his work and have borrowed it for my own project. The project that I have underway is a new VCD 10.6.1.1 lab. We secure it using SSL certificates from Let's Encrypt. Our security team handles the process of getting the new certificate and applying it to the AVI load balancer that fronts VCD for this lab. That certificate is then passed to me for it to be uploaded into the public address configuration within VCD. This last part is where I wanted to borrow Tomas' work and modify it. 

Since the cert is already generated, I only needed to modify Tomas' work for the VCD portion. If viewing his script from the above URL, we are replacing lines 91 to eof with my code below. For this example, the Root, Issuer, and Load Balancer certs have all been copied to the location of the script. In addition, notice that the API header for authorization has changed. And lastly, the GeneralSettings.SystemExternalAddressPublicCertChain
has been deprecated and replaced with GeneralSettings.TenantPortalPublicCertChain. Making these changes to his script will allow the new Let's Encrypt certificates to be uploaded into the VCD public addresses configuration. 


```
##Update vCloud Director with new certificates
##Variables
$Vcd = "<vcd-url>"
$VcdAdmin = "<vcd-system-admin>"
$VcdPassword = "<vcd-system-admin-password>" 

##Cert Info
$RootCert = Get-Content -Path "<let's encrypt root cert>" -Raw
$IssuerCert = Get-Content -Path "<let's encrypt issuer cert>" -Raw
$LBCertificate = Get-Content -Path "<vcd load balancer cert>" -Raw 

##Connect to VCD
$VcdSession = Connect-CIServer $Vcd -User $VcdAdmin -Password $VcdPassword

##API Connection 
$Uri = "https://"+$Vcd+"/api/admin/extension/settings/general"
$head = @{"Authorization"=$VcdSession.SessionSecret} + @{"Accept"="application/*;version=39.1"}

##API Call to fetch the vcd settings
$r = Invoke-WebRequest -URI $Uri -Method Get -Headers $head -ErrorAction:Stop
[xml]$sxml = $r.Content

##Modify the settings
$sxml.GeneralSettings.RestApiBaseUriPublicCertChain = $LBCertificate + $IssuerCert + $RootCert
$sxml.GeneralSettings.TenantPortalPublicCertChain = $LBCertificate + $IssuerCert + $RootCert
 
# API Call to put updated settings back
$r = Invoke-WebRequest -URI $Uri -Method Put -Headers $head -ContentType "application/vnd.vmware.admin.generalSettings+xml" -Body $sxml.OuterXML -ErrorAction:Stop
```
