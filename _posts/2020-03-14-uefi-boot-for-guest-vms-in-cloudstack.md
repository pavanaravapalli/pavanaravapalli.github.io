---
layout: post
title: "UEFI boot for guest VMs in Apache CloudStack"
date: 2020-03-14 00:30:00 +0530
categories: [cloudstack]
---

Apache CloudStack 4.14 added a boot type and a boot mode for guest VMs, so an instance can start with UEFI firmware instead of legacy BIOS on KVM and VMware. This post covers why that mattered, how it works, and what to check when you use it.

## Why UEFI

UEFI is the firmware interface that initialises the hardware and hands control to the operating system. It replaced BIOS, which has hard limits: an MBR partition table addresses at most 2 TiB per disk, and BIOS has no support for Secure Boot.

Some guests now expect UEFI. Windows 11, for example, requires a UEFI system that supports Secure Boot. Most Linux distributions boot under either BIOS or UEFI, but Secure Boot is only available with UEFI.

## The gap

KVM/QEMU (through OVMF) and VMware could already boot guests with UEFI firmware, but CloudStack had no way to request it. VMs were always defined for BIOS, so using UEFI meant changes outside the orchestrator.

The fix is to make firmware a property of the VM that CloudStack records and applies.

## How it works

Two optional parameters on `deployVirtualMachine` select the firmware:

| Parameter | Values | Notes |
|---|---|---|
| `boottype` | `UEFI`, `BIOS` | Omit for BIOS |
| `bootmode` | `Legacy`, `Secure` | Applies only when the boot type is UEFI. BIOS VMs are always Legacy |

A few design points:

- **Stored as VM details.** The boot type and mode are saved with the instance (in `user_vm_details`), so they apply on every start and cannot be changed after deployment.
- **Host capability is tracked.** A host's UEFI capability is detected when the host is added and recorded as `Host.Uefi.Enable` in `host_details`.
- **KVM.** The hypervisor host needs the `ovmf` or `edk2-ovmf` package. The agent reads its firmware settings from `/etc/cloudstack/agent/uefi.properties`, and CloudStack adds the matching loader and NVRAM definitions to the libvirt domain XML.
- **VMware.** CloudStack sets the firmware option on the VM specification. For templates deployed as-is (deploy-as-is), `boottype` and `bootmode` are ignored, because the template's own definition is used.
- **User VMs only.** System VMs keep their legacy boot.

The UI exposes the same choice in the Advanced Mode section of the instance deployment wizard.

## Deploying a UEFI instance

With CloudMonkey:

```bash
cmk deploy virtualmachine \
  name=uefi-test \
  zoneid=<zone-id> \
  serviceofferingid=<offering-id> \
  templateid=<uefi-capable-template-id> \
  boottype=UEFI \
  bootmode=Secure
```

## Checking it worked

On a KVM host, confirm the firmware package is installed:

```bash
rpm -q edk2-ovmf     # RHEL, CentOS, Alma, Rocky
dpkg -l ovmf         # Debian, Ubuntu
```

Inside a Linux guest, this directory exists only when the machine booted through UEFI:

```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
mokutil --sb-state   # Secure Boot state
```

On Windows, System Information (`msinfo32`) shows BIOS Mode and Secure Boot State.

## Limitations in the original implementation

These come from the 4.14 design and pull request. Later releases may have changed some of them, so check the documentation for your version.

- **Hosts must be UEFI-capable.** Hosts added before the feature do not gain the capability automatically.
- **Templates must match.** CloudStack does not check that the guest image and the chosen firmware agree. A UEFI instance needs a template or ISO installed for UEFI.
- **Fixed at deploy time.** You cannot convert an instance between BIOS and UEFI afterwards.
- **Secure Boot on KVM** needs a q35 machine type and a host with SMM support.
- **Migration.** UEFI instances could not be migrated to another host in the first release.

## References

- Pull request: [apache/cloudstack#3638, "UEFI Support on CloudStack"](https://github.com/apache/cloudstack/pull/3638) (merged March 2020, milestone 4.14.0.0)
- [deployVirtualMachine API reference](https://cloudstack.apache.org/api/apidocs-4.19/apis/deployVirtualMachine.html)
- [KVM host installation guide](https://docs.cloudstack.apache.org/en/latest/installguide/hypervisor/kvm.html)
- [Design document: Enable UEFI booting for Instance](https://cwiki.apache.org/confluence/display/CLOUDSTACK/Enable+UEFI+booting+for+Instance)
