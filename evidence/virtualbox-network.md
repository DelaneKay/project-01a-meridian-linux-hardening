# VirtualBox network settings evidence

These screenshots document the adapter configuration used for the local two-VM lab. They complement the guest-level addresses recorded in the project; they do not prove guest routes, IP forwarding, or successful connectivity by themselves.

## Admin VM: `mds-admin-01`

Adapter 1 was attached to NAT but disabled. Adapter 2 was enabled as a Host-only Adapter named `VirtualBox Host-Only Ethernet Adapter`. Adapter 3 was enabled on the Internal Network named `mds-sec-lab`.

![Admin VM Adapter 1: NAT disabled](virtualbox-admin-nat-disabled.png)

![Admin VM Adapter 2: Host-only network](virtualbox-admin-host-only.png)

![Admin VM Adapter 3: mds-sec-lab internal network](virtualbox-admin-internal-network.png)

The host-only adapter provided the management path from Windows/WSL to the admin VM at `192.168.56.101`. The internal adapter connected the admin VM to the target-side lab network; the project records the admin's internal address as `192.168.77.20`.

## Target VM: `mds-ubuntu-server-01`

Adapter 1 was attached to NAT but disabled. Adapter 2 was enabled on the same `mds-sec-lab` Internal Network as the admin VM. The project records the target address as `192.168.77.10`.

![Target VM Adapter 1: NAT disabled](virtualbox-target-nat-disabled.png)

![Target VM Adapter 2: mds-sec-lab internal network](virtualbox-target-internal-network.png)

This leaves the target attached to the internal lab network, without a VirtualBox NAT or host-only adapter. It was reachable from the admin VM on the lab network, while the laptop's host-only connection terminated at the admin VM.

## Relationship to the induction ZIP

The transferred `mds-induction.zip` documents a different preflight requirement: exactly one active adapter on the admin/control VM, with its only IPv4 address set to `192.168.77.20`. The screenshots show two enabled admin adapters. If these settings were active at deployment time, `lab.py preflight` would reject the admin VM before running the deployment playbook. The ZIP's own `START-HERE.md` says it had not been deployed. The repository therefore treats this ZIP as the artifact copied to the admin VM, not as verified proof of the initial vulnerable workload's deployment.

