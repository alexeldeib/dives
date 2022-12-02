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

TODO(ace): bring it back


[0]: https://github.com/0xAX/linux-insides
[1]: https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-10.md#first-steps-after-the-start_kernel
[2]: https://github.com/superfly/init-snapshot
[unit files]: https://www.freedesktop.org/software/systemd/man/systemd.unit.html
[cloud init]: https://cloudinit.readthedocs.io/en/latest/index.html
[cloud init stages]: https://cloudinit.readthedocs.io/en/latest/topics/boot.html
