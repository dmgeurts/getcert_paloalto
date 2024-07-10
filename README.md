# getcert_paloalto
Automate FreeIPA certificate deployments and renewals for Palo Alto Networks Firewalls.

See also: https://djerk.nl/2023/automating-freeipa-certificates-on-palo-alto-devices

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
+ "pan_getcert" --> https://github.com/dmgeurts/getcert_paloalto/blob/master/pan_getcert
  + Obtains a certificate from FreeIPA and sets the post-save to use pan_instcert for automatic installation of a renewed certificate.
+ "pan_instcert" --> https://github.com/dmgeurts/getcert_paloalto/blob/master/pan_instcert
  + Installs and optionally attaches the certificate to up to two SSL/TLS Profiles.
    + Management interface SSL/TLS Profile
    + Global Protect certificates can be attached to one or two SSL/TLS Profiles, as required.

# How It Works
### pan_getcert performs the following actions:
1. Uses the privileges set in FreeIPA (managed by) to call ipa-getcert and request a certificate from FreeIPA.
2. ipa-getcert will automatically renew a certificate when it's due, as long as the FQDN DNS record resolves, and the host and Service Principal still exist in FreeIPA.
3. Sets the post-save command to pan_instcert with the same parameters as issued to pan_getcert, for automated installation of renewed certificates.
    - Post-save will run on the first certificate save, using pan_instcert for certificate installation.

### pan_instcert performs the following actions:
1. Randomly generates a certificate passphrase using “openssl rand”.
2. Creates a temporary, password-protected PKCS12 cert file `/tmp/getcert_pkcs12.pfx` from the individual private and public keys issued by ipa-getcert.
3. Uploads the temporary PKCS12 file to the firewall using the randomly-generated passphrase.
    + (Optionally) adds a year (in 4-digit notation) to the certificate name.
4. Deletes the temporary PKCS12 certificate from the Linux host.
5. (Optionally) applies the certificate to up to two SSL/TLS Profiles.
    + Single SSL/TLS Profile: For example for the Management UI SSL/TLS profile.
    + Two SSL/TLS Profiles: For example for GlobalProtect Portal and GlobalProtect Gateway SSL/TLS Profiles.
6. Commits the candidate configuration (synchronously) and reports the commit result.
7. Logs all output to `/var/log/pan_instcert.log`.

### Script options
Run pan_getcert without sudo, doing so may block access to your keytab and then ipa-getcert can't authenticate to IPA. It will test for and run ipa-getcert with sudo privileges.

pan_getcert uses the following options:
```
Usage: pan_getcert [-hv] -c CERT_CN [-n CERT_NAME] [-Y] [OPTIONS] FQDN
This script requests a certificate from FreeIPA using ipa-getcert and calls a partner
script to deploy the certificate to a Palo Alto firewall or Panorama.

    FQDN              Fully qualified name of the Palo Alto firewall or Panorama
                      interface. Must be reachable from this host on port TCP/443.
    -c CERT_CN        REQUIRED. Common Name (Subject) of the certificate (must be a
                      FQDN). Will also present in the certificate as a SAN.

OPTIONS:
    -n CERT_NAME      Name of the certificate in PanOS configuration. Defaults to the
                      certificate Common Name.
    -Y                Parsed to pan_instcert to append the current year '_YYYY' to
                      the certificate name.
    -S SERVICE        Service of the Service Principal. Default: HTTP.
    -T CERT_PROFILE   Ask IPA to process the request using the named profile or
                      template.
    -G TYPE           Type of key to be generated if one is not already in place.
                      IPA CS uses RSA by default. (RSA, DSA, EC or ECDSA)
    -b BITS           In case a new key pair needs to be generated, this option
                      specifies the size of the key. Default: 2048 (RSA/DSA).
                      EC: 256, 384 or 512. See certmonger.conf for the default.

    -p PROFILE_NAME   Apply the certificate to a (primary) SSL/TLS Service Profile.
    -s PROFILE_NAME   Apply the certificate to a (secondary) SSL/TLS Service Profile.

    -h                Display this help and exit.
    -v                Verbose mode.
```

FreeIPA specific options [-S] SERVICE, [-T] CERT_PROFILE and [-G] TYPE, were added to assist in requesting certificates for Services other than HTTP, for example LDAP, host etc. For example, I use a host certificate as a Subordinate certificate. To do so I also needed to add the option to specify an alternative Certificate Profile. See [FreeIPA Subordinate Certificates](https://frasertweedale.github.io/blog-redhat/posts/2018-08-21-ipa-subordinate-ca.html) for more information about that. Finally, I explored the option of using ECDSA, only to find FreeIPA still doesn't support it. But rather than stripping this code out, I left it in place for future use.

**Note:** FreeIPA only supports RSA keys. Hence the -G option is in preparation of future support of other keys. [More info](https://www.reddit.com/r/FreeIPA/comments/134puyw/freeipa_ca_pki_ecdsa_support/).

Run pan_instcert with sudo, it will test if it's run with uid 0 (root). Ensure this command is added to sudo rules if it must be used manually.

pan_instcert uses the same options, minus the FreeIPA specifics:
```
Usage: pan_instcert [-hv] -c CERT_CN [-n CERT_NAME] [OPTIONS] FQDN
This script uploads a certificate issued by ipa-getcert to a Palo Alto firewall
or Panorama and optionally adds it to up to two SSL/TLS Profiles.

    FQDN              Fully qualified name of the Palo Alto firewall or Panorama
                      interface. Must be reachable from this host on port TCP/443.
    -c CERT_CN        REQUIRED. Common Name (Subject) of the certificate, to find
                      the certificate and key files.

OPTIONS:
    -n CERT_NAME      Name of the certificate in PanOS configuration. Defaults to the
                      certificate Common Name.
    -Y                Append the current year '_YYYY' to the certificate name.

    -p PROFILE_NAME   Apply the certificate to a (primary) SSL/TLS Service Profile.
    -s PROFILE_NAME   Apply the certificate to a (secondary) SSL/TLS Service Profile.

    -h                Display this help and exit.
    -v                Verbose mode.
```

+ **-n** _CERT_NAME_: The name you wish to give the certificate on the device (Palo Alto Networks GUI:  Device –> Certificate Management –> Certificates). Existing names will be overwritten.


# Automated Renewal and Installation
ipa-getcert requests the certificate with the post-save option, thus no cronjob is needed to install renewed certificates and pan_instcert does not need to be manually called.

The post-save process will upload the renewed certificates, and take the same actions against SSL/TLS Service Profiles as when the certificate was initially requested and deployed using pan_getcert. If the -Y option is given the certificate name will be appended with the year ("_YYYY"), otherwise the certificate will be overwritten in the Palo Alto firewall or Panorama configuration.

Check the configured post-save command with:
```
sudo ipa-getcert list [-i Request_ID | -f Path_to_cert_file]
```

### Notes
+ As best-practice, one should use separate SSL/TLS Service Profiles for Portal and Gateway. These scripts assume best-practices are followed, but will also work with single-profile configurations.
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

# Expected CLI output
When requesting and deploying a certificate:
```
user@host:~$ sudo pan_getcert -v -c gp.domain.com -Y -p GP_PORTAL_PROFILE -s GP_EXT_GW_PROFILE fw01.domain.local
Certificate Common Name: gp.domain.com
  verbose=1
  CERT_CN: gp.domain.com
  CERT_NAME: gp.domain.com_2023
  PAN_FQDN: fw01.domain.local
Primary SSL/TLS Profile name: GP_PORTAL_PROFILE
Secondary SSL/TLS Profile name: GP_EXT_GW_PROFILE
New signing request "20230427151532" added.
Certificate requested for: gp.domain.com
  Certificate issue took 6 seconds, waiting for the post-save process to finish.
  Certificate install and commit by the post-save process on: fw01.domain.local took 84 seconds.
FINISHED: Check the Palo Alto firewall or Panorama to check the commit succeeded.
```

# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Automated Renewals
pan_instcert will log to `/var/log/pan_instcert.log`.

pan_getcert doesn't log as it's expected to be run once (manually).
```
cat /var/log/pan_instcert.log
```

If successful, you should see something similar to the following in the log file:
```
[2023-04-27 15:15:33+00:00]: START of pan_instcert.
[2023-04-27 15:15:33+00:00]: Certificate Common Name: gp.domain.com
[2023-04-27 15:15:34+00:00]: XML API output for crt: <response status="success"><result>Successfully imported gp.domain.com_2023 into candidate configuration</result></response>
[2023-04-27 15:15:35+00:00]: XML API output for key: <response status="success"><result>Successfully imported gp.domain.com_2023 into candidate configuration</result></response>
[2023-04-27 15:15:35+00:00]: Finished uploading certificate: gp.domain.com_2023
[2023-04-27 15:15:37+00:00]: Starting commit, please be patient.
[2023-04-27 15:17:01+00:00]: commit: success: "Configuration committed successfully"
[2023-04-27 15:17:01+00:00]: The commit took 84 seconds to complete.
[2023-04-27 15:17:01+00:00]: END - Finished certificate installation to: fw01.domain.local
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
