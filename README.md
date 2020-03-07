**_Note:_** Because I'm running my own servers for serveral years, main development is done at at https://git.ypbind.de/cgit/ipxe-distributor/

----

# `ipxe-distributor` - A webservice to provide iPXE configuration based on MAC,serial or group name
## Preface
This service provides a defined method to serve [iPXE](https://ipxe.org/) configuration from a single
YAML configuration.

Except for the listener configuration (`global` -> `host` / `port`) no service restart is required on
configuration change.

## Requirements
Required Python packages:

* [PyYAML - The next generation YAML parser and emitter for Python.](https://github.com/yaml/pyyaml) to parse the YAML configuration
* [Bottle: Python Web Framework](https://bottlepy.org) for webservice

## Configuration file
The configuration format is [YAML](https://yaml.org/).

### Global configuration - `global`
The `global` dictionary may have two keys:

* `host` - address to listen on (Default: `localhost`)
* `port` - port to listen on, *must* be an integer (Default: `8080`)

### Defaults - `default`
Defaults are defined in the `default` dictionary. Subkeys are:

#### iPXE data to prepend to all iPXE output - `ipxe_prepend`
`ipxe_prepend` must be an array of data to be prepended to *all* iPXE results

#### iPXE data to append to all iPXE output - `ipxe_append`
`ipxe_append` must be an array of data to be appended to *all* iPXE results

#### Default iPXE if nothing matches - `default_image`
`default_image` is an array of data to send when no match was found.
Default is `sanboot --no-describe --drive 0x80` (boot from local disk - see [Boot from SAN device](https://ipxe.org/cmd/sanboot))

This can also be used for a server entry as image `default` (with `ipx_prepend` and `ipx_append` used if defined)

### Image definitions - `images`
`images` contains the dictionaries of images to provide. The only supported key is `action` which is an array of iPXE commands to present
to the requestor.

### Node definitions - `nodes`
`nodes` is a dictionary of mapping node names to the image definitions using one of the provided keys:

* `mac` - MAC address
* `serial` - serial number
* `group` - generic group name

### Example
---
global:
    host: "127.0.0.1"
    port: 8080

default:
    # prepend to _all_ IPXE data
    ipxe_prepend:
        - | 
            # ! *-timeout is in ms !
            set menu-timeout 60000
            set submenu-timeout ${menu-timeout}

            # set default action to "exit" if not set
            isset ${menu-default} || set menu-default exit_ipxe

    # append to _all_ IPXE data
    ipxe_append:
        - "choose --timeout ${menu-timeout} --default ${menu-default} selected || goto abort"
        - "set menu-timeout 0"

    # default if there is no match
    default_image:
        - "# Boot the first local HDD"
        - "sanboot --no-describe --drive 0x80"

images:
    centos_7.6:
        action:
            - "initrd http://mirror.centos.org/centos/7/os/x86_64/images/pxeboot/initrd.img"
            - "chain http://mirror.centos.org/centos/7/os/x86_64/images/pxeboot/vmlinuz net.ifnames=0 biosdevname=0 ksdevice=eth2 inst.repo=http://mirror.centos.org/centos/7/os/x86_64/ inst.lang=en_GB inst.keymap=be-latin1 inst.vnc inst.vncpassword=CHANGEME ip=x.x.x.x netmask=x.x.x.x gateway=x.x.x.x dns=x.x.x.x"

    redhat_7:
        action:
            - "initrd http://mirror.redhat.org/redhat/7/os/x86_64/images/pxeboot/initrd.img"
            - "chain http://mirror.redhat.org/redhat/7/os/x86_64/images/pxeboot/vmlinuz net.ifnames=0 biosdevname=0 ksdevice=eth2 inst.repo=http://mirror.redhat.org/redhat/7/os/x86_64/ inst.lang=en_GB inst.keymap=be-latin1 inst.vnc inst.vncpassword=CHANGEME ip=x.x.x.x netmask=x.x.x.x gateway=x.x.x.x dns=x.x.x.x"

nodes:
    server01:
        mac: 12:34:56:78:9a:bc
        image: "centos_7.6"
    server02:
        mac: bc-9a-78-56-34-12
        image: "redhat_7"
        serial: SX123456789
    server03:
        mac: cafebabebabe
        image: "default"
        serial: SX123456789
    servergroup:
        image: "redhat_7"
        group: "ldapservers"
```

## Webservice
### Standalone
Although not recommended this script can be run as standalone service.

### UWSGI
Optionally the script can be started as [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) application.

### Behind reverse proxy
For HTTPS or authentication the script can be run behind a reverse proxy (e.g. [NGINX](https://nginx.org), [Apache](https://httpd.apache.org/), ...).

## Request routing
### MAC address
The requests based on the MAC address should use `/mac/<mac>`. `<mac>` is the MAC address, octets separated by `:`, `-` or without separator.
For example:

```
#!ipxe
chain http://10.10.10.10/mac/${net0/mac}
```
(see [iPXE - MAC address setting](https://ipxe.org/cfg/mac))

### Serial
Serial number based requests should use `/serial/<serial>` as request.
At the moment the serial number should match the regular expression of `[a-zA-Z0-9_\[\]\-. ]+`

For example:

```
#!ipxe
chain http://10.10.10.10/serial/${serial}
```
(see [iPXE - Serial number](https://ipxe.org/cfg/serial))

**Note:** Serial numbers containing one or multiple slashes are not supported (yet ?) as the confuse the request routing.

### Group
To provide the same configuration for a group the path `/group/<groupname>` must be used.
The current regular expression for groups is `[a-zA-Z0-9_\[\]\-. ]+`

**Note:** Like the serial number the group names should not contain one or multiple slashes

