# network-configure

This module configures networking, it allows you to create an ethernet, bridge or bond device based on the set parameters.
It also creates a custom service for managing the network which allows backups of the previous network configuration to be kept, and also forces the network to be down when required rather than hanging.
Something of note is the ordering of devices is critical due to the way Ubuntu builds up network devices. If things are not ordered correctly, then this causes devices to hang and other issues.

## Requirements

This module requires Ansible 2.4

## Usage

This role builds up a single network config file in the default location of `/etc/network/pre_combined_interfaces` which is defined by `network_configure_pre_interface_path` variable the file then contains all the data which is built up by this role.

The role then creates a systemd service which is named `force-networking`, this has some slight differences to the way the normal networking service works. First of all with every restart it creates a
backup copy which is by default located in `/etc/network/backup_combined_interfaces` and defined by the variable `network_configure_backup_interface_path`. It then copies the above pre_combined_interfaces file
to the following default location `/etc/network/interfaces.d/0_combined_interfaces` defined by variable `network_configure_combined_interface_path`.

The service also differs in the way it brings the network down and uses the --force parameter with ifdown. It also calls a script which is created by this role which checks any remaining up interfaces
takes them down and then flushes. This is found in the location `/opt/cleanup-interfaces.sh`

The following variables give control over the way the service is restarted


`network_configure_restart_service` - this triggers a network-force restart on configure


The following list is also crucial:

`network_configured_allowed_device_type` - This is a list of values which drives the 'device_type' of 'network_configure_interfaces', the defaults are [ eth, bond, bridge ]. Any interface which has one of these items as a device type
will be taken into account when running this role and be configured.

Within the network_configure_interfaces data one of the subkeys `device_type`, is compared against the above network_configured_allowed_device_type and if none of the values intersect that entry will be ignored. Also note this variable is used in a similar fashion in other roles such as the dhcp role

To configure the networking, the following dict is where the bulk of data is held, and may be laid out as follows:

```yaml
network_configure_interfaces:
  primary:                      # This is a friendly name, to allow us to override this interface
    device_name: enp0s8         # This the physical interface name
    device_type: [ eth ]        # List of items, this needs to be one of the items in the list above, otherwise it will be ignored. This is used in other roles such as dhcp.
    bootproto: static           # The bootproto needs to be set to drive how the interface is handled
    network_order: 0            # Network order is 0 first and 100 last, and drives the order the interfaces are created in. This is important with things like bridges or bonds where the physical interfaces need configuring first
    networking:                 # General network config, every line in here is placed directly into the config file below the device.
      address: 192.168.50.4
      netmask: 255.255.255.0
```

## Examples

### Simple static interface

To assign a simple static network interface use the following:

```yaml
    network_configure_interfaces:
      primary:
        device_name: enp0s9
        device_type: [ eth ]
        bootproto: static
        network_order: 0
        networking:
          address: 192.168.50.4
          netmask: 255.255.255.0
```

Which would create the following config:

```
    auto enp0s9
      iface enp0s9 inet static
      netmask 255.255.255.0
      address 192.168.50.4
```

### Simple bond interface

To assign two physical network devices into a bond use the following config example
be careful to avoid using the name 'bond0' since this already exists when the bond module
is used and causes problems when trying to configure networking.
The ordering is also important and the two physical devices need to be brought up
before the bond device does

```yaml
    network_configure_interfaces:
      primary:
        device_name: enp0s16
        device_type: [ eth ]
        bootproto: manual
        network_order: 1
        networking:
          bond-master: bonddevice0
      secondary:
        device_name: enp0s17
        device_type: [ eth ]
        bootproto: manual
        network_order: 2
        networking:
          bond-master: bonddevice0
      tertiary:
        device_name: bonddevice0
        device_type: [ bond ]
        bootproto: static
        network_order: 10
        networking:
          address: 192.168.53.4
          netmask: 255.255.255.0
          bond-mode: active-backup
          bond-miimon: 100
          bond-updelay: 200
          bond-downdelay: 200
          bond-primary: enp0s16
          bond-slaves: none
```

This would create the following config:

```
    auto enp0s16
      iface enp0s16 inet manual
      bond-master bonddevice0

    auto enp0s17
      iface enp0s17 inet manual
      bond-master bonddevice0

    auto bonddevice0
      iface bonddevice0 inet static
      bond-miimon 100
      bond-downdelay 200
      bond-slaves none
      netmask 255.255.255.0
      bond-mode active-backup
      address 192.168.53.4
      bond-updelay 200
      bond-primary enp0s16
```

### Simple bridge interface

To assign a physical device to a simple network bridge the following configuration can be used:

```yaml
    network_configure_interfaces:
      primary:
        device_name: enp0s9
        device_type: [ eth ]
        bootproto: manual
        network_order: 0
      bonddevice:
        device_name: bond0
        device_type: [ bridge ]
        bootproto: static
        network_order: 10
        networking:
          bridge_stp: off
          bridge_waitport: 0
          bridge_fd: 0
          address: 192.168.50.4
          netmask: 255.255.255.0
          bridge_ports: enp0s9
```

This would create the following config:

```
  auto enp0s9
    iface enp0s9 inet manual

  auto bridge0
    iface bridge0 inet static
    bridge_stp False
    bridge_fd 0
    bridge_waitport 0
    netmask 255.255.255.0
    bridge_ports enp0s9
    address 192.168.50.4
```

## Example Playbook

Example to call:

    - hosts: all
      roles:
         - { role: network-configure }
