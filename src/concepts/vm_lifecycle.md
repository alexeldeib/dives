# VM lifecycle

I suggest reading [linux insides][0] for a deep dive into early
Linux initialization. We most care about things happening [after
kernel][1] initialization when PID 1 starts.

## Init

The kernel starts /sbin/init as the first process. The init process
is responsible for early filesystem configuration and becomes the
ancestor of all new processes (for example, reaping zombies).

Many modern systems use systemd as init. This decision has a lot
of historical controversy. Systemd is big and complex. An init
system does not need to be. Fly.io has an open source example 
of their [init process][2]. It can be an informative example.

## Systemd 

Systemd manages long lived services, consuming configuration in 
[unit files][unit files] and inferring a dependency hierarchy
from the unioned configuration of all units. 

Systemd offers helpers to view the critical chain for any unit,
or for the entire boot cycle:

```bash
 # systemd-analyze critical-chain
The time when unit became active or started is printed after the "@" character.
The time the unit took to start is printed after the "+" character.

graphical.target @20.972s
└─multi-user.target @20.972s
  └─docker.service @16.939s +4.032s
    └─network-online.target @16.910s
      └─systemd-networkd-wait-online.service @270ms +16.639s
        └─systemd-networkd.service @225ms +42ms
          └─systemd-udevd.service @197ms +27ms
            └─systemd-tmpfiles-setup-dev.service @190ms +5ms
              └─systemd-sysusers.service @184ms +5ms
                └─systemd-remount-fs.service @171ms +8ms
                  └─systemd-journald.socket @155ms
                    └─system.slice @107ms
                      └─-.slice @107ms
```

It can also output this data as an svg image viewable in a browser.

This data can help debug slow dependencies during VM boot.

Here's another example. This one was on a VM in Azure which was "preprovisioned".

```bash
# systemd-analyze critical-chain
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.


graphical.target @16min 55.434s
└─multi-user.target @16min 55.429s
  └─ephemeral-disk-warning.service @16min 55.339s +78ms
    └─cloud-config.service @16min 52.066s +3.266s
      └─basic.target @16min 51.683s
        └─sockets.target @16min 51.681s
          └─lxd.socket @16min 51.667s +6ms
            └─sysinit.target @16min 51.600s
              └─cloud-init.service @16min 44.344s +7.076s
                └─systemd-networkd-wait-online.service @16min 43.303s +1.033s
                  └─systemd-networkd.service @16min 42.924s +369ms
                    └─network-pre.target @16min 42.920s
                      └─cloud-init-local.service @2.504s +16min 40.410s
                        └─hv-kvp-daemon.service @2.500s
                          └─sys-devices-virtual-misc-vmbus\x21hv_kvp.device @2.496s
```

Notice that the VM was blocked for ~15 minutes between hv-kvp-daemon and cloud-init-local.
This is how preprovisioning works: it creates a dummy VM and then blocks early VM configuration
on a host service integration. When the block is released, the VM continues to boot and configure
normally with cloud init, overwriting the "dummy" configuration from the preprovisioned VM.

(Okay, Ace waved some hands, he doesn't 100% understand the magic.)

Here are a few more examples including cloud-final.service, which is the cloud init stage responsible
for writing files to disk, walinuxagent.service which runs VM extensions and reports provisioning state.

```bash
# systemd-analyze critical-chain cloud-final.service
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.

cloud-final.service +11.703s
└─multi-user.target @33.929s
  └─ephemeral-disk-warning.service @33.896s +29ms
    └─cloud-config.service @31.392s +2.499s
      └─basic.target @31.204s
        └─paths.target @31.188s
          └─update_certs.path @31.185s
            └─sysinit.target @31.146s
              └─cloud-init.service @10.912s +20.216s
                └─cloud-init-local.service @2.528s +6.098s
                  └─systemd-remount-fs.service @1.453s +35ms
                    └─systemd-journald.socket @1.403s
                      └─system.slice @1.387s
                        └─-.slice @1.359s
```

```bash
# systemd-analyze critical-chain walinuxagent.service
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.

walinuxagent.service @31.385s
└─basic.target @31.204s
  └─paths.target @31.188s
    └─update_certs.path @31.185s
      └─sysinit.target @31.146s
        └─cloud-init.service @10.912s +20.216s
          └─cloud-init-local.service @2.528s +6.098s
            └─systemd-remount-fs.service @1.453s +35ms
              └─systemd-journald.socket @1.403s
                └─system.slice @1.387s
                  └─-.slice @1.359s
```

## Cloud init

[Cloud init][cloud init] is a standard cross-distro tool for instance initialization
and first boot configuration. It has multiple [stages][cloud init stages], each with different responsibilities:

- Local: blocks all further boot until correct datasource/config has been applied.
 Otherwise, an instance may come up with stale configuration.
- Network: This stage processes user data, including resolving it via network, meaning it requires networking to be configured. It also runs bootcmds, and may handle disk and filesystem configuration depending on platform.
- Config: This stage runs other configuration modules and modules which have few other dependencies, such as runcmd.
- Final: This stage writes all late-stage user-configuration to disk, including write_files, apt packages, or user configuration on the machine.

## Observations

Let's bring this back to Azure and Kubernetes.

In the [status quo](../node/status_quo.md) doc, we summarized the high level process of node provisioning.
Now we can return to that in a bit more depth.

Each nodepool in each of the thousands of AKS clusters may require slightly different configuration.
Some configuration details are per-nodepool, some are per-cluster, some per Kubernetes version, etc.

AKS has a few tools to customize nodes:
- We build custom VM images. Since we don't know specific details, there are limits to what we can do here.
  - We can build VM images for specific scenarios to optimize e.g. latency, but this comes with maintenance cost and COGS.
- We can use cloud init for first boot customization of Linux nodes. This can consume cluster-specific data.
  - Cloud init (the agent) consumes configuration from a disk/file mounted to the VM by the Azure platform.
  - We can also read this file directly. It used to be possible to retrieve customData via IMDS but this was disabled for security reasons.
- We can use Azure VM extensions to configure node. Notably, the custom script extension (CSE) can deliver arbitrary bash payloads. This can also consume cluster-specific data.
- We can use userData and retrieve the data via IMDS (yes, I just said this was disabled, but it's back!).

We use custom VM images, cloud init, and CSE today.

### How does this impact latency?

Our VM images cache binaries and images for many Kubernetes versions,
but we do not know cluster details like Kubernetes version until runtime.
Today we rely on CSE to configure and start Kubelet. CSE itself depends on 
waagent. Waagent depends on several early cloud-init stages. If we knew
which version of kubelet we needed, we could start it as a systemd unit in
the VM image. This would likely take a dependency on network-online.target
(for connectivity to API server), but would remove any cloud init, CSE, or 
waagent dependency. In this particular case we depend on cloud-final,
which above adds 11 seconds of latency. network-online also precedes
cloud-config, removing another 3 seconds.

### Summary and Goals

To recap:
- We want to provision Kubernetes nodes as fast as possible for a good user experience.
- We have some data we can't possibly know until the user creates a cluster.
- We want to get that data onto a VM and process it as quickly as possible, so we can finish provisioning.

Today, we do a lot of things with cloud init and CSE.
Some of these are truly necessary. Some are not.

To meet the goals above:
- We should do as much as possible in the VM image, like starting the correct version of Kubelet.
- We should deliver runtime configuration using either customData directly from disk or userData.
  - We already depend on cloud init to process customData, so I suggest using userData via IMDS.
  - Additionally, userData may be changed on a live VM in VMSS without reimage, and queried via IMDS for updates.
  - This facilitates in-place update, but we need a signalling mechanism (or else we need to resort to polling).

In practice this means refactoring many of our scripts, moving things we do unnecessarily
at runtime into the VM image, and building more VM images to optimize more scenarios.

The in-place update point is also key. More on that later...

[0]: https://github.com/0xAX/linux-insides
[1]: https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-10.md#first-steps-after-the-start_kernel
[2]: https://github.com/superfly/init-snapshot
[unit files]: https://www.freedesktop.org/software/systemd/man/systemd.unit.html
[cloud init]: https://cloudinit.readthedocs.io/en/latest/index.html
[cloud init stages]: https://cloudinit.readthedocs.io/en/latest/topics/boot.html
