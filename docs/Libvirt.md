<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Libvirt VM Monitoring](#libvirt-vm-monitoring)
  - [Overview](#overview)
  - [Configuration](#configuration)
  - [Instance Cache](#instance-cache)
  - [Metrics Cache](#metrics-cache)
  - [Per-Instance Metrics](#per-instance-metrics)
    - [host_alive_status Codes](#host_alive_status-codes)
    - [Ping Checks](#ping-checks)
      - [Requirements](#requirements)
      - [Detection](#detection)
      - [Algorithm](#algorithm)
      - [Client Configuration](#client-configuration)
      - [Troubleshooting](#troubleshooting)
  - [VM Dimensions](#vm-dimensions)
  - [Aggregate Metrics](#aggregate-metrics)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# Libvirt VM Monitoring

## Overview
The Libvirt plugin provides metrics for virtual machines when run on the hypervisor server.  It provides two sets of metrics per measurement: one designed for the owner of the VM, and one intended for the owner of the hypervisor server.

## Configuration
The `monasca-setup` program will configure the Libvirt plugin if `nova-compute` is running, its `nova.conf` config file is readable by the Monasca Agent user (default: 'mon-agent'), and `python-novaclient` is installed.

In order to fetch data on hosted compute instances, the Libvirt plugin needs to be able to talk to the Nova API.  It does this using credentials found in `nova.conf` under `[keystone_authtoken]`, obtained when `monasca-setup` is run, and stored in `/etc/monasca/agent/conf.d/libvirt.yaml` as `admin_user`, `admin_password`, `admin_tenant_name`, and `admin_password`.  These credentials are only used to build and update the [Instance Cache](#instance-cache).

The Libvirt plugin uses a cache directory to persist data, which is `/dev/shm` by default.  On non-Linux systems (BSD, Mac OSX), `/dev/shm` may not exist, so `cache_dir` would need to be changed accordingly, either in `monasca_setup/detection/plugins/libvirt.py` prior to running `monasca-setup`, or `/etc/monasca/agent/conf.d/libvirt.yaml` afterwards.

If the owner of the VM is in a different tenant the Agent Cross-Tenant Metric Submission can be setup. See this [documentation](https://github.com/openstack/monasca-agent/blob/master/docs/MonascaMetrics.md#cross-tenant-metric-submission) for details.

`nova_refresh` specifies the number of seconds between calls to the Nova API to refresh the instance cache.  This is helpful for updating VM hostname and pruning deleted instances from the cache.  By default, it is set to 14,400 seconds (four hours).  Set to 0 to refresh every time the Collector runs, or to None to disable regular refreshes entirely (though the instance cache will still be refreshed if a new instance is detected).

`metadata` specifies the list of instance metadata keys to be included as dimensions with the cross-tenant metrics for the operations project. This is helpful to give more information about an instance. When using the agent setup scripts, by default `scale_group` metadata is enabled for supporting auto scaling in Heat.

`customer_metadata` specifies the list of instance metadata keys to be included as dimensions with customer metrics. This is helpful to give more information about an instance.

`vm_probation` specifies a period of time (in seconds) in which to suspend metrics from a newly-created VM.  This is to prevent quickly-obsolete metrics in an environment with a high amount of instance churn (VMs created and destroyed in rapid succession).  The default probation length is 300 seconds (five minutes).  Setting to 0 disables VM probation, and metrics will be recorded as soon as possible after a VM is created.

`ping_check` includes the entire command line (sans the IP address, which is automatically appended) used to perform a ping check against instances, with a keyword `NAMESPACE` automatically replaced with the appropriate network namespace for the VM being monitored.  Set to False (or omit altogether) to disable ping checks.  See [ping checks](#ping-checks) below for more detail on how ping checks are set up and how they work.

`alive_only` will suppress all per-VM metrics aside from `host_alive_status` and `vm.host_alive_status`, including all I/O, network, memory, ping, and CPU metrics.  [Aggregate Metrics](#aggregate-metrics), however, would still be enabled if `alive_only` is true.  By default, `alive_only` is false.

`network_use_bits` will submit network metrics in bits rather than bytes.  This will stop submitting the metrics `net.in_bytes_sec` and `net.out_bytes_sec`, and instead submit `net.in_bits_sec` and `net.out_bits_sec`.

Example config:
```
init_config:
    admin_password: pass
    admin_tenant_name: service
    admin_user: nova
    identity_uri: 'http://192.168.10.5:35357/v2.0'
    region_name: 'region1'
    cache_dir: /dev/shm
    nova_refresh: 14400
    metadata:
    - scale_group
    vm_probation: 300
    ping_check: /opt/stack/venv/monasca_agent-20160224T213950Z/bin/ip netns exec NAMESPACE
      /bin/ping -n -c1 -w1 -q
    alive_only: false
    network_use_bits: false
instances:
    - {}
```
`instances` are null in `libvirt.yaml`  because the libvirt plugin detects and runs against all provisioned VM instances; specifying them in `libvirt.yaml` is unnecessary.

Note: If the Nova service login credentials are changed, `monasca-setup` would need to be re-run to use the new credentials.  Alternately, `/etc/monasca/agent/conf.d/libvirt.yaml` could be modified directly.

Example `monasca-setup` usage:
```
monasca-setup -d libvirt -a 'ping_check=false alive_only=false' --overwrite
```

## Instance Cache
The instance cache (`/dev/shm/libvirt_instances.json` by default) contains data that is not available to libvirt, but queried from Nova.  To limit calls to the Nova API, the cache is only updated if a new instance is detected (libvirt sees an instance not already in the cache), or every `nova_refresh` seconds (see Configuration above).

Example cache (pretty-printed):
```
{
   "last_update" : 1450121034,
   "instance-00000005" : {
      "created" : "2015-12-14T19:10:07Z",
      "instance_uuid" : "94b8511c-d4de-40c3-9676-558f28e0c3c1",
      "network" : [
         {
            "ip" : "10.0.0.3",
            "namespace" : "qrouter-ae714057-4453-48c4-81cb-15f8db9434a8"
         }
      ],
      "disk" : 1,
      "tenant_id" : "7d8e24a1e0cb4f8c8dedfb2010992b62",
      "zone" : "nova",
      "scale_group": "a1207522-c5fb-4621-a839-c00b638cfb47",
      "vcpus" : 1,
      "hostname" : "vm01",
      "ram" : 512
   }
}
```

## Metrics Cache
The libvirt inspector returns *counters*, but it is much more useful to use *rates* instead.  To convert counters to rates, a metrics cache is used, stored in `/dev/shm/libvirt_metrics.yaml` by default.  For each measurement gathered, the current value and timestamp (UNIX epoch) are recorded in the cache.  The subsequent run of the Monasca Agent Collector compares current values against prior ones, and computes the rate.

Since CPU Time is provided in nanoseconds, the timestamp recorded has nanosecond resolution.  Otherwise, integer seconds are used.

Example cache (pretty-printed excerpt, see next section for complete list of available metrics):
```
{
   "instance-00000005" : {
      "net.tx_bytes" : {
         "tap65d5c428-b4" : {
            "value" : 5178,
            "timestamp" : 1450121045.53221
         }
      },
      "io.read_requests" : {
         "hdd" : {
            "timestamp" : 1450121045.51788,
            "value" : 1
         },
         "vda" : {
            "timestamp" : 1450121045.50513,
            "value" : 512
         }
      },
      "net.tx_packets" : {
         "tap65d5c428-b4" : {
            "value" : 54,
            "timestamp" : 1450121045.53221
         }
      },
      "net.rx_packets" : {
         "tap65d5c428-b4" : {
            "timestamp" : 1450121045.53221,
            "value" : 63
         }
      },
      "net.rx_bytes" : {
         "tap65d5c428-b4" : {
            "value" : 6909,
            "timestamp" : 1450121045.53221
         }
      },
      "io.write_requests" : {
         "vda" : {
            "timestamp" : 1450121045.50513,
            "value" : 51
         },
         "hdd" : {
            "value" : 0,
            "timestamp" : 1450121045.51788
         }
      },
      "cpu.time" : {
         "value" : 17060000000,
         "timestamp" : 1450121045.4782
      },
      "io.errors" : {
         "hdd" : {
            "value" : -1,
            "timestamp" : 1450121045.51788
         },
         "vda" : {
            "timestamp" : 1450121045.50513,
            "value" : -1
         }
      },
      "io.read_bytes" : {
         "vda" : {
            "timestamp" : 1450121045.50513,
            "value" : 11591680
         },
         "hdd" : {
            "value" : 30,
            "timestamp" : 1450121045.51788
         }
      },
      "io.write_bytes" : {
         "vda" : {
            "timestamp" : 1450121045.50513,
            "value" : 230400
         },
         "hdd" : {
            "timestamp" : 1450121045.51788,
            "value" : 0
         }
      }
   }
}
```
## Per-Instance Metrics

| Name                 | Description                            | Associated Dimensions  |
| -------------------- | -------------------------------------- | ---------------------- |
| cpu.utilization_perc | Overall CPU utilization (percentage)   |                        |
| cpu.utilization_norm_perc | Normalized CPU utilization (percentage) |                  |
| host_alive_status    | See [host_alive_status Codes](#host_alive_status-codes) below | |
| io.read_ops_sec      | Disk I/O read operations per second    | 'device' (ie, 'hdd')   |
| io.write_ops_sec     | Disk I/O write operations per second   | 'device' (ie, 'hdd')   |
| io.read_bytes_sec    | Disk I/O read bytes per second         | 'device' (ie, 'hdd')   |
| io.write_bytes_sec   | Disk I/O write bytes per second        | 'device' (ie, 'hdd')   |
| io.errors_sec        | Disk I/O errors per second             | 'device' (ie, 'hdd')   |
| net.in_packets_sec   | Network received packets per second    | 'device' (ie, 'vnet0') |
| net.out_packets_sec  | Network transmitted packets per second | 'device' (ie, 'vnet0') |
| net.in_bytes_sec     | Network received bytes per second      | 'device' (ie, 'vnet0') |
| net.out_bytes_sec    | Network transmitted bytes per second   | 'device' (ie, 'vnet0') |
| mem.free_mb          | Free memory in Mbytes                  |                        |
| mem.total_mb         | Total memory in Mbytes                 |                        |
| mem.used_mb          | Used memory in Mbytes                  |                        |
| mem.free_perc        | Percent of memory free                 |                        |
| mem.swap_used_mb     | Used swap space in Mbytes              |                        |
| ping_status          | 0 for ping success, 1 for ping failure |                        |
| cpu.time_ms          | Cumulative CPU time (in ms), an Operations-only metric |        |
| mem.resident_mb      | Total memory used on host, an Operations-only metric |          |

### host_alive_status Codes
| Code | Description                          | value_meta 'detail'                    |
| ---- | -------------------------------------|--------------------------------------- |
| -1   | No state                             | VM has no state                        |
|  0   | Running / OK                         | None                                   |
|  1   | Idle / blocked                       | VM is blocked                          |
|  2   | Paused                               | VM is paused                           |
|  3   | Shutting down                        | VM is shutting down                    |
|  4   | Shut off                             | VM has been shut off                   |
|  4   | Nova suspend                         | VM has been suspended                  |
|  5   | Crashed                              | VM has crashed                         |
|  6   | Power management suspend (S3 state)  | VM is in power management (s3) suspend |


Memory statistics require a balloon driver on the VM.  For the Linux kernel, this is the `CONFIG_VIRTIO_BALLOON` configuration parameter, active by default in Ubuntu, and enabled by default as a kernel module in Debian, CentOS, and SUSE.

Since separate metrics are sent to the VM's owner as well as Operations, all metric names designed for Operations are prefixed with "vm." to easily distinguish between VM metrics and compute host's metrics.

### Ping Checks
The Libvirt plugin provides the ability to perform an ICMP ping test against hosted VMs.
It is helpful for determining, for example, if a VM is in a panicked or halted state, which in both cases may appear to the hypervisor as "Running / OK."  However, in order for ping checks to work, certain environmental requirements must be met.

#### Requirements
1. Neutron L3 agent in DVR mode (legacy mode is supported on single-node installations, such as devstack).
2. Neutron L2 plugin with a tenant network type of `vlan` or `vxlan` (other types may be supported, but have not been tested).
3. The `python-neutronclient` library and its dependencies installed and available to the Monasca Agent
4. Each VM needs an appropriate security group configuration to allow ICMP

#### Detection
The monasca-setup detection plugin for libvirt performs the following tests and tasks before configuring ping checks:

1. Ability to determine the name of the user under which monasca-agent processes run (eg, `mon-agent`)
2. Availability of the `python-neutronclient` library (by attempting to import `client` from `neutronclient.v2_0`)
3. A separate enhanced-capabilities `ip` command exists:
   a. The detection plugin copies `/sbin/ip` to `sys.path[0]` (see the [configuration](#configuration) section above for an example)
   b. Permissions on the copy are changed to the `mon-agent` user (or whichever Agent user is configured), mode 0700.
   c. The `/sbin/setcap` command is called, applying `cap_sys_admin+ep` to the copy, as `cap_sys_admin` is the only capability which provides `setns`, necessary to execute commands in a separate namespace.
   d. The detection plugin confirms that the enhanced capabilities were successfully applied
4. Existence of a ping command; detection will try `/usr/bin/fping`, `/sbin/fping`, and `/bin/ping` in that order.  `fping` is preferred because it allows for sub-second timeouts, but is not installed by default in some Linux distributions.

If any of the above requirements fail, a WARN-level message is output, describing the problem.  The libvirt plugin will continue to function without these requirements, but ping checks will be disabled.

#### Algorithm
Instance IP and namespace information is stored in the instance cache, rebuilt only when a new VM has been detected, or every 300 second (by default), whichever comes first.  Here is the general algorithm for how IP, namespace, and security parameters are detected.

First, build a list of all Neutron ports (networks).  This list will be referenced frequently later.  Connection to the Neutron API, through python-neutronclient, uses the same authentication credentials as calls to the Nova API.  These credentials are captured from nova.conf and stored in libvirt.yaml during the monasca-setup process.

Look for the presence of network namespaces on the compute node by running `ip netns list`.  This determines if:

1. Neutron is running in distributed-routing mode
2. Neutron is running in legacy or centralized-routing mode but on a single node (a la devstack)

If no network namespaces have been found, there is no need to continue trying to fetch information to support ping checks.

Assuming namespaces have been found, poll Neutron for a complete list of security groups, and store these as a dictionary object for quick access.

Start walking through each detected VM instance.  Each instance can have one or more Neutron networks, and each network can have one or more interface.  If the interface has a 'fixed' IP address (as opposed to floating), consult the port cache to find the subnet ID for that IP address.  The subnet ID is a unique identifier that will be used to find the correct namespace ID, which is also found in the port cache.

Once we find an active matching router interface for the tenant and VM interface subnet ID, we can verify that the VM's security policy allows ICMP from that router interface's IP address (see Client Configuration below for an example of this IP address).

The process of looking for namespaces and security rules occurs each time the instance cache is refreshed, so for normal operation, all this information is stored in the instance cache.

#### Client Configuration
The VM owner would need to add a security rule to allow ICMP access to their VM.  The simplest implementation would be to allow ICMP globally:
```
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```
However, more security-conscious customers may not want the world to ping their VM, so at a minimum, ICMP needs to be allowed for the subnet gateway IP address.

#### Troubleshooting

To limit false negative, ping checks will not be performed unless all the requirements are met.  The most direct way to verify all the requirements are met for a single VM is to look at the `/dev/shm/libvirt_instances.json` file on the compute node where the VM is hosted.  Any VM that meets all the requirements will have a "network" section, containing the IP address and network namespace for each correctly-configured device.
```
"network" : [ { "ip" : "10.0.0.3", "namespace" : "qrouter-ae714057-4453-48c4-81cb-15f8db9434a8" } ],
```
You can attempt to ping the IP address through the given namespace with a command like
```
sudo ip netns exec qrouter-ae714057-4453-48c4-81cb-15f8db9434a8 ping 10.0.0.3
```

Other questions you could ask, if ping checks are not configured, are:
* Do _any_ VMs have the "network" section in `/dev/shm/libvirt_instances.json`?  If so, security rules for the VM in question may be the cause.
* Does the command `ip netns list |grep qrouter` produce any output on the compute node?  If not, perhaps Neutron is not configured in DVR mode, or no VMs are present on that compute node.
* Is the `ping_check` command defined `/etc/monasca/agent/conf.d/libvirt.yaml`?  If not, try running `monasca-setup -d libvirt` as root from within the appropriate Python virtual environment

## VM Dimensions
All metrics include `resource_id` and `zone` (availability zone) dimensions.  Because there is a separate set of metrics for the two target audiences (VM customers and Operations), other dimensions may differ.

| Dimension Name | Customer Value            | Operations Value        |
| -------------- | ------------------------- | ----------------------- |
| hostname       | name of VM as provisioned | hypervisor's hostname   |
| zone           | availability zone         | availability zone       |
| resource_id    | resource ID of VM         | resource ID of VM       |
| service        | "compute"                 | "compute"               |
| component      | "vm"                      | "vm"                    |
| device         | name of net or disk dev   | name of net or disk dev |
| port_id        | port ID of the VM port    | port ID of the VM port  |
| tenant_id      | (N/A)                     | owner of VM             |

## Aggregate Metrics

In addition to per-instance metrics, the Libvirt plugin will publish aggregate metrics across all instances.

| Name                            | Description                                        |
| ------------------------------- | -------------------------------------------------- |
| nova.vm.cpu.total_allocated     | Total CPUs allocated across all VMs                |
| nova.vm.disk.total_allocated_gb | Total Gbytes of disk space allocated to all VMs |
| nova.vm.mem.total_allocated_mb  | Total Mbytes of memory allocated to all VMs     |

Aggregate dimensions include hostname and component from the Operations Value column above.

# License
(C) Copyright 2015-2016 Hewlett Packard Enterprise Development Company LP
