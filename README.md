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

![diagram](./img/getcert_paloalto%20interaction.png)

<sup>\*</sup>) _The Linux host doesn't need to be Ubuntu, I should have used a Tux logo instead..._

### Before getting started, the following are required:
+ A FreeIPA server with Dogtag CA.

+ A linux-based container, VM, or host with reachability to your device’s management interface and the following packages installed:
  + ipa-getcert - part of ipa-client-install on a FreeIPA enrolled host.
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
  + When using centralised authentication, consider whether the API user should be a user or dedicated API account.
    + Using a user account will mean the API calls stop working when the user is removed!
  + `/etc/ipa/.panrc` is the assumed location:
  ```
  panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k | sudo tee /etc/ipa/.panrc
  sudo chmod 600 /etc/ipa/.panrc
  ```


# The Scripts
There are two scripts:
+ "pan_getcert" --> https://github.com/dmgeurts/getcert_paloalto/edit/master/pan_getcert
  + Obtains a certificate from FreeIPA and sets the post-save to use pan_instcert for automatic installation of a renewed certificate.
+ "pan_instcert" --> https://github.com/dmgeurts/getcert_paloalto/edit/master/pan_instcert
  + Installs and optionally attaches a certificate to an SSL/TLS Profile.
    + Management interface SSL/TLS Profile
    + Global Protect certificates can be attached to one or two SSL/TLS Profiles, as required.

# How It Works
### pan_getcert performs the following actions:
1. Uses the privileges set in FreeIPA (managed by) to call ipa-getcert and request a certificate from FreeIPA.
2. ipa-getcert will automatically renew a certificate when it's due, s long as the FQDN DNS record resolves, and the host and Service Principal still exist in FreeIPA.
3. Calls pan_instcert to get the certificate installed.
4. Sets port-renewal to run pan_instcert for automated installation of a renewed certificate.

### pan_instcert performs the following actions:
1. Randomly generates a certificate passphrase using “openssl rand”.
2. Creates a temporary, password-protected PKCS12 cert file named “getcert_pkcs12.pfx” from the individual private and public keys issued by ipa-getcert, and places this file in /etc/opa/ssl. This folder is created if it doesn't exist.
3. Uploads the temporary PKCS12 file to the firewall using the randomly-generated passphrase.
  + (Optionally) adds a year (in 4-digit notation) to the certificate name.
4. Deletes the temporary PKCS12 certificate from the linux host.
5. (Optionally) Sets the certificate used within the SSL/TLS profile to the name of the new LetsEncrypt certificate.
  + The SSL/TLS profile name can be parsed.
  + Or the GlobalProtect Portal and GlobalProtect Gateway SSL/TLS profiles can set.
6. Commits the candidate configuration (synchronously) and reports for the commit result.

### Script options
The scripts use the following options:

```
Usage: ${0##*/} [-hv] -c CERT_CN [-n CERT_NAME] [-Y] [OPTIONS] FQDN
This script requests a certificate from FreeIPA using ipa-getcert and calls a partner
script to deploy the certificate to a Palo Alto firewall or Panorama.
    FQDN                Fully qualified name of the Palo Alto firewall or Panorama
                        interface. Must be reachable from this host on port TCP/443.
    -c CERT_CN          REQUIRED. Common Name of the certificate (must be an FQDN). Will
                        be added as to the certificate as a SAN.
OPTIONS:
    -n CERT_NAME        Name of the certificate in PanOS configuration. Defaults to the
                        certificate Common Name.
    -s PROFILE_NAME     Apply the certificate to a SSL/TLS Service Profile.
                        (For example for a management interface).
    -p [PROFILE_NAME]   Apply the certificate to a GlobalProtect SSL/TLS Service Profile
                        used on the Portal. Omit this option to avoid automatic linking.
                        Default profile name: GP_PORTAL_PROFILE
    -g [PROFILE_NAME]   Apply the certificate to a GlobalProtect SSL/TLS Service Profile
                        used on the Gateway. Omit this option to avoid automatic linking.
                        Default proifile name: GP_EXT_GW_PROFILE
    -Y                  Append the current year '_YYYY' to the certificate name. Also
                        when the certificate is automatically renewed by ipa-getcert.
    -h                  Display this help and exit.
    -v                  Verbose mode.
```

**Note:** `-Y` and -c are unique to pan_getcert.

+ *CERT_NAME*: The name you wish to give the certificate on the device (Palo Alto Networks GUI:  Device –> Certificate Management –> Certificates)
+ **-p** _GP_PORTAL_TLS_PROFILE_: The name of the GlobalProtect SSL/TLS Service Profile used on the Portal.
+ **-g** _GP_GW_TLS_PROFILE_: The name of the GlobalProtect SSL/TLS Service Profile used on the Gateway. For single Portal/Gateway deployments using a single SSL/TLS profile, this may be the same as “GP_PORTAL_TLS_PROFILE”.


# Automated Renewal and Installation
ipa-getcert requests the certificate with the post-save option so that no cronjob is needed to install renewed certificates.
The post-save will upload the renewed certificates, and take the same actions against SSL/TLS Service Profiles as when the certificate was initially installed. If the -Y option is given the certificate name will be appended with the year ("_YYYY"), otherwise the certificate will be overwritten in the Palo Alto forewall or Panorama configuration.


### Notes
+ As best-practice, one should use separate SSL/TLS Service Profiles for each Portal and Gateway. These scripts assume best-practices are followed, but will also work with single-profile configurations.
+ With Palo Alto Networks Firewalls specifically, updating the SSL/TLS Service Profiles is only required when the name of the certificate referenced by the SSL/TLS Service Profile changes.
  + pan_instcert won't know if it's being called to install or update a certificate. Hence the SSL/TLS profile is always updated if parsed.


# Prepare the Script for Execution
+ In an environment with centralised credentials it's best to not run code under a user that may be removed. Install the code to a common location: `/usr/local/bin` is assumed location. When using a different location, change variable INST_CERT in pan_getcert accordingly.
  + Install and ensure pan_getcert and pan_instcert are executable:
    ```
    sudo cp getcert_paloalto/pan_{get,inst}cert /usr/local/bin/
    sudo chmod +x /usr/local/bin/pan_{get,inst}cert
    ```
+ The file `.panrc` is best placed under etc. `/etc/ipa/.panrc` is assumed, and is configurable in pan_getcert.


# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Your Runs [TBC]
`cat /var/log/pan_getcert.log`

If successful, you should see something similar to the following:
```
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

Also check the following locations on the Palo Alto Networks firewall for additional confirmation:
**Monitor –> Logs –> Configuration**
There should be 3-5 operations shown, depending on whether or not the SSL/TLS service profile(s) are being updated.

1. A web upload to /config/shared/certificate.
2. A web upload to /config/shared/certificate/entry[@name=’FQDN’] under the FreeIPA root CA certificate.
3. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_PORTAL_PROFILE’]
4. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_EXT_GW_PROFILE’]
5. And a web “commit” operation.
