# VirtualBox network settings

This written record summarizes the adapter settings used for the local two-VM lab. It documents VirtualBox attachments, not guest routes, IP forwarding, or successful connectivity by themselves.

## Admin VM: `mds-admin-01`

Adapter 1 was attached to NAT but disabled. Adapter 2 was enabled as a Host-only Adapter named `VirtualBox Host-Only Ethernet Adapter`. Adapter 3 was enabled on the Internal Network named `mds-sec-lab`. Adapter 4 was not documented.

The host-only adapter provided the management path to the admin VM at `192.168.56.101`. The internal adapter placed the admin VM on the target-side lab network. The induction archive expects the control node to use `192.168.77.20`, but that guest address is not confirmed by the saved adapter information.

## Target VM: `mds-ubuntu-server-01`

Adapter 1 was attached to NAT but disabled. Adapter 2 was enabled on the same `mds-sec-lab` Internal Network as the admin VM. The Ansible inventory records the target address as `192.168.77.10`; the adapter information does not display the guest address. Adapters 3 and 4 were not documented.

The recorded settings place the admin and target on a named internal network segment and show a separate host-only management adapter on the admin VM. They do not establish the full target adapter configuration or prove that guest-level routing and forwarding were disabled.

## Relationship to the induction ZIP

The transferred `mds-induction.zip` documents a preflight requirement of exactly one active adapter on the admin/control VM, with its only IPv4 address set to `192.168.77.20`. The saved adapter settings show admin Adapters 2 and 3 enabled, so at least two are active in the recorded configuration. If that configuration was in place when deployment was attempted, it would conflict with the package's one-adapter preflight. The ZIP's own `START-HERE.md` says it had not been deployed. The repository therefore records the ZIP as the artifact copied to the admin VM, not as verified proof of how the initial vulnerable workload was created.
