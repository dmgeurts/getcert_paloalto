# getcert_paloalto
Automate FreeIPA certificate deployments and renewals for Palo Alto Networks Firewalls.

For a detailed guide, please refer to the following: TBD

CREDIT - The basis for this repository was taken from: https://github.com/psiri/letsencrypt_paloalto


# FreeIPA Certificates for Palo Alto Firewalls

While free LetsEncrypt certificates are great, when you have an internal CA then this opens the door for using your own certificates. As long as you guard your root CA private keys well, issuing your own certificates is no less secure than using LetsEncrypt. This code came about as I had the following requirements:
- Automate certificate renewal (both LetsEncrypt and FreeIPA fit the bill).
- User-ID based rules using FreeIPA as IAM/IDP.
  - All clients are FreeIPA enrolled, thus they automatically have the FreeIPA root CA installed.
  - Use Global Protect agents for Users.

Not using LetsEncrypt avoids problems with permitting ingress HTTP-01 traffic or setting up DNS-01 verification, both have their uses and I use both, but I have no need to present a trusted certificate to anyone or anything without my FreeIPA root CA installed.

The following will detail how to automate the ipa-getcert certificate process for network and security devices. While the focus is on Palo Alto Networks firewalls for the purpose of this demonstration, the following can be adapted to work with almost any network device, and is particularly useful for VPN headend devices (regardless of make and model).


# Prerequisites

### Before getting started, the following are required:
+ A FreeIPA server with Dogtag CA.

+ A linux-based container, VM, or host with reachability to your device’s management interface and the following packages installed:
  + Host enrolled onto FreeIPA server
    + ipa-getcert (part of ipa-client-install)
  + pan-python (https://pypi.org/project/pan-python/)
    + _**Note:** pan-os-python is a Palo Alto managed Python library, but doesn't cater to managing certificates AFAIK. (https://github.com/PaloAltoNetworks/pan-os-python)_
  ```
  # Based on Ubuntu 22.04
  sudo apt install python3-pip
  sudo pip install pan-python
  ```

+ A Service Principal for the firewall:
  + Manually create the host with the FQDN of the firewall management interface.
    + Edit the host to add the enrolled host under 'managed by'.
  + Create a Service Principal for the firewall management interface: `HTTP/fqdn@DOMAIN.COM`
    + Edit it Service Principal to add the enrolled host under 'managed by'.

+ An API key for the firewall:
  + Use the following command to generate an API key:
  ```
  panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k
  ```
+ It's recommended to store the API key in a securely stored file (.panrc):
  + As a good practice, avoid storing your API key directly in your script:
  ```
  panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k >> ~/.panrc
  chmod 600 ~/.panrc
  ```


# The Script
The script is located in the file "pan_getcert" --> https://github.com/dmgeurts/getcert_paloalto/edit/master/pan_getcert


# How It Works
### This small bash script performs the following actions:
1. Uses the privileges set in FreeIPA (managed by) to call ipa-getcert and request a certificate from FreeIPA.
2. ipa-getcert will automatically renew a certificate when it's due, s long as the FQDN DNS record resolves, and the host and Service Principal still exist in FreeIPA.
3. Randomly generates a certificate passphrase using “openssl rand”.
4. Creates a temporary, password-protected PKCS12 cert file named “getcert_pkcs12.pfx” from the individual private and public keys issued by ipa-getcert.
5. Uploads the temporary PKCS12 file to the firewall using the randomly-generated passphrase. Certificate name is set to variable $CERT_NAME.
6. Deletes the temporary PKCS12 certificate from linux host.
7. (Optionally) Sets the certificate used within the GlobalProtect Portal’s SSL/TLS profile to the name of the new LetsEncrypt certificate.
8. (Optionally) Sets the certificate used within the GlobalProtect Gateway’s SSL/TLS profile to the name of the new LetsEncrypt certificate.
9. Commits the candidate configuration (synchronously) and reports for the commit result.


# Automated Renewal and Installation
ipa-getcert can be given a command to execute on renewal, this is used to upload renewed certificates rather than using cron.
The command will upload the renewed certificates, modify the SSL/TLS Service Profiles (if required), and commit the configuration.

To make this a bit more adaptable to different scenarios, I have included the following variables:
+ *CERT_NAME*: The name you wish to give the certificate on the device (Palo Alto Networks GUI:  Device –> Certificate Management –> Certificates)
+ *GP_PORTAL_TLS_PROFILE*: The name of the GlobalProtect SSL/TLS Service Profile used on the Portal.
+ *GP_GW_TLS_PROFILE*: The name of the GlobalProtect SSL/TLS Service Profile used on the Gateway. For single Portal/Gateway deployments using a single SSL/TLS profile, this may be the same as “GP_PORTAL_TLS_PROFILE”.

### Notes
+ As best-practice, you should use separate SSL/TLS Service Profiles for each Portal and Gateway. This script assumes you have followed best-practices, but will also work with single-profile configurations.
+ With Palo Alto Networks Firewalls specifically, updating the SSL/TLS Service Profiles is only required when the name of the certificate referenced by the SSL/TLS Service Profile changes.
+ If the SSL/TLS Service Profiles have been updated with the name of the FreeIPA certificate (and you do not plan on timestamping or otherwise changing the certificate name), no modifications are required when a certificate is renewed. If you choose to append a timestamp or rename the certificate, you will need to programmatically update the SSL/TLS Service profiles.
  + To update the SSL/TLS Service Profile(s), uncomment lines 20 (for a single SSL/TLS Profile) and 22 (for two SSL/TLS Profiles).
  ```
  panxapi.py -h $PAN_MGMT -K $API_KEY -S "$CERT_NAME" "/config/shared/ssl-tls-service-profile/entry[@name='$GP_PORTAL_TLS_PROFILE']"
  panxapi.py -h $PAN_MGMT -K $API_KEY -S "$CERT_NAME" "/config/shared/ssl-tls-service-profile/entry[@name='$GP_GW_TLS_PROFILE']"
  ```
  + **If your certificate name does not change, you can safely leave these commands enabled if desired.**


# Prepare the Script for Execution
+ In an environment with centralised credentials it's best to not run code under a user that may be removed. Thus install the code to a common location like: `/usr/local/bin`, which is the assumed location.
  + Ensure the pan_getcert is executable
    ```
    sudo chmod +x /usr/local/bin/pan_getcert
    ```
+ Similarly the `.panrc` file is best placed under etc, for example: `/etc/ipa/.panrc`, which is the assumed location.
+ Ensure there are no extensions after the filename. In most cases, only underscores and dashes are supported


# Creating a Cronjob for Fully-Automated Certificate Installation
### Note: This article is written for Ubuntu

1. Create a custom cronjob to execute the certificate renewal at your desired interval:
  ```
  sudo vi /etc/cron.d/pan_certbot_cron
  ```
2. Insert the following within the /etc/cron.d/pan_certbot file to ensure the cron generates a log file to debug any issues:
  + The following cronjob will run twice weekly (Midnight Weds and Saturday).  To change execution frequency, modify the schedule to meet your needs.
  + The log file will be timestamped and stored in /var/log/pan_certbot.log
  + **Change “/home/psiri/pan_certbot” to the full path of the script**
  ```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash

0 0 * * 3,6 root (/bin/date && /home/psiri/pan_certbot) >> /var/log/pan_certbot.log 2>&1
```

# Validate Your Runs [TBC]
```
cat /var/log/pan_getcert.log
```
If successful, you should see something similar to the following:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Renewing an existing certificate
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/bitbodyguard.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/bitbodyguard.com/privkey.pem
   Your cert will expire on 2020-01-21. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411     34   1204  0:00:03  0:00:03 --:--:--  1204
<response status="success"><result>Successfully imported LetsEncryptWildcard into candidate configuration</result></response> 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    103   3667  0:00:01  0:00:01 --:--:--  3666
<response status="success"><result>Successfully imported LetsEncryptWildcard into candidate configuration</result></response> 
set: success [code="20"]: "command succeeded"
set: success [code="20"]: "command succeeded"
commit: success: "Configuration committed successfully"
```

If the cron was successful, but you have it a rate-limit, you will typically see the following:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Renewing an existing certificate
An unexpected error occurred:
There were too many requests of a given type :: Error creating new order :: too many certificates already issued for exact set of domains: *.bitbodyguard.com: see https://letsencrypt.org/docs/rate-limits/
Please see the logfiles in /var/log/letsencrypt for more details.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    101   3582  0:00:01  0:00:01 --:--:--  3583
Successfully imported LetsEncryptWildcard into candidate configuration 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    160   5678 --:--:-- --:--:-- --:--:--  5676
Successfully imported LetsEncryptWildcard into candidate configuration 
set: success [code="20"]: "command succeeded"
set: success [code="20"]: "command succeeded"
commit: success: "Configuration committed successfully"
```

You should also be able to check the following locations on the Palo Alto Networks firewall for additional confirmation:
**Monitor –> Logs –> Configuration**
You should see 3-5 operations, depending on whether or not you chose to modify the SSL/TLS service profile(s).  In my setup (PA-850), it takes 7-8 seconds to renew, upload, and commit the configuration (not including actual commit time):

1. A web upload to /config/shared/certificate
2. A web upload to /config/shared/certificate/entry[@name=’LetsEncryptWildcard’]
3. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_PORTAL_PROFILE’]
4. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_EXT_GW_PROFILE’]
5. And a web “commit” operation
