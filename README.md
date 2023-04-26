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
  + `/etc/ipa/.panrc` is the assumed location.
  + The following format is detailed at https://github.com/kevinsteves/pan-python/blob/master/doc/panrc.rst
  ```
  api_username=api-admin
  api_password=strong-password
  hostname=192.168.1.1
  
  # Generic API key
  api_key=C2M1P2h1tDEz8zF3SwhF2dWC1gzzhnE1qU39EmHtGZM=
  ```
  However, the location of this file is assumed to be `~/.panrc` by panxapi.py. There is a request to add an option to specify the location of this file: https://github.com/kevinsteves/pan-python/issues/53
  + To create `.panrc`:
    + First create an API user on the Palo Alto
    + Use a strong password for this user
    + Generate the API key with:
    `panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k`
    + Create the .panrc file, and enter the api_key as follows.
    `sudo vi /etc/ipa/.panrc`
    At a bare minimum it should look like:
    ```
    api_key=C2M1P2h1tDEz8zF3SwhF2dWC1gzzhnE1qU39EmHtGZM=
    ```
    + Secure the file:
    `sudo chmod 600 /etc/ipa/.panrc`


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
    -c CERT_CN          REQUIRED. Common Name (Subject) of the certificate (must be a
                        FQDN). Will also present in the certificate as a SAN.
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

+ **-n** _CERT_NAME_: The name you wish to give the certificate on the device (Palo Alto Networks GUI:  Device –> Certificate Management –> Certificates)
+ **-p** _GP_PORTAL_TLS_PROFILE_: The name of the GlobalProtect SSL/TLS Service Profile used on the Portal.
+ **-g** _GP_GW_TLS_PROFILE_: The name of the GlobalProtect SSL/TLS Service Profile used on the Gateway. For single Portal/Gateway deployments using a single SSL/TLS profile, this may be the same as “GP_PORTAL_TLS_PROFILE”.


# Automated Renewal and Installation
ipa-getcert requests the certificate with the post-save option so that no cronjob is needed to install renewed certificates.
The post-save will upload the renewed certificates, and take the same actions against SSL/TLS Service Profiles as when the certificate was initially installed. If the -Y option is given the certificate name will be appended with the year ("_YYYY"), otherwise the certificate will be overwritten in the Palo Alto forewall or Panorama configuration.

Check the configured post-save command with:
```
sudo ipa-getcert list [-i Request_ID | -f Path_to_cert_file]
```


### Notes
+ As best-practice, one should use separate SSL/TLS Service Profiles for each Portal and Gateway. These scripts assume best-practices are followed, but will also work with single-profile configurations.
+ With Palo Alto Networks Firewalls specifically, updating the SSL/TLS Service Profiles is only required when the name of the certificate referenced by the SSL/TLS Service Profile changes.
  + pan_instcert won't know if it's being called to install or update a certificate. Hence the SSL/TLS profile is always updated if parsed.


# Prepare the Script for Execution
+ In an environment with centralised credentials it's best to not run code under a user that may be removed. Install the code to a common location: `/usr/local/bin` is assumed location. When using a different location, change variable INST_CERT in pan_getcert accordingly.
  + To configure an API user on a Palo Alto firewall:
    + Add a local administrator (Device >> Administrators)
      + Authentication Profile: None
      + Set a strong password (only used to obtain the API key)
      + Administrator Type: Role Based
      + Profile: Create a new profile with the folowing rights (Device >> Admin Roles):
        + Web UI - Disable everything
        + XML API - Enable only:
          + Configuration
          + Operational Requests
          + Commit
          + Import
        + Command Line - None
        + REST API - Disable everything
+ Install and ensure pan_getcert and pan_instcert are executable:
  ```
  sudo cp getcert_paloalto/pan_{get,inst}cert /usr/local/bin/
  sudo chmod +x /usr/local/bin/pan_{get,inst}cert
  ```
+ Ensure `/etc/ipa/.panrc` exists, see above.


# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Automated Renewals
pan_instcert will log to `/var/log/pan_instcert.log`.

pan_getcert doesn't log as it's expected to be run once (manually).
```
cat /var/log/pan_instcert.log
```

If successful, you should see something similar to the following:
```
[2023-04-26 14:13:53+00:00]: START of pan_instcert.
Certificate Common Name: test.domain.local
Parsed certificate Name: test.domain.local_2023
  verbose=1
  PAN_FQDN: fw01.domain.local
SSL/TLS profile SSL_TLS_PROFILE: Tst_profile
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3342  100   121  100  3221    118   3146  0:00:01  0:00:01 --:--:--  3266
<response status="success"><result>Successfully imported test.mm.eu_2025 into candidate configuration</result></response>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3342  100   121  100  3221    118   3145  0:00:01  0:00:01 --:--:--  3266
<response status="success"><result>Successfully imported test.mm.eu_2025 into candidate configuration</result></response>
Finished uploading the certificate.
set: success [code="20"]: "command succeeded"
test.domain.local_2023 has been set on SSL/TLS profile: Test_profile
commit: success: "Configuration committed successfully"
[2023-04-26 14:14:09+00:00]: END - Certificate installation done to: fw01.domain.local.
```

If the SSL/TLS Service Profile doesn't exist it will be created, but the following error will be shown and the commit will fail:
```
commit: success: "Validation Error:
 ssl-tls-service-profile -> Test_profile  is missing 'protocol-settings'
 ssl-tls-service-profile is invalid"
```

Also check the following locations on the Palo Alto Networks firewall for additional confirmation:
**Monitor –> Logs –> Configuration**
There should be 3-5 operations shown, depending on whether or not the SSL/TLS service profile(s) are being updated.

1. A web upload to /config/shared/certificate.
2. A web upload to /config/shared/certificate/entry[@name=’FQDN(\_YYYY)’] under the FreeIPA root CA certificate.
3. One or more web “set” commands to /config/shared/ssl-tls-service-profile/entry[@name=’YOUR_PROFILE(S)’]
4. And a web “commit” operation.
