freeradius
===

This is a freeradius setup that uses a python script to control user-password authentication and MACs to place user+MAC combinations into the proper vlan. This was done on Arch linux (in a container, as root)

## goals

* Support a port-restricted LAN (+wifi) in a controlled physical area
* Provide a singular authentication strategy for supported clients (Windows, Linux, Android)
* Map authenticated user+MAC combinations to specific VLANs
* Integrate with Ubiquiti devices
* Avoid client-issued certificates (and management)
* Centralized configuration file
* As few open endpoints as possible

## plan

* This expects _ALL_ endpoints to support peap+mschapv2 (tested on Android 7.1.1, Arch using NetworkManager, and Windows 10)
* Avoids deviating from standard configs at all, assumes users are capable of handling things like systemd themselves
* The "default" config is the open point (e.g. ports 1812 and 1813 are open for udp traffic) so it has been stripped down where possible
* There is _NO_ issuance of any cert to clients for this implementation, we do handle managing the internal radius certs. This implementation is for a restricted area LAN.

---

## setup/install

to have freeradius actually able to execute the python scripts
```
vim /etc/environment
---
# append
PYTHONPATH=/etc/raddb/mods-config/python/
```

```
vim /usr/lib/systemd/system/freeradius.service
---
# add to the [Service] section
EnvironmentFile=/etc/environment
```

install python2 and freeradius
```
pacman -S freeradius python2
```

freepydius logging
```
mkdir /var/log/radius/freepydius
chown radiusd:radiusd /var/log/radius/freepydius
```

need to set the internal certs. all passwords _MUST_ be the same (example here: somestring)
```
vim /etc/raddb/certs/passwords.mk
---
PASSWORD_SERVER = 'somestring'
PASSWORD_CA = 'somestring'
PASSWORD_CLIENT = 'somestring'
USER_NAME   = 'somestring'
CA_DEFAULT_DAYS  = '1825'
```

now run the renew script
```
/etc/raddb/certs/renew.sh
```

we need the system to have knowledge of the key
```
vim /etc/raddb/clients.conf
---
certprivkey = somestring
```

---

## configuration

the json required (as shown below) uses a few approaches to get users+MAC into the proper VLAN
* [vlan].[name] is the user name which is used to set their password for freeradius configuration/checking
* [vlan] is used to set attributes to allow switches to get the corresponding attributes back
* [mac] is used to allow only certain devices to be used by certain users in certain vlans
* this corresponds to the implementation (in python) called freepydius under the mods-config/python folder

the following json represents a routing definition, prod.* and dev.* would be users, the 1234567890ab represents a MAC-device authenticating
```
vim /etc/raddb/mods-config/python/network.json
---
{
    "users":
    {
        "prod.user1":
        {
            "pass": "prodaccount$1",
            "macs":
            [
                "4fff3a7e7a11",
                "50009d1ea6cc",
                "6600991144dd"
            ]
        },
        "prod.user2":
        {
            "pass": "prodaccount$2",
            "macs":
            [
                "4fff3a7e7a11",
                "7fff3a777a11"
            ]
        },
        "dev.user1":
        {
            "pass": "devaccount",
            "macs":
            [
                "4fff3a7e7a11",
                "aaffbb112233"
            ]
        },
        "1234567890ab":
        {
            "pass": "1234567890ab",
            "vlan": "dev",
            "owner": "user1",
            "macs":
            [
                "1234567890ab"
            ]
        }
    },
    "vlans":
    {
        "prod": "10",
        "dev": "20"
    },
    "blacklist": ["dev"]
}
```

### blacklist

support blacklisting by
* users (e.g. user1)
* vlan (e.g. prod)
* vlan.user (e.g. prod.user1)

devices (MACs) per user can not be blacklisted, they should just be removed from the list of MACs. This is rationalized as either a device is 'bad' and needs to be removed OR the user is using a device and the user should be blacklisted until the problem is resolved

---

## Implementation Details

### Current

We are using: [peap](https://en.wikipedia.org/wiki/Protected_Extensible_Authentication_Protocol)+[mschapv2](https://en.wikipedia.org/wiki/Protected_Extensible_Authentication_Protocol#PEAPv0_with_EAP-MSCHAPv2) (no CA validation).

### Flow

1. The user does not enter credentials - the switch will send a request (on behalf of the system) to do MAC-based authentication
2. The user does enter credentials - those credentials will be accepted/rejected, if rejected go-to 1

Unauthenticated users are assigned no VLAN by radius and will be, by the networking backbone, assigned to a VLAN with restricted connection for troubleshooting/requests only.

### Analysis

We are able to review information about accounting (e.g. Start/Stop) to see connection information (e.g. packets/octets). See what devices are connecting over certain ports and switches (and by which user). Generate statistics and/or review stats over accepts/rejects/etc.

### Future

* Distribute the CA cert that freeradius uses; however, it is only minimally correct (it is correct-enough for inner-tunnel in freeradius) and becomes a management problem
* Limit further port authorization at the freeradius endpoint to switch + port (or similar)
* Perform functions based on being wired or wireless

---

## debugging

there are a few utilities in the mods-config/python/ folder associated with freepydius.py

### harness

allows for playing key/value pairs into the radius module

### replay

supports playing back a log (from freepydius output) back into the radius module
