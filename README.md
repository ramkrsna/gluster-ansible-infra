glusterfs.infra
=========

This role helps the user to get started in deploying GlusterFS filesystem. The
glusterfs.infra role has multiple sub-roles which are invoked depending on the
variables that are set. The sub-roles are

1. firewall_config - Set up firewall rules (open ports, add services to zone)
2. backend_setup:
        - Create VDO volume (If vdo is opted) 
        - Create volume groups, logical volumes (thinpool, thin lv, thick lv)
        - Create xfs filesystem
        - Mount the filesystem

Requirements
------------

Ansible version 2.5 or above
GlusterFS version 3.2 or above
VDO utilities (Optional)

Role Variables
--------------

These are the superset of role variables. They are explained again in the
respective sub-roles directory.

### firewall_config
-------------------
| Name                     |Choices| Default value         | Comments                          |
|--------------------------|-------|-----------------------|-----------------------------------|
| glusterfs_infra_fw_state | enabled / disabled / present / absent    | UNDEF   | Enable or disable a setting. For ports: Should this port accept(enabled) or reject(disabled) connections. The states "present" and "absent" can only be used in zone level operations (i.e. when no other parameters but zone and state are set). |
| glusterfs_infra_fw_ports |    | UNDEF    | A list of ports in the format PORT/PROTO. For example 111/tcp. This is a list value.  |
| glusterfs_infra_fw_permanent  | true/false  | true | Whether to make the rule permanenet. |
| glusterfs_infra_fw_zone    | work / drop / internal / external / trusted / home / dmz / public / block | public   | The firewalld zone to add/remove to/from |
| glusterfs_infra_fw_services |    | UNDEF | Name of a service to add/remove to/from firewalld - service must be listed in output of firewall-cmd --get-services. This is a list variable

### backend_setup
-----------------
| Name                     |Choices| Default value         | Comments                          |
|--------------------------|-------|-----------------------|-----------------------------------|
| glusterfs_infra_vdo_state | present / absent  | present   | Optional variable, default is taken as present. |
| glusterfs_infra_vdo || UNDEF | Mandatory argument if vdo has to be setup. Key/Value pairs have to be given. name and device are the keys, see examples for syntax. |
| glusterfs_infra_disktype | JBOD / RAID6 / RAID10  | UNDEF   | Backend disk type. |
| glusterfs_infra_diskcount || UNDEF | RAID diskcount, can be ignored if disktype is JBOD  |
| glusterfs_infra_vg_name  || glusterfs_vg | Optional variable, if not provided glusterfs_vg is used as vgname. |
| glusterfs_infra_pvs  | | UNDEF | Comma-separated list of physical devices. If vdo is used this variable can be omitted. |
| glusterfs_infra_stripe_unit_size || UNDEF| Stripe unit size (KiB). *DO NOT* including trailing 'k' or 'K'  |
| glusterfs_infra_lv_poolmetadatasize || 16G | Metadata size for LV, recommended value 16G is used by default. That value can be overridden by setting the variable. Include the unit [G\|M\|K] |
| glusterfs_infra_lv_thinpoolname || glusterfs_thinpool | Optional variable. If omitted glusterfs_thinpool is used for thinpoolname. |
| glusterfs_infra_lv_thinpoolsize || 100%FREE | Thinpool size, if not set, entire disk is used. Include the unit [G\|M\|K] |
| glusterfs_infra_lv_logicalvols || UNDEF | This is a list of hash/dictionary variables, with keys, lvname and lvsize. See below for example. |
| glusterfs_infra_lv_thicklvname || glusterfs_infra_lv_thicklvname | Optional. Needed only if thick volume has to be created. The variable will have default name glusterfs_infra_lv_thicklvname if thicklvsize is defined. |
| glusterfs_infra_lv_thicklvsize || UNDEF | Optional. Needed only if thick volume has to be created. Include the unit [G\|M\|K]|
| glusterfs_infra_mount_devices | | UNDEF | This is a dictionary with mount values. path, and lv are the keys. |
| glusterfs_infra_ssd_disk | | UNDEF | SSD disk for cache setup, specific to HC setups. Should be absolute path. e.g /dev/sdc |
| glusterfs_infra_lv_cachelvname | | glusterfs_ssd_cache | Optional varialbe, if omitted glusterfs_ssd_cache is used by default. |
| glusterfs_infra_lv_cachelvsize | | UNDEF | Size of the cache logical volume. Used only while setting up cache. |
| glusterfs_infra_lv_cachemetalvname | | UNDEF | Optional. Cache metadata volume name. |
| glusterfs_infra_lv_cachemetalvsize | | UNDEF | Optional. Cache metadata volume size. |
| glusterfs_infra_cachemode | | writethrough | Optional. If omitted writethrough is used. |




#### VDO Variable
------------
If the backend disk has to be configured with VDO the variable glusterfs_infra_vdo has to be defined.

| Name                     |Choices| Default value         | Comments                          |
|--------------------------|-------|-----------------------|-----------------------------------|
| glusterfs_infra_vdo || UNDEF | Mandatory argument if vdo has to be setup. Key/Value pairs have to be given. name and device are the keys, see examples for syntax. |

```
For Example:
glusterfs_infra_vdo:
   - { name: 'hc_vdo_1', device: '/dev/vdb' }
   - { name: 'hc_vdo_2', device: '/dev/vdc' }
   - { name: 'hc_vdo_3', device: '/dev/vdd' }
```

#### Logical Volume variable
-----------------------
| Name                     |Choices| Default value         | Comments                          |
|--------------------------|-------|-----------------------|-----------------------------------|
| glusterfs_infra_lv_logicalvols || UNDEF | This is a list of hash/dictionary variables, with keys, lvname and lvsize. See below for example. |

```
For Example:
glusterfs_infra_lv_logicalvols:
   - { lvname: 'hc_images', lvsize: '500G' }
   - { lvname: 'hc_data', lvsize: '500G' }
```

#### Variables for mounting the filesystem
-----------------------------------------
| Name                     |Choices| Default value         | Comments                          |
|--------------------------|-------|-----------------------|-----------------------------------|
| glusterfs_infra_mount_devices | | UNDEF | This is a dictionary with mount values. path, and lv are the keys. |

```
For example:
glusterfs_infra_mount_devices:
        - { path: '/mnt/thinv', lv: <lvname> }
        - { path: '/mnt/thicklv', lv: "{{ glusterfs_infra_lv_thicklvname }}" }
```




Example Playbook
----------------

Configure the ports and services related to GlusterFS, create logical volumes and mount them.


```
---
- name: Setting up backend
  remote_user: root
  hosts: servers
  gather_facts: false

  vars:
     # Firewall setup
     glusterfs_infra_fw_ports:
        - 2049/tcp
        - 54321/tcp
        - 5900/tcp
        - 5900-6923/tcp
        - 5666/tcp
        - 16514/tcp
     glusterfs_infra_fw_services:
        - glusterfs

     # Set a disk type, Options: JBOD, RAID6, RAID10
     glusterfs_infra_disktype: RAID6

     # RAID6 and RAID10 diskcount (Needed only when disktype is raid)
     glusterfs_infra_diskcount: 10
     # Stripe unit size always in KiB
     glusterfs_infra_stripe_unit_size: 128

     # Variables for creating volume group
     glusterfs_infra_vg_name: glusterfs_vg
     glusterfs_infra_pvs: /dev/vdb,/dev/vdc

     # Create a thick volume name
     glusterfs_infra_lv_thicklvname: glusterfs_thicklv
     glusterfs_infra_lv_thicklvsize: 20G

     # Create a thinpool
     glusterfs_infra_lv_thinpoolname: glusterfs_thinpool
     glusterfs_infra_lv_thinpoolsize: 100G # If not provided entire disk is used

     # Create a thin volume
     glusterfs_infra_lv_logicalvols:
        - { lvname: 'hc_images', lvsize: '500G' }
        - { lvname: 'hc_data', lvsize: '500G' }


     # Mount the devices
     glusterfs_infra_mount_devices:
        - { path: '/mnt/thinv', lv: 'glusterfs_thinlv' }
        - { path: '/mnt/thicklv', lv: "{{ glusterfs_infra_lv_thicklvname }}" }

  roles:
     - glusterfs.infra

```

License
-------

GPLv3

