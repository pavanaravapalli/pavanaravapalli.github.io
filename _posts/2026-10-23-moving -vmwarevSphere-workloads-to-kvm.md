---
layout: post
title: "Moving VMware vSphere Workloads to KVM"
date: 2026-10-23 15:00:00 +0530
categories: [vmware, kvm, cloudstack]
---

With changing licensing dynamics and rising operational costs in the enterprise virtualization landscape, migrating workloads from **VMware vSphere (ESXi)** to **KVM** and **Apache CloudStack** has become a top strategic priority for infrastructure teams
.

Having spent 14 years architecting private cloud platforms and virtualization engines
, I’ve seen that successful migrations come down to execution rigor—specifically, proper driver preparation, automated image conversion, and post-migration kernel tuning
.

This playbook walks you through the step-by-step technical pipeline to convert and import running VMDK workloads into an Apache CloudStack / KVM environment with minimal downtime.


### Prerequisites & Migration Architecture
Before touching production disks, ensure your target CloudStack cluster and staging environment meet these requirements:

* **Staging Conversion Host**: A Linux host with qemu-img, virt-v2v, and guestfs-tools installed, connected to both VMware datastores and target CloudStack primary/secondary storage.

* **Apache CloudStack**: Version 4.16+ (4.18+ recommended for enhanced Volume Import APIs and KVM features).

* **Target Hypervisor**: KVM on RHEL/Rocky Linux or Ubuntu with virtio support enabled in the host kernel.

![Alt text describing the image]({{"/assets/images/vmware_cloudstack_nfs_migration_diagram.png" | relative_url}})


## Phase 1: Pre-Migration Preparation (Guest OS Readiness)
The single biggest cause of boot failures after migrating from ESXi to KVM is missing storage (virtio-blk / virtio-scsi) or network (virtio-net) drivers in the guest OS.

#### 1. Linux Guests (RHEL/CentOS/Ubuntu/Debian)
   Ensure the virtio kernel modules are included in the initramfs / initrd before shutting down the VM on ESXi:

        # Check if virtio drivers are present in dracut / initramfs (RHEL/CentOS)
        lsinitrd /boot/initramfs-$(uname -r).img | grep -i virtio
        
        # If missing, add virtio drivers to dracut configuration:
        echo 'add_drivers+=" virtio virtio_pci virtio_ring virtio_net virtio_scsi virtio_blk "' > /etc/dracut.conf.d/virtio.conf
        
        # Rebuild initramfs
        dracut -f -v /boot/initramfs-$(uname -r).img $(uname -r)
#### 2. Windows Guests
   * Install the latest Fedora VirtIO Drivers (virtio-win.iso) on the Windows VM while it is still running on VMware.
   * Uninstall VMware Tools prior to final cutover to prevent driver conflicts on first boot under KVM.

## Phase 2: Disk Export & Format Conversion Pipeline
There are two primary methods for converting .vmdk files to .qcow2 or .raw formats for CloudStack: virt-v2v (automated driver injection + conversion) or qemu-img (fast direct disk conversion).
    
#### Method A: Automated Conversion via virt-v2v (Recommended for Complex VMs)
   virt-v2v automatically inspects the guest OS, installs missing VirtIO drivers, removes VMware Tools, and outputs a ready-to-boot qcow2 image:
        
        # Convert directly from vCenter ESXi host to QCOW2 format
        virt-v2v -ic vpx://vcenter.domain.internal/Datacenter/Cluster?no_verify=1 \
        -o local -os /mnt/staging_storage \
        -of qcow2 \
        -bridge cloudbr0 \
        "prod-app-vm01"

#### Method B: Direct Conversion using qemu-img (Fastest for Simple/Pre-Prepared VMs)
   If you have already injected VirtIO drivers, export the monolithic VMDK or copy it from the datastore and convert directly:

            # Convert stream-optimized or flat VMDK to QCOW2 with zlib compression
            qemu-img convert -p -f vmdk -O qcow2 -c \
            /mnt/vmware_tmp/prod-app-vm01.vmdk \
            /mnt/staging_storage/prod-app-vm01.qcow2

            # Verify converted QCOW2 virtual size and integrity
            qemu-img info /mnt/staging_storage/prod-app-vm01.qcow2

## Phase 3: Importing Volumes & Instances into Apache CloudStack
Once your converted .qcow2 image is residing on CloudStack secondary storage or a accessible HTTP location:

### 1. Register Image as a Template or Volume in CloudStack
   In the CloudStack UI or via CloudMonkey CLI:

    Navigate to: Images -> Templates -> Register Template (or Volumes -> Upload Volume).
    Hypervisor: Select KVM.
    Format: Select QCOW2.
    OS Type: Choose the matching guest OS type (e.g., Ubuntu 22.04 LTS (64-bit) or Windows Server 2022 (64-bit)).
### 2. Provision / Attach to New Instance
   * Deploy a new Instance using the registered template, ensuring the Service Offering matches your required CPU/RAM allocations.
   * Alternatively, attach the uploaded Volume to a blank VM instance shell.

## Phase 4: Post-Migration Validation & Tuning

   After the VM boots up under KVM in Apache CloudStack, execute these post-migration checks:

1. **Verify VirtIO Disk Controller:** Confirm the disk is bound to virtio-scsi or virtio-blk rather than IDE emulation for optimal IOPS performance.
2. **Network Interface Binding:** Verify that network interface names (e.g., eth0 -> ens3 / enp1s0) have updated cleanly and cloudbr0 bridges are routing traffic.
3. **Storage I/O Tuning:** On KVM hosts, ensure IOThreads are configured for heavy database or transaction workloads:
     
        <!-- Libvirt domain XML snippet configured automatically by CloudStack -->
           <driver name='qemu' type='qcow2' cache='none' io='native' iothread='1'/>


### Field Notes & Gotchas
        💡 1. EFI vs. BIOS Boot Modes: 
            If the VMware VM was configured with EFI firmware (nvram), 
            ensure you select an EFI-enabled KVM Template in CloudStack, 
            or update the libvirt guest XML definition accordingly.

        💡 2. MAC Address Preservation:
            If your application licensing relies on static MAC addresses, 
            specify the original VMware MAC address when attaching the network interface (NIC) in CloudStack.

        💡 3. Storage Sparse Allocation: 
            Always use -c (compression) or sparse flags during qemu-img conversions 
            to avoid copying unallocated zero-blocks across network links.
