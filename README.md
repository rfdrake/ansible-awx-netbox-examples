# Overview

This repository includes three examples of ways to use netbox with ansible.


# Inventory

This uses the inventory file inventory/10-netbox-inventory.yml

This is an example of using netbox as an inventory source.

Because of reasons (https://github.com/ansible/awx/issues/223 and
https://github.com/ansible/awx/issues/137) ansible doesn't have good support
for vaults in inventory sources.  There are a couple of potential workarounds.

First you can define a custom credential type in AWX (described below in the
next section), which will inject the variables using either environment variables,
or extra_vars.

This has the advantage of not storing any passwords in your source repository,
but the disadvantage of the passwords not being encrypted within the AWX ->
ansible environment.  This is probably not a concern, and this is probably the
best option if you can't use a vault.

Another possibility is to store your ansible vault password in a file inside
the execution environment.  The downside to this is that your vault password
will be unencrypted in the container inside your docker registry.

Another option is to just hardcode the netbox token in your inventory file,
but that isn't good because you will need to commit this repo to an SCM to
import it to AWX, so you should try to avoid plaintext passwords.

## Creating a custom credential type

Name: Netbox

Input Configuration:
```
fields:
  - type: string
    id: netbox_url
    label: Netbox URL

  - type: string
    id: netbox_token
    label: "Netbox Token"
    secret: True
required:
  - netbox_url
  - netbox_token
```

Injector Configuration:
```
extra_vars:
  netbox_url: '{{ netbox_url }}'
  netbox_token: '{{ netbox_token }}'
env:
  NETBOX_URL: '{{ netbox_url }}'
  NETBOX_TOKEN: '{{ netbox_token }}'
```

# Adding a new device with an IP from a range

This uses the playbook add_new_device_with_ip.yml

This is a playbook showing how to use the netbox collection to pull data (an
IP range) from netbox and apply it to an interface.

This requires you to pre-define some variables first.  You can define them in AWX
and set "prompt on launch" to change them.

```
device_name: 'test-device-009'
site: 'my-site-slug'
device_type: 'BigServer 90'
device_serial: ''
device_role: 'Server'
comments: ''
status: 'Active'
ip_range: '10.1.2.0/24'
```

# Adding multiple devices with IP and mac address

This uses the playbook devices.yml

This playbook shows how to add many devices with a YAML source file.  Our method of generating this config is to
make a tab separated file that looks like the following, then use https://www.convertcsv.com/csv-to-yaml.htm to change it to YAML.

```
name	serial	ip6	mac	ip4	manufacturer	device_type	comments	status	device_role	site
sw1-test-network			abcdabcdab01	10.16.11.11/24	raisecom	iscom2624g-4c-pwr-ac		staged	Switch	test-network
sw2-test-network			abcdabcdab02	10.16.11.12/24	raisecom	iscom2624g-4c-pwr-ac		staged	Switch	test-network
```

Here is an example of the required YAML config:

```
devices:
  - name: sw1-test-network
    serial: null
    ip6: null
    mac: abcdabcdab01
    ip4: 10.16.11.11/24
    manufacturer: raisecom
    device_type: iscom2624g-4c-pwr-ac
    comments: null
    status: staged
    device_role: Switch
    site: test-network
  - name: sw2-test-network
    serial: null
    ip6: null
    mac: abcdabcdab02
    ip4: 10.16.11.12/24
    manufacturer: raisecom
    device_type: iscom2624g-4c-pwr-ac
    comments: null
    status: staged
    device_role: Switch
    site: test-network
```

# Update to the above, can use CLI if needed (using the yq program)

I should note that as of this writing I have not tested the following commands
together.  Our usual workflow was to put the output YAML into ansible AWX and
execute the playbook there.  I will update this once I verify the commands
work.

Install the yq program from https://github.com/mikefarah/yq/releases

```
name,serial,ip6,mac,ip4,manufacturer,device_type,comments,status,device_role,site
sw1-test-network,,,abcdabcdab01,10.16.11.11/24,raisecom,iscom2624g-4c-pwr-ac,,staged,Switch,test-network
sw2-test-network,,,abcdabcdab02,10.16.11.12/24,raisecom,iscom2624g-4c-pwr-ac,,staged,Switch,test-network
```

    (echo devices: ; yq -p=csv -o=yaml input.csv) > device-list.yml

This also works with tsv for tab seperated content.

    time ansible-playbook -e @device-list.yml devices.yml


# Generating templates in AWX from a Netbox webhook

This uses the playbook generate_template.yml

This shows how to extract information about a device and its related objects,
then transform it into something ansible can use to build a template. I
change the mac address format,  derive the default gateway from the subnet,
and change the generated filename based on if a mac_address is defined.  I
also show how to call different roles based on the manufacturer name.

When netbox fires a webhook it sends the object that changed, but this does
not include objects that are attached to it.  For example, a device change
will send the device data, but it won't include interfaces associated with it.

So if you need information from a device interface, like a mac address, then
you need to get it a different way.  This is why the playbook makes separate requests
for interface data from netbox.

## What the webhook looks like on netbox side

This is going to be split into two different playbooks/webhooks.  The reason
being that if you do it on "Create" and "Update" then netbox will send
webhooks for every interface when a device is created.  This can cause
amplification to 3,000 webhooks when creating lots of devices.

### DCIM > Device

1.  Content types is set to "DCIM > Device"
2.  Events is set to "Creations" and "Updates"
3.  URL is https://<awx server>/api/v2/job_templates/<awx template id>/launch/
4.  HTTP Method: POST
5.  HTTP content type: application/json
6.  Additional headers: echo "Authorization: Basic $(echo -n "user:pass" | base64)"
7.  Body Template looks like the following:

```
{
  "extra_vars": {
        "webhook_data": {{ data|tojson }},
        "webhook_model": "{{ model }}",
        "username": "{{ username }}"
  }
}
```
8. Conditional looks like the following.  I also restricted the webhook to
only fire on device types that we are making a template for.
```
{
    "and": [
        {
            "attr": "primary_ip",
            "negate": true,
            "value": null
        },
        {
            "attr": "device_type.manufacturer.name",
            "op": "in",
            "value": [
                "Raisecom",
                "Cisco",
                "Brocade",
                "MikroTik"
            ]
        }
    ]
}
```

### DCIM > Interface

The URL for the second webhook must be different.  I didn't really make use of
this because it fires more than I want, but I'm leaving it here as an example.

1.  Content types is set to "DCIM > Interface"
2.  Events is set to "Updates"
3.  URL is https://<awx server>/api/v2/job_templates/<awx template id>/launch/
4.  HTTP Method: POST
5.  HTTP content type: application/json
6.  Additional headers: echo "Authorization: Basic $(echo -n "user:pass" | base64)"
7.  Body Template looks like the following:

```
{
  "extra_vars": {
        "webhook_data": {{ data|tojson }},
        "webhook_model": "{{ model }}",
        "username": "{{ username }}"
  }
}
```

## AWX templates side

You probably want to enable "Concurrent Jobs" so it will fire off multiple
jobs if you create more than one device at a time in netbox.


## Why do we use netbox API rather than ansible netbox.netbox collections?

I tried using netbox.netbox.netbox_device to get the device, but the data it
returns isn't as useful as the direct API queries.  Some of the links are not
followed, so while the devices API gives you the "site" object attached to the
device, the netbox_device ansible command would just give you the site id.

I later realized you can do some of this with nb_lookup for a more ansible-ish
way to do things.  I've mixed both in the example so that you can see how it
works.  The downside being that it requires another python dependency for
jmespath.

## Notes about configuration contexts

In some examples you'll find "context.trunk_ports" and "context.uplink_port".
These are defined as JSON inside netbox configuration contexts and associated
with different switch models.  You can have a more specific configuration
override less specific ones.  So maybe in all switch models from a certain
vendor, the management port is me0.  You can specify that in a context
associated with the manufacturer, then put more specific information in the
individual devices.  For another example, we can specify that a 12 port
switch has 11 trunk_ports and 1 uplink_port, or many other combinations.

Here is an edited example from a Nexus 3548P-10G

```
{
    "management_interface": "mgmt0",
    "trunk_ports": [
        "e1/1-44, e1/46, e1/48"
    ],
    "uplink_port": "Ethernet1/47"
}
```

## Other notes

I'm including a full example of a template generator for Nexus switches.  This
does not include variables that you would need to define in your environment,
like TACACS servers, NTP servers, SNMP communities, etc.  You can define these
in your configuration contexts, or you can put them into ansible vault or
wherever you wish.

# AWX quirks

## Use a custom EE

If you're going to use netbox commands from the netbox.netbox collection you will
probably need to run a custom execution environment.  This is due to the
collection requiring the pynetbox python module, but not being able to install
it in the default collection.

You can use https://github.com/rfdrake/awx-netbox-ee to build one if needed.

## connection: local

ansible really likes to ssh to a remote host to run it's playbooks.  That
doesn't work if you're connecting to localhost and running inside of a
container.  "connection: local" inside the playbook forces it to use local
instead of trying to ssh.

## delegate_to: localhost

Most netbox commands are being executed on the box running ansible (localhost).
So traditionally, the inventory doesn't get utilized.  This is sub-optimal because it
won't fork for most of the playbooks.  You can get around this by sending your
device list as the inventory, then running delegate_to: localhost which forces the task
to run on the local machine.

## copy the template from awx to somewhere else

The last step in my example is an scp to push the template to another server.
The reason for this is extracting the template from the execution environment
in awx is a bit of a challenge.  We could mount a custom filesystem into the
container (via persistent volume), but it's easier to send it out via scp.

# Running this without AWX (python and ansible requirements)

If you're running this in ansible CLI without the benefits of AWX, or if
you're not using my custom execution environment, or if you are
using a different management system like rundeck you might need to modify
things a bit.

There are a couple of unstated dependencies which I found when trying to test
this in other environments.  Here is a cli list of dependencies needed.

```
ansible-galaxy collections install netbox.netbox community.network ansible.utils
pip install pynetbox packaging pytz netaddr jmespath
```
