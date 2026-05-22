# VCTA – VMware Technical Associate: vSphere 8

## Table of Contents

- [Section 1: Introduction to vSphere](#section-1-introduction-to-vsphere)
- [Section 2: Changes to VMware under Broadcom](#section-2-changes-to-vmware-under-broadcom)
- [Section 3: Configure and Manage vSphere and ESXi 8](#section-3-configure-and-manage-vsphere-and-esxi-8)
- [Section 4: Configure and Manage vSphere 8 Networking](#section-4-configure-and-manage-vsphere-8-networking)
- [Section 5: Configure and Manage vSphere 8 Storage](#section-5-configure-and-manage-vsphere-8-storage)
- [Section 6: Deploy and Administer vSphere 8 VMs](#section-6-deploy-and-administer-vsphere-8-vms)
- [Section 7: Using vMotion to Migrate vSphere 8 VMs](#section-7-using-vmotion-to-migrate-vsphere-8-vms)
- [Section 8: Administer Availability in vSphere 8](#section-8-administer-availability-in-vsphere-8)
- [Section 9: Administer Resource Management Features in vSphere 8](#section-9-administer-resource-management-features-in-vsphere-8)

---

## Section 1: Introduction to vSphere

> ### Lesson 1: Introduction to Virtualization

#### What is a Virtual Machine (VM)?

A Virtual Machine is very similar to a physical computer — it has an operating system (Windows, Linux, etc.) installed on it, but it does not have dedicated physical hardware. The OS running inside a VM is completely unaware that it's running in a virtualized environment.

Key characteristics of a VM:
- Has its own independent operating system
- Uses virtual hardware (virtual CPU, virtual memory, virtual network adapters)
- Runs alongside other VMs on the same physical host
- Shares the underlying physical resources of the host

---

#### What is an ESXi Host?

An ESXi Host is the physical server on which virtual machines run. Instead of installing a regular OS on this server, you install a Hypervisor called ESXi.

- ESXi is VMware's Type 1 Hypervisor (runs directly on bare metal hardware)
- It allows multiple VMs to run simultaneously on the same physical server
- Physical resources shared among VMs include: CPU, Memory, Network, and Storage

---

#### What is a Hypervisor?

A Hypervisor is software that enables multiple virtual machines to run on a single physical host at the same time.

| Type | Description | Example |
|------|-------------|---------|
| Type 1 | Runs directly on physical hardware (bare metal) | ESXi |
| Type 2 | Runs on top of a host OS | VMware Workstation |

ESXi is a Type 1 Hypervisor — it installs directly onto the physical server.

---

#### Layer of Abstraction

The Hypervisor adds a Layer of Abstraction — a software layer sitting between the VMs and the physical hardware. This layer:
- Hides the actual physical resources from the VMs
- Presents virtualized versions of resources (vCPU, vRAM, vNIC, etc.)
- Intelligently allocates resources based on VM needs and priorities

Analogy: Think of covering a bowl of M&Ms with a napkin. The kids (VMs) can't see the actual candy (physical resources). You (the Hypervisor) decide how much each one gets.

---

#### Resource Allocation Example

| Virtual Machine | Allocated Memory |
|----------------|-----------------|
| VM 1 (high priority) | 4 GB |
| VM 2 (low priority) | 2 GB |

---

#### Oversubscription

Oversubscription is the practice of assigning more virtual resources than the available physical resources — because not all VMs need 100% of their resources 100% of the time.

Example: 12 VMs on a host with only 4 physical CPUs. This works because not all VMs are busy at the same time, and ESXi intelligently schedules CPU time across VMs.

Analogy: You expect 200 restaurant customers per day, but they come in waves — so you only need 50 seats. Building for 200 simultaneous guests would waste resources.

The goal is to make the most efficient use of physical resources while still delivering good performance.

---

> ### Lesson 2: The Four Food Groups of VMs

The four core resources that ESXi provides to virtual machines are: CPU, Memory, Storage, and Networking.

---

#### Food Group 1: CPU

Each VM is configured with a number of virtual CPUs (vCPUs), which represent how many physical processor cores that VM can potentially use.

- A VM with 4 vCPUs can utilize up to 4 physical processors
- A VM with 2 vCPUs can utilize up to 2 physical processors
- Multiple VMs share physical CPUs by taking turns — managed by the Hypervisor

Balancing VMs per Host:
- Too many VMs → CPU contention → degraded performance (VMs wait for CPU time)
- Too few VMs → wasted physical resources (idle CPUs)
- Goal: find the right balance to maximize utilization without hurting performance

---

#### Food Group 2: Memory

VMs are allocated virtual memory (vRAM). Like CPU, memory can be oversubscribed.

Example:

| Config | Value |
|--------|-------|
| Number of VMs | 12 |
| Memory per VM | 4 GB |
| Total virtual memory promised | 48 GB |
| Actual physical RAM on host | 32 GB |

This works because not all VMs need their full memory allocation at the same time. The Hypervisor maps memory pages from the physical RAM to whichever VMs need it at a given moment. Oversubscribing memory too heavily will degrade performance — find the right balance.

---

#### Food Group 3: Networking

Virtual networking introduces a new component: the vSphere Standard Switch (virtual switch).

How it works:
1. A virtual network adapter is created on the VM
2. That adapter connects to a virtual switch (vSwitch) on the ESXi Host
3. VMs connected to the same vSwitch can communicate directly without traffic leaving the host
4. For external communication (other hosts, physical servers), traffic flows through a physical network adapter (pNIC) on the host

```
VM1 --+
VM2 --+--> vSwitch --> Physical NIC --> External Network
VM3 --+
       (VMs can also communicate with each other internally through the vSwitch)
```

---

#### Food Group 4: Storage

Each VM has virtual disks to store its guest operating system (Windows, Linux, etc.) and data.

How storage I/O works:
1. The guest OS generates SCSI commands
2. The Hypervisor receives and redirects those commands to the appropriate physical storage device

Supported storage types:
- Fibre Channel storage arrays
- iSCSI storage arrays
- NFS (Network File System)
- Local storage inside the ESXi Host

---

> ### Lesson 3: VM Files and Live State

#### Virtual Machine Files

When a VM is powered off, it is not running on any ESXi Host — but it does not disappear. It exists as a set of files stored on a datastore. The two most critical files are:

| File | Description |
|------|-------------|
| VMDK | The virtual disk of the VM — stores the OS, applications, and data |
| VMX | The configuration file — defines how much CPU, memory, and other resources the VM gets |

---

#### What is a Datastore?

A datastore is any storage solution that has been properly configured to store Virtual Machine Files. Options include:
- Local storage inside the ESXi Host
- iSCSI
- Fibre Channel
- vSAN
- Virtual Volumes (vVols)
- NFS

---

#### Live State of a Virtual Machine

When a VM is powered on and running, it has a Live State — everything that is happening in that VM at a given moment. This includes:
- Active memory pages on the physical host
- CPU scheduling on the physical processors
- Network activity through the virtual switch (vSwitch or Distributed Switch)
- Storage I/O through the virtual SCSI controller

All of these are intercepted and managed by the Hypervisor, which grants the VM access to the underlying physical resources.

The concept of Live State becomes important later when discussing vMotion — the ability to migrate a running VM from one host to another without powering it off or interrupting its operation.

---

#### VM Registration

Even when a VM is powered off, it remains registered to a specific ESXi Host. When powered on, it will start on that registered host using the files from its datastore. VMs can be moved to different hosts when needed.

---

> ### Lesson 4: SDDCs and the Cloud

#### Software Defined Datacenter (SDDC)

ESXi Hosts are the building blocks of a Software Defined Datacenter. An SDDC is made up of:
- Multiple ESXi Hosts running many virtual machines
- Physical network adapters providing network access to VMs
- Storage hardware providing volumes for VM operating systems and applications

The key distinction between a physical datacenter and an SDDC is that an SDDC is not defined by physical walls or location. It is defined entirely by software. This means:
- Multiple SDDCs can exist within the same physical building
- Multiple SDDCs can share the same physical networking and storage hardware
- An SDDC can span multiple physical facilities
- You define the boundaries of an SDDC — the hardware does not

---

#### Cloud Computing Models

Traditionally, organizations built their own physical datacenters — handling power, cooling, rack space, hosts, networking, and storage themselves. Cloud computing offers an alternative.

**Private Cloud**

A virtualized environment that you own and manage yourself, running on-premises with your own physical hardware. You are responsible for all infrastructure.

**Public Cloud**

A model where a third-party provider owns and manages all physical infrastructure. You simply consume the resources.

Analogy: Building a house and powering it yourself (solar panels, generator, batteries) versus just calling the power company. The Public Cloud is the utility company — they handle everything physical, you just consume the service.

Examples of Public Cloud providers:
- VMware Cloud on AWS
- Amazon Web Services (AWS)
- Microsoft Azure
- Google Cloud

**Hybrid Cloud**

The most common real-world model. A mix of Private and Public Cloud where workloads run in both environments simultaneously, often with integration between them such as a dedicated physical connection.

---

> ### Lesson 5: Types of Hypervisors

#### Type 1 Hypervisor (Bare Metal)

A Type 1 Hypervisor is installed directly on the physical hardware of the host, in the same way an operating system would be. ESXi is the primary example used in this course.

Characteristics:
- Acts as a lightweight operating system
- Has direct access to the physical resources of the host
- Isolated from guest operating systems, making it highly secure and difficult to attack
- Delivers better performance than Type 2

---

#### Type 2 Hypervisor (Hosted)

A Type 2 Hypervisor runs as a software application on top of an existing operating system. VMware Workstation is a common example.

How it works:
1. A standard OS (e.g. Windows) is installed on the physical hardware
2. The OS has direct access and control of the physical resources
3. The Type 2 Hypervisor is installed on top of that OS like any other application
4. Virtual machines run inside the Hypervisor software

The guest VMs can still run any operating system regardless of what the host OS is — for example, Linux VMs can run inside VMware Workstation on a Windows machine.

---

#### Type 1 vs Type 2 Comparison

| Feature | Type 1 (Bare Metal) | Type 2 (Hosted) |
|---------|-------------------|----------------|
| Installed on | Physical hardware directly | On top of a host OS |
| Resource access | Direct | Through the host OS |
| Performance | Higher | Lower (extra latency) |
| Security | More isolated and secure | Less isolated |
| Example | ESXi | VMware Workstation |
| Use case | Enterprise production environments | Labs, experimentation, home use |

---

#### Why Type 2 Has Higher Latency

With a Type 2 Hypervisor, a VM requesting CPU resources must pass through two layers of software — the Hypervisor and the host OS — before reaching the physical hardware. Each additional software layer introduces latency. In an enterprise environment, this overhead is unacceptable, which is why Type 1 Hypervisors are the standard.

---

> ### Lesson 6: Introduction to vCenter 8

#### What is vCenter?

vCenter is the central management system for all vSphere resources. It provides a single point of control for managing ESXi Hosts, Virtual Switches, Datastores, and Virtual Machines across an entire vSphere environment.

---

#### What vCenter Manages

- ESXi Hosts
- Virtual Machines
- Virtual Switches
- Datastores
- Virtual Datacenters (Software Defined Datacenters)

---

#### vCenter Server Appliance (VCSA)

vCenter runs exclusively as a pre-configured, Linux-based virtual appliance called the vCenter Server Appliance (VCSA). There is no longer a Windows version of vCenter.

Key components of the VCSA:
- Operating System: Photon OS
- Embedded built-in database storing all vCenter configuration data
- Various services that make up the functional components of vCenter
- vSphere Lifecycle Manager — provides automated patch and version management for ESXi Hosts

The VCSA runs as a virtual machine on one of the ESXi Hosts it manages. This means vCenter is managing the very host it runs on, which is by design.

---

#### Do You Need vCenter?

Technically, ESXi Hosts can run virtual machines without vCenter, managed individually through the Host Client. However, many of the most important vSphere features require vCenter:

| Feature | Description | Requires vCenter |
|---------|-------------|-----------------|
| vMotion | Migrate a running VM from one host to another | Yes |
| Storage vMotion | Migrate a running VM from one datastore to another | Yes |
| High Availability (HA) | Automatically restart VMs on a new host if their host fails | Yes |
| Fault Tolerance (FT) | Zero-downtime failover for VMs | Yes |
| DRS | Automatically balance VM load across hosts | Yes |
| Storage DRS | Automatically balance VM load across datastores | Yes |

---

#### vCenter as the Management Plane

vCenter is the management plane of the vSphere environment. This means:
- It gives administrators the ability to make configuration changes
- If vCenter fails, running VMs are not affected — they continue running on their ESXi Hosts
- Features like HA will continue to function even if vCenter is down
- A vCenter failure only means loss of the ability to make new configuration changes

---

#### Clients for Connecting to vCenter

**vSphere Client (HTML5)**
The only supported client for managing vCenter. It is a web-based interface accessed through a browser. This should be used for all configuration changes once vCenter is up and running.

**Host Client**
Used to connect directly to a single ESXi Host. Use cases:
- Setting up the very first ESXi Host before vCenter exists
- Troubleshooting when vCenter is down

Changes made through the Host Client are made outside of vCenter and should be avoided in normal operation.

---

#### vCenter High Availability

The vCenter Server Appliance supports native High Availability without a load balancer. It maintains an active appliance and a standby appliance, so that if the active appliance fails, the standby takes over and vCenter remains available.

---

> ### Lesson 7: vCenter 8 Enhanced Linked Mode

#### The Platform Services Controller (PSC) — History

In older versions of vSphere, a separate component called the Platform Services Controller (PSC) existed to provide shared services across multiple vCenter instances. These services included:
- Licensing management
- Certificate management
- Authentication via Single Sign-On (SSO)

The idea was that multiple vCenter servers could share one PSC, allowing centralized authentication and licensing. While useful, the external PSC introduced significant complexity — for example, making it highly available required additional configuration.

---

#### PSC in vSphere 7 and 8

As of vSphere 7, the external PSC no longer exists. All PSC functions are now embedded directly into the vCenter Server Appliance. This applies to vSphere 8 as well.

Benefits of this change:
- Reduced complexity — only one appliance to manage, update, and back up
- No Windows license or external database required
- No load balancer needed for vCenter High Availability
- Shared services like Single Sign-On are still available but built into the VCSA

---

#### Enhanced Linked Mode

Enhanced Linked Mode allows multiple vCenter Server instances to be connected together, giving administrators a single unified view and a single login across all of them.

Use cases for having multiple vCenter instances:
- One vCenter for server VMs, another for virtual desktops
- Separate vCenter per organizational unit
- Separate vCenter for an acquired company

With Enhanced Linked Mode:
- Up to 15 vCenter Server Appliances can be joined together
- A Single Sign-On (SSO) domain spans all linked vCenter instances
- Authenticating to one vCenter automatically grants access to view and manage the inventory of all linked vCenter servers

---
## Section 2: Changes to VMware under Broadcom

This section covers the major changes to VMware and vSphere following the acquisition by Broadcom. Key topics include:
- The shift from perpetual licenses to a subscription-based licensing model
- Changes to vSphere packaging and product lines
- Updates to the VMware certification process

---

> ### Lesson 1: Changes to the Purchasing Model under Broadcom

#### From Perpetual to Subscription Licensing

The most significant change under Broadcom is the shift in how vSphere is licensed. Previously, VMware used a perpetual licensing model — you purchased a license once and owned it forever, with support purchased separately. Under Broadcom, vSphere has moved to a subscription-based model.

| Model | Description |
|-------|-------------|
| Perpetual (old) | One-time purchase, license lasts forever, support purchased separately |
| Subscription (new) | Ongoing payments for a chosen duration, license expires when subscription ends |

---

#### Frequently Asked Questions on Licensing

**Can I still use my perpetual license for vSphere 8 updates?**
Yes. If you hold a current perpetual license, you can continue performing vSphere 8 updates without purchasing a subscription. However, VMware is moving toward subscriptions, so this should factor into future planning.

**What happens when my support contract expires on a perpetual license?**
You can continue using the perpetual license, but you will not be able to renew the support contract. Without an active support contract you lose access to updates, security patches, and support services. At that point, transitioning to a subscription is the expected path.

**Can I convert my perpetual license to a subscription?**
Yes. VMware offers the Subscription Upgrade Program, which allows conversion of perpetual licenses to subscription licenses. An active support contract is typically required to do this. Contact your VMware sales representative for specifics.

---

#### vSphere Licensing Packages

| Package | Key Features | Approx. Price |
|---------|-------------|---------------|
| vSphere Essentials Plus | Up to 3 hosts, max 2 CPUs per host | ~$35/core |
| vSphere Standard | vSphere Standard + vCenter Standard | ~$50/core |
| vSphere Foundation | vSphere Enterprise Plus + vCenter + Tanzu Kubernetes Grid + VMware Cloud Foundation Operations | Higher |
| vSphere Cloud Foundation | vSphere Enterprise Plus + vCenter + vSAN + NSX Networking + HCX | Highest |

Notes on specific packages:
- vSphere Foundation adds intelligent operations management features to improve performance and efficiency
- vSphere Cloud Foundation is the most complete option and includes HCX, which is particularly useful for migrating workloads to the cloud
- Pricing varies — always confirm current pricing with your VMware sales representative

---
## Section 3: Configure and Manage vSphere and ESXi 8

This section covers the installation and management of vSphere 8. Topics include:
- Installing ESXi 8 on a host
- Deploying the vCenter Server Appliance
- Adding hosts to vCenter
- Licensing vCenter and ESXi hosts
- Configuring hosts and creating a content library

---

> ### Lesson 1: Preparing to Install ESXi 8

#### Overall vSphere Installation Order

When building a new vSphere environment from scratch, the process must follow this order:

1. Install ESXi on at least one physical host
2. Deploy the vCenter Server Appliance on that host
3. Use the vSphere Client to organize inventory, add more hosts, license everything, create clusters and Resource Pools

vCenter cannot be deployed before at least one ESXi host exists, because vCenter itself runs as a virtual machine.

---

#### ESXi 8 Hardware Requirements

| Requirement | Minimum |
|-------------|---------|
| CPU cores | At least 2 cores, using a supported processor |
| NX/XD bit | Must be enabled in CPU and BIOS |
| Memory | 8 GB RAM |
| Network | At least one Gigabit Ethernet controller or faster |
| Boot disk | 32 GB of persistent storage (HDD, SSD, or NVMe) |

Booting ESXi from an SD card or USB device is now officially deprecated and should be avoided.

---

#### ESXi Installation Methods

**Interactive Installation**
The simplest method. Boot the physical host from the ESXi installation media and follow the on-screen menu options. Best suited for new environments or small clusters where you are setting up a few hosts.

**Scripted Installation**
ESXi installation is scripted so that the same configuration can be applied to many hosts automatically. Useful in larger environments.

**Auto Deploy (PXE Boot)**
A more advanced method for large environments where hosts are added frequently. How it works:
1. A bare metal host with nothing installed boots up
2. It issues a DHCP request and receives an IP address plus the location of a PXE boot server
3. The PXE boot server feeds the host an ESXi image to boot from
4. ESXi is automatically installed on the host

Auto Deploy allows any new host added to the environment to be provisioned automatically without manual intervention.

---

#### Which Method to Use

| Environment | Recommended Method |
|-------------|-------------------|
| Small or new environments | Interactive Installation |
| Large environments, frequent host additions | Scripted Install or Auto Deploy |

---

> ### Lesson 2: ESXi Configuration Maximums

VMware provides a Configuration Maximums website where you can look up the maximum supported values for any vSphere component or related product. For the certification exam, the following ESXi Host maximums are the most important to memorize.

---

#### ESXi Host Maximums — Exam-Relevant Values

| Resource | Maximum |
|----------|---------|
| Logical CPUs per host | 896 |
| CPUs per NUMA Node | 16 |
| VMs per host | 1024 |
| Memory per host | 24 TB |
| Virtual disks per host | 2048 |

---

#### Additional Maximums (Less Likely on Exam)

- Networking maximums — not typically tested
- Cluster and Resource Pool maximums (e.g. VMs per Cluster) — not typically tested
- Datastore size maximums — covered in the Storage section

---

> ### Lesson 3: Free Hands-On Labs from VMware

VMware provides free hands-on lab exercises at hol.vmware.com. These labs are a great way to practice vSphere tasks without setting up your own home lab.

How to access:
1. Go to hol.vmware.com
2. Create a free account
3. Search the catalog for "vSphere" to find relevant labs
4. Launch any lab directly in the browser

Available labs cover topics such as ESXi Host performance testing, new features in vSphere 7 and 8, and general vSphere walkthroughs. The catalog is updated regularly as VMware adds new content.

This is a good alternative if you prefer not to build a personal home lab, though setting up your own home lab is still highly recommended for deeper hands-on experience.

---

> ### Lesson 4: About My Lab Environment

#### VMUG Advantage

VMUG (VMware User Group) Advantage is a membership program that provides access to 365-day evaluation licenses and downloads for virtually all VMware software. It is the most cost-effective way to build a personal home lab for vSphere practice.

Software available through VMUG Advantage includes:
- ESXi installer
- vCenter Server Appliance
- VMware Workstation Pro
- Site Recovery Manager
- NSX-T
- VMware Horizon
- And much more

---

#### Lab Setup Used in This Course

The instructor uses VMware Workstation Pro as a Type 2 Hypervisor running on a personal computer. ESXi hosts are then installed as virtual machines on top of VMware Workstation, creating a fully functional nested vSphere lab environment without requiring dedicated physical servers.

This approach allows you to:
- Run ESXi and vCenter as VMs on a single laptop or desktop
- Use all software downloaded through VMUG Advantage
- Practice all vSphere tasks covered in this course

A home lab is not required to follow along with this course, but it is strongly recommended for hands-on experience. VMUG Advantage is available at vmug.com.

---

> ### Lesson 5: Demo — Installing ESXi 8 Using Interactive Installation

This lesson walks through the interactive installation of ESXi 8 using VMware Workstation as the lab environment. The same process applies when installing on a physical host.

---

#### Step 1: Create the Virtual Machine (Lab Only)

In VMware Workstation, create a new VM with a custom configuration:
- Attach the ESXi 8 ISO image as the boot media (equivalent to inserting a bootable DVD/USB on a physical server)
- Set the guest OS to "ESXi 7 or later"
- Configure CPU, memory, and disk size
- Add a network adapter (required by ESXi)

---

#### Step 2: Boot from the ISO and Run the Installer

Power on the VM. It will boot from the ISO and launch the ESXi installer.

Key steps in the installer:
1. Press Enter to begin
2. Press F11 to accept the End User License Agreement
3. Select the disk to install ESXi on — the selected disk will be formatted with VMFS-5

If a disk already shows a star (*) next to it, it has already been formatted with VMFS and may contain VM files. Proceed with caution.

4. Choose keyboard layout (US Default)
5. Set a root password
6. Press F11 to confirm and begin installation

---

#### Step 3: Remove Installation Media and Reboot

Once installation completes, disconnect the ISO image (or remove physical boot media) before rebooting — otherwise the installer will run again on next boot.

---

#### Step 4: Initial Configuration via DCUI

After rebooting, the host loads the Direct Console User Interface (DCUI). Press F2 and log in with root credentials.

The DCUI is used for basic initial setup only — not for day-to-day management. Tasks available in the DCUI:

| Task | Description |
|------|-------------|
| Configure Management Network | Set IP address, subnet, gateway, DNS, and VLAN for the management VMkernel port |
| Test Management Network | Run ping tests to verify connectivity to gateway and DNS server |
| Configure Keyboard Layout | Change the keyboard layout if needed |
| Troubleshooting Options | Enable/disable ESXi Shell and SSH access |
| Restart Management Agents | Useful if vCenter cannot communicate with the host |
| View System Logs | Access ESXi logs for diagnostics |
| View Support Information | Shows serial numbers and info useful when working with VMware Support |
| Reset System Configuration | Wipes all configuration and returns ESXi to factory defaults (ESXi remains installed) |

---

#### Management Network Configuration

The IP address configured here is assigned to the Management VMkernel port. This is the address the ESXi host uses to communicate with vCenter. Configure:
- Static IP address: 192.168.1.10
- Subnet mask: 255.255.255.0
- Default gateway: 192.168.1.1
- DNS server: 192.168.1.200

After applying, the management network restarts automatically.

---

#### ESXi Shell and SSH

Both are disabled by default. They can be enabled from Troubleshooting Options in the DCUI when needed for command-line access. Keep them disabled when not in use for security best practices.

- ESXi Shell — local command-line access on the host
- SSH — remote command-line access using a client like PuTTY

---

> ### Lesson 6: Demo — Create a Datastore using the Host Client

Before deploying the vCenter Server Appliance, you need a datastore to store it on. This lesson demonstrates how to add a new disk to an ESXi host and create a datastore using the Host Client.

---

#### Step 1: Add a New Disk to the Host (Lab Only)

In VMware Workstation, add a new virtual disk to the ESXi VM:
1. Right-click the ESXi VM and go to Settings
2. Click Add and choose Hard Disk
3. Select SCSI as the disk type
4. Create a new virtual disk and set the size (e.g. 100 GB)
5. Store as a single file — do not allocate all space now (no need for thick provisioning)

On a physical host, this step is replaced by physically connecting a new disk or storage device.

---

#### Step 2: Connect to the Host Client

Open a browser and navigate to the ESXi host's IP address to reach the Host Client login page. Log in using the root credentials configured during ESXi installation.

---

#### Step 3: Verify the New Disk is Visible

In the Host Client, go to Storage > Devices. The new disk should appear as an unused device.

If the disk is not visible, perform a rescan on the storage adapters. This forces ESXi to search for newly added disks.

---

#### Step 4: Create the Datastore

1. Click New Datastore
2. Give the datastore a name (e.g. 100GBLocal)
3. Select the new disk and choose to use all available space
4. Format the disk with the VMFS file system
5. Click Finish and confirm — all existing data on the disk will be wiped

The datastore is now ready to store virtual machine files, including the vCenter Server Appliance.

---

> ### Lesson 7: Demo — Deploy the vCenter Server Appliance 8

Deploying the vCenter Server Appliance (VCSA) is a two-stage process. Before starting, DNS must be configured for the VCSA.

---

#### Lab Environment IP Layout

| Device | IP Address |
|--------|-----------|
| ESXi(A) (deployment target) | 192.168.1.10 |
| DNS / Domain Controller | 192.168.1.200 |
| vCenter Server Appliance (VCSA) | 192.168.1.100 |

Every device must have a unique IP. The VCSA cannot share an IP with the ESXi host it is being deployed on.

---

#### Pre-requisite: Configure DNS

Before deploying vCenter, create DNS records for the VCSA in both the Forward and Reverse Lookup Zones on your DNS server. The VCSA name in DNS must exactly match the name you will use during installation.

| Zone | Record Type | Name | Value |
|------|------------|------|-------|
| Forward Lookup Zone (lab.local) | A (Host) | vcsa-demo | 192.168.1.100 |
| Reverse Lookup Zone | PTR (Pointer) | 192.168.1.100 | vcsa-demo.lab.local |

This is a critical step — DNS must be configured before installation begins.

---

#### Launching the Installer

Mount the VCSA ISO image and launch the installer (vcsa-ui-installer). The same installer can also be used to:
- Upgrade an existing VCSA
- Migrate from a Windows vCenter Server to a VCSA
- Restore vCenter from backup

Select Install for a fresh deployment.

---

#### Stage 1: Deploy the Virtual Appliance

Stage 1 creates the virtual machine that vCenter will run on.

Key configuration steps:

| Step | Detail |
|------|--------|
| Target host | 192.168.1.10 (ESXi(A)) with root credentials |
| Appliance name | vcsa-demo (must match DNS entry exactly) |
| Root password | Set the root password for the VCSA |
| Deployment size | Tiny (suitable for small lab environments) |
| Datastore | 100GBLocal (created in Lesson 6) |
| Disk mode | Thin Disk Mode enabled |
| Network | VM Network |
| IP assignment | Static |
| FQDN | vcsa-demo.lab.local |
| IP address | 192.168.1.100 |
| Subnet mask | 255.255.255.0 |
| Default gateway | 192.168.1.1 |
| DNS server | 192.168.1.200 |

Deployment size options:

| Size | Use Case |
|------|----------|
| Tiny | Very small lab environments with few hosts and VMs |
| Small | Small production environments |
| Medium | Medium deployments (up to ~400 hosts) |
| Large | Large enterprise deployments |

When Stage 1 completes, the VCSA virtual machine is created and running. A URL is provided for the VAMI (vCenter Server Appliance Management Interface) — this can be used to resume the process if the installer is closed.

---

#### Stage 2: Configure the vCenter Server

Stage 2 configures the vCenter software inside the appliance.

Key configuration steps:

| Setting | Value |
|---------|-------|
| IP address | 192.168.1.100 |
| Subnet mask | 24 (255.255.255.0) |
| Host name (FQDN) | vcsa-demo.lab.local |
| Gateway | 192.168.1.1 |
| DNS servers | 192.168.1.200 |
| Time synchronization mode | Synchronize time with NTP servers |
| NTP server | 192.168.1.200 |
| SSH access | Activated |
| SSO domain | vsphere.local |
| SSO username | administrator |
| CEIP | Opted in |

There is no external Platform Services Controller option in vSphere 7 or 8 — all SSO and related services are embedded in the VCSA.

---

#### Accessing the vSphere Client

Once Stage 2 completes, open a browser and navigate to the VCSA address to reach the vSphere Client login page.

- URL: `https://<vcenter-ip>` or `https://<vcenter-fqdn>` — in this lab: https://192.168.1.100 or https://vcsa-demo.lab.local
- Username: administrator@vsphere.local
- Password: the SSO password set during Stage 2

The first login may take longer than expected while plugins are being enabled. License keys can be applied after login — expired license warnings are normal on a fresh install.

---

> ### Lesson 8: Demo — vCenter 8 VAMI

#### What is the VAMI?

The vCenter Server Appliance Management Interface (VAMI) is used to manage the vCenter Server Appliance itself — not for managing VMs, ESXi hosts, or virtual switches. That is done through the vSphere Client.

| Interface | Purpose |
|-----------|---------|
| vSphere Client | Manage VMs, ESXi hosts, datastores, virtual switches |
| VAMI | Manage the vCenter Server Appliance itself |

**Access URL:** `https://<vcenter-ip>:5480` — in this lab: https://192.168.1.100:5480

Log in using the root credentials set during VCSA installation.

Note: The VCSA runs as a VM on ESXi(A). If ESXi is rebooted, the VCSA will shut down with it and will not come back up automatically unless autostart is configured.

To power vCenter back on manually after an ESXi reboot:
1. Open the ESXi Host Client at `https://<esxi-ip>` — in this lab: https://192.168.1.10
2. Go to Virtual Machines and find the VCSA VM (vcsa-demo)
3. Power it on and wait 5-10 minutes for it to fully boot
4. Access the VAMI again at https://192.168.1.100:5480

To configure autostart (recommended):
1. In the Host Client, go to Manage → Autostart
2. Enable autostart and add the VCSA VM to the autostart list
3. vCenter will now power on automatically every time ESXi reboots

---

#### VAMI Tabs Overview

**Summary**
- Shows running services (e.g. Single Sign-On status)
- Displays the vCenter version and build number
- Shows uptime and overall health status — all items should show green/good

**Monitor**
- View performance characteristics of the VCSA itself
- Includes database space utilization, network performance, and virtual disk metrics

**Access**
- Shows which access methods are enabled or disabled:
  - SSH Login
  - DCUI
  - Console CLI
  - Bash Shell

**Networking**
- View and edit network configuration of the VCSA
- Includes DNS servers, default gateway, hostname, and IP settings
- Click Edit to modify any of these settings

**Firewall**
- Create firewall rules to control which IP ranges can connect to the VCSA
- These rules apply only to the VCSA itself, not to VMs or ESXi hosts
- Example: allow only the 192.168.1.0/24 network and block everything else

**Time**
- View and configure NTP time synchronization
- A green check mark confirms the NTP server is reachable
- The same NTP server should be used for vCenter and all ESXi hosts to keep time in sync

**Services**
- View the status of all vCenter services
- Shows whether each service is set to start automatically and its current state
- Services can be restarted from here if needed

**Update**
- Used to check for and apply updates to the current version of the VCSA
- This is not for upgrading to a new major version — only for applying patches to the current version

**Administration**
- Configure password complexity requirements
- Reset passwords and modify related settings

**Syslog**
- Configure a remote syslog host to send VCSA system logs externally
- These logs are specific to the VCSA only — not VM or ESXi host logs

**Backup**
- Configure backups of the vCenter Server Appliance
- Covered in more detail in the Availability section of the course

---

> ### Lesson 9: Demo — Add ESXi Hosts to vCenter Inventory

Adding an ESXi host to vCenter inventory means bringing an already configured host under vCenter management. This is not creating a new host — it is connecting an existing host to vCenter so it can be managed through the vSphere Client and gain access to features like vMotion, HA, and DRS.

---

#### Step 1: Create a Virtual Datacenter

In the vSphere Client, go to Hosts and Clusters, right-click the vCenter Server Appliance and select New Datacenter. Give it a name (e.g. Training) and click OK.

---

#### Step 2: Create a Host and Cluster Folder (Optional but Recommended)

Right-click the new Datacenter and create a New Host and Cluster Folder. Folders are useful because:
- Permissions configured on a folder apply to all hosts inside it
- Alarms defined on a folder apply to all hosts inside it
- The Monitor tab shows all triggered alarms, tasks, and events for all hosts in the folder in one place

---

#### Step 3: Add the Host

Right-click the folder and select Add Host. Enter the IP address of the ESXi host (192.168.1.10) and provide the root credentials. Accept the certificate warning (expected when using a self-signed certificate).

Review the host summary and click Next.

---

#### Step 4: Assign a License

Choose a license key to apply to the host. An evaluation license can be used temporarily. Proper license keys can be applied later.

---

#### Step 5: Configure Lockdown Mode

Lockdown mode controls how the ESXi host can be accessed. There are three options:

| Mode | Description |
|------|-------------|
| Disabled | Host can be accessed via vCenter, Host Client, SSH, and DCUI freely |
| Normal Lockdown | Host management is forced through vCenter — Host Client and SSH are blocked, but DCUI remains accessible |
| Strict Lockdown | All access except vCenter is blocked — DCUI is also disabled |

For a lab environment, leave Lockdown mode disabled. In production, Normal or Strict Lockdown is recommended to enforce management through vCenter.

---

#### Step 6: Place Existing VMs

If the host already has VMs running on it, choose where to place them in the vCenter inventory. In this lab, the VCSA itself is running as a VM on ESXi(A), so it will appear here. Select the Training datacenter as the location and click Finish.

---

#### Result

The host will briefly show as disconnected, then connect. It is now visible in the vSphere Client under the folder and can be fully managed through vCenter. Expanding the host will show any VMs running on it — in this case, the vCenter Server Appliance VM.

---

> ### Lesson 10: Licensing, Pricing, and Packaging for vSphere 8

#### vSphere 8 Editions

vSphere 8 is available in two main editions:

| Edition | Description |
|---------|-------------|
| vSphere Standard | Base edition with core features |
| vSphere Enterprise Plus | Higher cost, includes all Standard features plus additional enterprise features |

Features exclusive to Enterprise Plus:
- Fault Tolerance (with higher vCPU limits)
- VM Encryption
- vSphere Distributed Switch (vDS)
- Auto Deploy
- DRS (Distributed Resource Scheduler)

For a full feature comparison, refer to the official VMware document: vSphere & vSphere+ Compute Virtualization: Licensing, Pricing and Packaging.

---

#### Licensing Model

vSphere 8 is licensed on a per-processor basis, identical to the vSphere 7 model.

| Rule | Detail |
|------|--------|
| Unit of licensing | Per physical processor (CPU socket) |
| Cores covered per license | Up to 32 cores per processor |
| More than 32 cores | Additional processor licenses required |

Example: A host with 2 CPU sockets, each with 32 cores → requires 2 processor licenses. If each socket has more than 32 cores, additional licenses are needed per socket.

---

#### Essentials Editions

For smaller environments just getting started with vSphere:

| Edition | Use Case |
|---------|---------|
| vSphere Essentials | Entry-level, limited features, small environments |
| vSphere Essentials Plus | Slightly more features than Essentials, still limited scale |

These editions are more restricted in features compared to Standard and Enterprise Plus but are a cost-effective starting point for small deployments.

---

> ### Lesson 11: Demo — Licensing vCenter and ESXi

License keys need to be applied to both vCenter and each ESXi host. In this lab, Enterprise Plus licenses obtained through VMUG Advantage EvalExperience are used.

---

#### Step 1: Add License Keys

In the vSphere Client:
1. Click the hamburger menu (three bars) at the top left
2. Go to Administration → Licenses
3. Click Add and paste in the license keys — one per line
4. Name each license key for easy identification (e.g. vCenter EvalExperience, ESXi EvalExperience)
5. Click Next then Finish

Two license keys are added:
- One for vCenter Server 8
- One for vSphere 8 (ESXi hosts) — Enterprise Plus edition, covers up to 16 CPUs and 32 processor cores each

---

#### Step 2: Assign License to vCenter

1. In the Licenses section, go to Assets
2. Select the vCenter Server Appliance
3. Click Assign License and choose the vCenter license
4. Click OK

---

#### Step 3: Assign License to ESXi Hosts

1. In the Assets section, click on Hosts
2. Select the ESXi host (ESXi(A))
3. Click Assign License and choose the vSphere Enterprise Plus license
4. Click OK

Repeat for each additional ESXi host added to the environment.

---

> ### Lesson 12: Demo — Basic ESXi Configuration

#### vSphere Client vs Host Client

| Interface | Scope | Use For |
|-----------|-------|---------|
| vSphere Client | All hosts in vCenter inventory | vMotion, DRS, HA, and all vCenter-dependent features |
| Host Client | Single ESXi host only | Local host configuration, emergency access |

The Host Client is accessed at `https://<esxi-ip>` using the host's own local credentials — not vCenter credentials. It should be used sparingly. All major management tasks should go through the vSphere Client.

---

#### Manage Tab — System Settings

**Autostart**
Controls whether VMs on this host start automatically when the host boots. A start delay can also be configured. Recommended: enable autostart for the VCSA so it comes back up after an ESXi reboot.

**Swap**
Configures system-level swap for the ESXi host itself — not for VMs. This is a memory reclamation process for non-VM workloads running on the host. Has no impact on VM performance.

**Time & Date**
Configure NTP time synchronization. The NTP server should match the one used by vCenter to keep all components in sync. In this lab, NTP server is 192.168.1.200.

Steps to enable NTP:
1. Go to Manage → System → Time & Date → Edit
2. Set NTP servers to 192.168.1.200
3. Set startup policy to Start and stop with host
4. Go to Manage → System → Services → start the ntpd service
5. Verify NTP status shows Running in Time & Date

Note: Precision Time Protocol (PTP) is also available but is heavier than NTP and better suited for specialized use cases.

---

#### Manage Tab — Hardware

View all hardware devices on the host. Options include rebooting the ESXi host and managing power policies.

**Power Management Policies:**

| Policy | Description |
|--------|-------------|
| High Performance | Maximum performance, highest power consumption |
| Balanced (default) | Balance between performance and power conservation |
| Low Power | Reduced power consumption, potential performance impact |
| Custom | User-defined policy |

---

#### Manage Tab — Licensing

License keys can be viewed and managed here, but when vCenter is managing the host, licenses should be managed centrally through the vSphere Client instead.

---

#### Manage Tab — Services

Start, stop, and manage all services running on the ESXi host. Key services:

| Service | Purpose |
|---------|---------|
| ntpd | NTP time synchronization — must be started after configuring NTP |
| vpxa | vCenter agent — restart if the host is not communicating properly with vCenter |
| SSH | Remote command-line access — should be stopped when not in use for security |
| ESXi Shell | Local shell access — should be stopped when not in use for security |

---

#### Security & Users Tab

**Authentication**
ESXi hosts can be joined to an Active Directory domain, allowing logins with AD credentials instead of local credentials. This enables tracking of who is accessing the host.

**Users**
Local users can be created and assigned different roles. By default, only the root user exists.

**Roles**
Pre-built roles are available and custom roles can be created:

| Role | Description |
|------|-------------|
| Administrator | Full access |
| Read-only | View only, no changes |
| No Access | No permissions |
| Custom | User-defined permissions via Add Role |

These roles are local to the host only — they are not vCenter roles.

**Lockdown Mode**
Lockdown mode forces all management through vCenter, blocking direct access via Host Client and SSH.

| Mode | Effect |
|------|--------|
| Disabled (default) | Full direct access allowed |
| Normal | Management forced through vCenter — DCUI still accessible |
| Strict | Management forced through vCenter — DCUI also disabled |

Exception users can be defined to retain access even when Lockdown mode is enabled. This is important as a failsafe if vCenter goes down — without exception users, the host becomes unmanageable without vCenter.

---

#### Networking — Firewall Rules

Firewall rules in the Host Client apply only to the ESXi host itself, not to VMs running on it. They control which IP address ranges can access host services such as SSH, DNS, and DHCP.

Example: restricting SSH access to only the local network (e.g. 192.168.1.0/24) reduces the attack surface significantly even if SSH is left enabled.

---

> ### Lesson 13: Demo — Content Libraries

#### What is a Content Library?

A Content Library is a centralized repository in vCenter that stores:
- ISO images
- OVF templates
- VM templates

Content Libraries can be shared across multiple vCenter instances using a publish/subscribe model.

Access: vSphere Client → hamburger menu → Shortcuts → Content Libraries

---

#### Creating a Content Library

1. Go to Content Libraries → New Content Library
2. Name the library (e.g. RickDemo) and associate it with the vCenter Server
3. Choose the library type:

| Type | Description |
|------|-------------|
| Local | Content available only to this vCenter Server |
| Published | Content can be shared — other vCenter Servers can subscribe to it |
| Subscribed | Pulls content from a published library on another vCenter Server |

4. Optionally enable authentication (password required to subscribe)
5. Select a datastore to store the library contents
6. Click Finish

---

#### Creating a VM Template from a Golden Image

A golden image is a VM configured exactly as desired — fully updated and set up — used as the base for creating future VMs.

To clone a VM as a template to the Content Library:
1. Right-click the VM (e.g. GoldenImage) → Clone → Clone as Template to Library
2. Choose VM Template or OVF format
3. Name the template (e.g. MyTemplate)
4. Select the Content Library (e.g. RickDemo), compute resource, and datastore
5. Click Finish

The template will now appear in the Content Library and can be used to create new VMs.

---

#### Updating a Template (Check Out / Check In)

Templates can be updated without recreating them from scratch:

1. In the Content Library, right-click the template → Check Out VM From This Template
2. Name the temporary VM (e.g. Temporary), select a host and datacenter
3. The template is locked while checked out
4. Boot the temporary VM, make changes (updates, software installs, etc.)
5. Shut down or power off the temporary VM
6. Go back to the Content Library → right-click the template → Check In
7. Changes from the temporary VM are applied back to the template

---

#### Subscribed Content Libraries

A subscribed Content Library pulls content from a published Content Library on another vCenter Server.

To subscribe:
1. Go to Content Libraries → New Content Library
2. Name it (e.g. DemoSubscribed) and select Subscribed Content Library
3. Paste the subscription URL from the published library's Summary tab
4. Enter the password if authentication was configured
5. Choose download timing:

| Option | Description |
|--------|-------------|
| Download immediately | All content pulled and stored locally right away |
| Download when needed | Content fetched on demand — saves storage space |

6. Select a datastore to store the downloaded content
7. Click Finish

---

> ### Lesson 14: VMware Flings

#### What are VMware Flings?

VMware Flings are unofficial software tools and utilities released by VMware developers. They are available at flings.vmware.com and represent early-access features or community tools that have not yet made it into the official vSphere product.

Many features that are now standard in vSphere were first released as Flings. For example, the current HTML5-based vSphere Client was originally released as a Fling before becoming the official client.

---

#### What Flings are Available?

Flings are organized by category (e.g. vSphere, vCenter). Examples include:

- vSphere Console for Kubernetes
- Virtual Machine Desired State Configuration — allows specifying CPU and memory desired state that takes effect on Guest OS reboot, without requiring scheduled downtime to reconfigure the VM

New Flings are released regularly by VMware developers.

---
## Section 4: Configure and Manage vSphere 8 Networking

This section covers virtual networking in vSphere 8. Topics include port groups, trunk ports, VMkernel ports, jumbo frames, vSphere Standard Switches, vSphere Distributed Switches, failover configurations, and NIC teaming methods.

---

> ### Lesson 1: Introduction to Virtual Networking

#### Virtual Machine Networking Basics

A VM communicates over the network using a virtual NIC (vNIC). The guest operating system sees the vNIC as a regular physical NIC — it has no awareness that it is virtual.

VM traffic flows as follows:
- VM → vNIC → virtual switch Port Group → (if external) → vmnic → physical switch → external network

---

#### Key Networking Components

| Component | Description |
|-----------|-------------|
| vNIC | Virtual Network Interface Card assigned to a VM |
| Virtual Switch (vSwitch) | A software switch inside the ESXi host that connects VMs to each other and to the physical network |
| Port Group | A named group of ports on a virtual switch — defines VLAN membership and security policies |
| vmnic | The physical network adapter on the ESXi host — used when traffic needs to leave the host |
| VMkernel Port | A special port on a virtual switch used for host traffic (management, vMotion, storage) — not VM traffic |

---

#### VM-to-VM Communication

VMs connected to the same Port Group on the same host can communicate directly with each other without traffic ever touching the physical network.

VMs on different Port Groups (different VLANs) must have their traffic routed — it leaves the host via a trunk port, hits a physical router, and returns.

---

#### VLANs and Trunk Ports

Each Port Group on a virtual switch is assigned a VLAN. A trunk port on the physical uplink carries traffic for multiple VLANs on a single physical connection, with each frame tagged to identify its VLAN.

Traffic flow between VMs on different VLANs:
1. VM1 sends traffic → hits Port Group tagged VLAN 10
2. Traffic exits the host via the trunk port, tagged for VLAN 10
3. Physical switch forwards to the router (default gateway)
4. Router routes the traffic to VLAN 20
5. Traffic returns via trunk port tagged VLAN 20 → delivered to VM2

---

#### VMkernel Ports

VMkernel ports handle all non-VM traffic on the ESXi host:

| Traffic Type | Example |
|-------------|---------|
| Management | ESXi host communicating with vCenter |
| vMotion | Live VM migration between hosts |
| IP Storage | iSCSI, NFS |
| vSAN | VMware storage cluster traffic |

---

#### Jumbo Frames and MTU

Jumbo Frames are large Ethernet frames (typically MTU 9000) that improve network efficiency by reducing the number of frames needed to transfer large amounts of data — similar to shipping four guitars in one box instead of four separate boxes.

Both vSphere Standard Switch and vSphere Distributed Switch support Jumbo Frames.

**Critical requirement:** MTU must be configured consistently across all virtual switches and physical switches. A mismatch causes fragmentation and reassembly at the physical switch, which has a significant negative impact on performance.

| Scenario | Result |
|----------|--------|
| vSwitch MTU 9000, physical switch MTU 9000 | Efficient — no fragmentation |
| vSwitch MTU 9000, physical switch MTU 1524 | Physical switch must fragment every large frame — major performance impact |

---

> ### Lesson 2: vSphere Standard Switch Failure Detection

#### VMNICs and Virtual Switches

Each VMNIC (physical adapter on the ESXi host) can only be assigned to one virtual switch. Virtual switches cannot share VMNICs.

A virtual switch with multiple VMNICs automatically distributes VM traffic across all of them. The specific distribution method can be configured (covered in a later lesson).

---

#### Failure Type 1: Link State Failure

A link state failure occurs when the physical cable between a VMNIC and the physical switch is cut or disconnected.

- The green link light on the VMNIC turns red
- This is the simplest failure to detect — the ESXi host sees it immediately
- The host reassigns all VMs that were using the failed VMNIC to a healthy VMNIC

---

#### Failure Type 2: Upstream Device Failure

An upstream failure occurs when the physical connection between switches is broken — the VMNIC still has a green link light, but traffic flows into a dead end.

Example: Two physical switches are interconnected. One VMNIC connects to the isolated switch. When the inter-switch link is cut, the VMNIC link light stays green but all traffic from it goes nowhere.

This type of failure cannot be detected by link state alone — it requires **Beacon Probing**.

---

#### Beacon Probing

With Beacon Probing enabled, all VMNICs periodically send out beacon probe signals to each other. If a VMNIC stops receiving beacon probes from the other adapters, the host concludes that upstream connectivity has been lost on that adapter.

| Condition | Result |
|-----------|--------|
| Beacon probes received normally | VMNIC considered healthy |
| Beacon probes not received | VMNIC marked as down — VMs reassigned to healthy adapters |

Beacon Probing detects failures that occur beyond the directly connected switch — failures that link state monitoring alone cannot see.

---

> ### Lesson 3: NIC Teaming Methods for vSphere Standard Switches

#### What is NIC Teaming?

NIC Teaming allows a virtual switch to spread VM traffic across multiple VMNICs (physical uplinks). The goal is load balancing and redundancy. The method chosen determines how the virtual switch decides which VMNIC to use for outbound traffic.

---

#### Method 1: Originating Virtual Port ID

Each VM connects to a specific virtual port on the virtual switch. Based on that port ID, the VM is permanently assigned to one VMNIC for all physical network traffic.

- VM1 → Port 1 → VMNIC 1
- VM2 → Port 2 → VMNIC 2
- VM3 → Port 3 → VMNIC 3

Each VM is tethered to one specific physical adapter. Traffic from a given VM always exits through the same VMNIC.

Physical switch configuration: **No Port Channel, no EtherChannel, no LACP.** The physical switch must treat each uplink as an independent connection. If Port Channel were enabled, the switch might send return traffic out the wrong port for a given MAC address.

---

#### Method 2: Source MAC Hash

Very similar to Originating Port ID, but the VMNIC is selected based on the source MAC address of the VM rather than the virtual port ID. Since every VM has a unique MAC address, each VM is still tethered to one specific VMNIC.

Physical switch configuration: **No Port Channel, no EtherChannel, no LACP** — same reason as above.

---

#### Method 3: IP Hash

The VMNIC is selected based on a hash of both the source IP and destination IP of each packet. Because the destination IP changes per connection, a single VM can use multiple VMNICs simultaneously — one connection may go out VMNIC 1 while another goes out VMNIC 2.

This is the only method where a single VM can actively use more than one physical adapter at the same time.

Physical switch configuration: **Port Channel / EtherChannel / LACP must be configured** on the physical switch to bond the uplinks together.

---

#### NIC Teaming Methods Comparison

| Method | Basis for VMNIC Selection | VM Uses Multiple VMNICs? | Physical Switch Config |
|--------|--------------------------|--------------------------|----------------------|
| Originating Port ID | Virtual port the VM connects to | No — one VMNIC per VM | No Port Channel / LACP |
| Source MAC Hash | Source MAC address of the VM | No — one VMNIC per VM | No Port Channel / LACP |
| IP Hash | Hash of source IP + destination IP | Yes — per connection | Port Channel / LACP required |

---

> ### Lesson 4: Traffic Shaping on vSphere Standard Switches

#### What is Traffic Shaping?

Traffic Shaping is a policy applied to a port group that limits how much bandwidth VMs in that port group can consume. This prevents one port group from monopolizing the shared VMNICs and starving other port groups of bandwidth.

Note: Traffic Shaping works slightly differently on the vSphere Distributed Switch — covered in a later lesson.

---

#### Traffic Shaping Parameters

| Parameter | Description |
|-----------|-------------|
| Peak Bandwidth | The maximum bandwidth any VM in the port group can use at any moment |
| Average Bandwidth | The maximum bandwidth a VM must average out to over time |
| Burst Size | The maximum amount of data a VM can transmit at Peak Bandwidth before being throttled back to Average Bandwidth |

---

#### How Burst Size Works

Burst size works like a savings allowance:
- A VM is granted its Average Bandwidth continuously
- If the VM uses less than its Average Bandwidth, the unused bandwidth accumulates as burst credit up to the configured Burst Size maximum
- When the VM needs extra bandwidth, it can transmit at Peak Bandwidth until it exhausts its saved burst credit
- Once burst credit is exhausted, the VM is capped back at Average Bandwidth until it saves up more

Example with Peak 100 Mbps, Average 50 Mbps, Burst Size 100 MB:
- VM can transmit at 100 Mbps until it sends 100 MB of data at that rate
- After exhausting the burst, VM is limited to 50 Mbps
- As long as the VM stays below 50 Mbps, it rebuilds its burst credit back toward 100 MB

---

> ### Lesson 5: Security Settings on vSphere Standard Switches

#### Where Security Settings Are Configured

Security settings can be applied at two levels:

| Level | Scope |
|-------|-------|
| Virtual Switch | Global — propagates down to all port groups by default |
| Port Group | Overrides the switch-level settings for that specific port group |

This means port groups can have different security policies from each other and from the switch defaults.

---

#### Security Setting 1: Forged Transmits

Controls whether a VM is allowed to send traffic with a MAC address different from the one assigned by vSphere to its virtual NIC.

- **Accept** — MAC spoofing for outbound traffic is allowed
- **Reject** — Only the vSphere-assigned MAC address can be used for outbound traffic

Use case: A VM migrated from a physical machine may need to retain its original MAC address for licensing purposes. Forged Transmits makes this possible by allowing the OS to override the MAC address on outbound frames.

Default: **Accept** (allowed) — recommended to set to Reject if MAC spoofing is not needed.

---

#### Security Setting 2: MAC Address Changes

Controls whether a VM is allowed to receive traffic addressed to a MAC address different from the one assigned by vSphere to its virtual NIC.

- **Accept** — Inbound traffic to a manually configured MAC address is allowed
- **Reject** — Only the vSphere-assigned MAC address can receive traffic

Note: Forged Transmits handles outbound MAC spoofing. MAC Address Changes handles inbound MAC spoofing. Both must be set to Accept for full MAC spoofing to work.

Default: **Accept** (allowed) — recommended to set to Reject if not needed.

---

#### Security Setting 3: Promiscuous Mode

Controls whether a VM can see all traffic on the virtual switch, not just traffic addressed to it.

- **Accept** — The VM's virtual NIC receives all frames on the switch (enables network sniffing)
- **Reject** — The VM only receives traffic addressed to it

Use case: Installing a network sniffer or monitoring tool on a VM that needs to capture all traffic on the switch.

Default: **Reject** — should remain disabled unless specifically needed. Leaving promiscuous mode enabled is a significant security risk.

---

#### Security Settings Summary

| Setting | Default | Controls | Recommended |
|---------|---------|----------|-------------|
| Forged Transmits | Accept | Outbound MAC spoofing | Reject unless needed |
| MAC Address Changes | Accept | Inbound MAC spoofing | Reject unless needed |
| Promiscuous Mode | Reject | Sniffing all switch traffic | Keep Reject unless needed |

---

> ### Lesson 6: TCP/IP Stacks

#### What is a TCP/IP Stack?

A TCP/IP Stack defines the networking configuration used by VMkernel ports for host-generated traffic. Each stack can have its own Default Gateway and DNS Servers, allowing different types of traffic to be directed to different networks.

This applies only to VMkernel port traffic — not to VM traffic.

---

#### Available TCP/IP Stacks

| Stack | Used For |
|-------|---------|
| Default | All management traffic, vMotion, and general system traffic — used unless a specific stack is assigned |
| vMotion | Dedicated stack for vMotion traffic — useful for long-distance vMotions requiring a separate network or gateway |
| Provisioning | Dedicated stack for cold migrations, cloning, and snapshots |
| Custom | User-defined stacks — can be created and assigned to VMkernel ports as needed |

---

#### Why Use Multiple TCP/IP Stacks?

By default, all VMkernel traffic uses the same Default Gateway. If different types of traffic need to reach different networks — for example, vMotion traffic routed over a dedicated WAN link — assigning that traffic to its own TCP/IP Stack allows it to use a different Default Gateway and DNS Server independently of other traffic types.

---

> ### Lesson 7: Demo — Creating a vSphere Standard Switch

A vSphere Standard Switch (vSS) is local to one specific ESXi host. It is created and managed per-host, unlike the vSphere Distributed Switch which is managed centrally from vCenter.

---

#### Default Configuration on a New ESXi Host

When an ESXi host is set up, the following are created automatically:

| Object | Name | Purpose |
|--------|------|---------|
| Virtual Switch | vSwitch0 | Default switch |
| VMkernel Port | vmk0 | Management traffic — used by the host to communicate with vCenter |
| Port Group | VM Network | Default port group for VM traffic |

Access: vSphere Client → Hosts and Clusters → ESXi host → Configure → Virtual Switches

---

#### Creating a New Standard Switch

1. Go to Configure → Virtual Switches → Add Networking
2. Select Virtual Machine Port Group for a Standard Switch
3. Select New Standard Switch
4. Configure MTU (default 1500 — must match the physical network)
5. Add VMNICs (physical adapters) — or leave empty to create an Isolated switch
6. Name the port group and assign a VLAN ID
7. Review and click Finish

---

#### Isolated Virtual Switch

A virtual switch with no VMNICs assigned has no connection to the physical network. VMs connected to it can communicate with each other but cannot reach:
- Physical network devices
- VMs on other ESXi hosts
- VMs on other port groups

This is useful for isolated test environments.

---

#### VLAN Configuration on a Port Group

| VLAN Setting | Behavior |
|-------------|---------|
| None (0) | No VLAN tag — untagged traffic |
| Specific VLAN ID (e.g. 20) | Tags outbound frames with that VLAN — requires trunk port on physical switch |
| All (4095) | Virtual Guest Tagging — VLAN is assigned inside the guest OS and passed through as-is |

---

#### Adding a Physical Adapter (VMNIC) to a Switch

Each VMNIC can only be assigned to one virtual switch. To add a VMNIC to an existing switch:
1. Go to Add Networking → Physical Network Adapter
2. Select the target virtual switch
3. Move the unclaimed VMNIC to Active adapters
4. Click Finish

Note: In a nested lab (ESXi running inside VMware Workstation), adding a new VMNIC requires adding a network adapter in Workstation settings and rebooting the ESXi host. Shut down the VCSA first before rebooting the host.

---

#### Migrating a VM Between Port Groups

To move a VM from one port group to another:
1. Right-click the VM → Edit Settings
2. Select the Network Adapter
3. Click Browse and choose the new port group
4. Click OK

VMs can be moved between port groups on different switches as long as both switches share the same physical network and VLAN configuration.

---

#### Lab Reference

| Object | Detail |
|--------|--------|
| Default switch | vSwitch0 — VMNIC0, vmk0 management port, VM Network port group |
| New switch | vSwitch1 — VMNIC1, RickDemo port group, VLAN None |

---

> ### Lesson 8: Demo — Configuring a vSphere Standard Switch

#### Accessing Switch Settings

In the vSphere Client, go to the ESXi host → Configure → Virtual Switches → select the switch (e.g. vSwitch1) → Edit.

---

#### MTU Settings

The MTU (Maximum Transmission Unit) controls the maximum Ethernet frame size. Default is 1500. Set to 9000 to enable Jumbo Frames. Must match the physical network configuration.

---

#### Port Settings

| Setting | Description |
|---------|-------------|
| Elastic (default) | Switch starts with 8 ports and automatically expands as more VMs connect, and contracts when not needed |

No manual port management is needed with Elastic mode unless approaching configuration maximums.

---

#### Security Settings

Configured under the Security tab. These settings were covered in depth in Lesson 5. Quick reference:

| Setting | Default | Recommendation |
|---------|---------|---------------|
| Promiscuous Mode | Reject | Keep Reject — only enable for sniffers or IPS/IDS tools |
| MAC Address Changes | Accept | Set to Reject unless MAC spoofing is needed (inbound) |
| Forged Transmits | Accept | Set to Reject unless MAC spoofing is needed (outbound) |

MAC spoofing use case: a physical server virtualized where the software is licensed by MAC address. Both MAC Address Changes and Forged Transmits must be set to Accept to allow full MAC spoofing.

---

#### Traffic Shaping Settings

Configured per virtual switch to prevent any single VM from overwhelming the physical adapters. Covered in depth in Lesson 4. Quick reference:

| Parameter | Example Value | Description |
|-----------|--------------|-------------|
| Average Bandwidth | 50 Mbps | Sustained bandwidth limit per VM over time |
| Peak Bandwidth | 75 Mbps | Maximum burst speed per VM |
| Burst Size | Configurable | How much data can be sent at Peak before throttling back to Average |

---

#### NIC Teaming and Failover Settings

Options for load balancing and failover are also available on this tab. Covered in detail in earlier lessons:
- Load balancing methods: Originating Port ID, Source MAC Hash, IP Hash (Lesson 3)
- Failure detection: Link Status, Beacon Probing (Lesson 2)

---

> ### Lesson 9: Demo — Migrating a VMkernel Port Between Standard Switches

A VMkernel port can be migrated from one virtual switch to another. This process is performed from the **destination** switch — the switch you want to move the VMkernel port to.

Access: vSphere Client → ESXi host → Configure → Virtual Switches → select the destination switch → More Options → Migrate VMkernel Adapter

---

#### Migration Steps

1. Navigate to the destination virtual switch
2. Click More Options → Migrate VMkernel Adapter
3. Select the VMkernel port to migrate (e.g. the vMotion VMkernel port)
4. Confirm the network label
5. Click Finish

---

#### Critical Warnings

**Do not accidentally break management connectivity.** Migrating the management VMkernel port (vmk0) to a switch that is not connected to the correct network will cut off communication between the ESXi host and vCenter — leaving the host unmanageable through vCenter.

**Verify VLANs and network connectivity before migrating.** If the destination switch is not connected to the same physical network or the correct VLANs as the source switch, migrating the VMkernel port will break whatever service that port handles (e.g. vMotion, storage).

| VMkernel Port | Risk if Migrated Incorrectly |
|---------------|------------------------------|
| vmk0 (Management) | Host loses connection to vCenter |
| vMotion VMkernel | vMotion breaks — VMs cannot be live migrated |
| Storage VMkernel | IP storage access lost |

---

> ### Lesson 10: Demo — Managing Port Groups and VMkernel Ports

#### Port Group Settings

Port group settings can override the virtual switch defaults. This allows different port groups on the same switch to have different configurations.

Settings configurable per port group:
- VLAN ID
- Security policies (Promiscuous Mode, MAC Address Changes, Forged Transmits)
- Traffic Shaping (useful to limit bandwidth per VM for a specific group)

Access: vSphere Client → ESXi host → Configure → Virtual Switches → select port group → Edit

---

#### VMkernel Ports

VMkernel ports are viewed at the host level, not per-switch. They handle traffic generated by the ESXi host itself — not VM traffic.

Access: vSphere Client → ESXi host → Configure → VMkernel Adapters

Common VMkernel ports:

| Port | Service | Purpose |
|------|---------|---------|
| vmk0 | Management | Host communication with vCenter, Host Client, SSH |
| vmk1 (example) | Storage | Transmits storage reads/writes to the storage system |
| vmk2 (example) | vMotion | Transfers VM data during live migration between hosts |

VMkernel ports using the same TCP/IP Stack share the same Default Gateway and DNS servers. Ports on different TCP/IP Stacks can route to different gateways — for example, the vMotion VMkernel port can use a separate Default Gateway if assigned to the vMotion TCP/IP Stack.

---

#### Creating a New VMkernel Port

1. Go to Add Networking → VMkernel Network Adapter → Next
2. Select the target virtual switch (e.g. vSwitch0)
3. Configure the network label (name)
4. MTU is inherited from the virtual switch
5. Select the TCP/IP Stack
6. Enable the services this port will handle:

| Service Option | Use Case |
|---------------|---------|
| Management | Adds a second management path for HA |
| vMotion | Enables live VM migration traffic on this port |
| Fault Tolerance Logging | FT heartbeat and logging traffic |
| vSAN | VMware software-defined storage traffic |

7. Assign an IP address and subnet mask
8. Click Finish

---

#### Physical Adapters Tab

Shows all VMNICs on the host and which virtual switch each is assigned to. A VMNIC can only belong to one virtual switch.

---

#### TCP/IP Configuration Tab

Shows all configured TCP/IP Stacks on the host. Stacks can be edited to set a Default Gateway, rename the stack, or configure advanced settings.

---

#### Standard Switch Scope Reminder

A vSphere Standard Switch is always local to one ESXi host. It cannot span multiple hosts. The vSphere Distributed Switch — covered in upcoming lessons — spans multiple ESXi hosts and is managed centrally from vCenter.

---

> ### Lesson 11: Introduction to vSphere Distributed Switch

#### Why Use a Distributed Switch?

With vSphere Standard Switches, every ESXi host requires its own individually configured switch. Port groups, VMNICs, VMkernel ports, VLANs, and security settings all have to be configured manually on each host. This is time consuming and error prone — a wrong VLAN or security setting on one host is easy to miss.

The vSphere Distributed Switch (vDS) solves this by allowing a single switch configuration to be created once in vCenter and distributed to all participating ESXi hosts automatically.

Requires: **vSphere Enterprise Plus** licensing.

---

#### Management Plane vs Data Plane

| Plane | Role | What Happens if it Fails |
|-------|------|--------------------------|
| Management Plane (vCenter) | Controls the configuration of the vDS | VMs keep running — switch keeps working — only configuration changes are blocked |
| Data Plane (hidden switches on each ESXi host) | Carries actual VM traffic | If a host fails, VMs on that host are affected — but other hosts are not |

vCenter manages the vDS configuration but no VM traffic flows through vCenter. A vCenter outage does not cause network downtime.

---

#### How the Distributed Switch Works

1. The vDS is configured once in vCenter
2. The configuration is pushed down to all member ESXi hosts
3. A hidden virtual switch (host proxy switch) is created on each host to handle the actual data plane traffic
4. From the administrator's perspective it looks like one single switch spanning all hosts

---

#### Distributed Port Groups

A distributed port group is created once in vCenter and a copy is automatically pushed to all ESXi hosts in the vDS. This means:
- VLANs, security settings, and traffic shaping are configured once and applied consistently everywhere
- Eliminates per-host configuration errors
- Speeds up provisioning when adding new hosts

---

#### VMkernel Ports — Still Per Host

VMkernel ports cannot be centrally deployed via the vDS because each host's VMkernel ports have unique IP addresses. They must still be created individually on each host.

---

> ### Lesson 12: vSphere Distributed Switch — Review and Advanced Features

#### Review: vDS Core Concepts

| Concept | Detail |
|---------|--------|
| Management / Control Plane | vCenter — stores and manages the centralized vDS configuration |
| Data Plane | Hidden virtual switches on each participating ESXi host — carry actual VM traffic |
| Distributed Port Groups | Created once in vCenter, pushed to all member hosts with identical configuration |
| VMkernel Ports | Still configured per host — each has a unique IP address so cannot be centrally deployed |

---

#### Features Available on vDS but NOT on Standard Switch

**Private VLANs (PVLANs)**
Allows the creation of VLANs within a VLAN. Secondary VLANs are created inside a primary VLAN with three modes:

| Secondary VLAN Type | Description |
|--------------------|-------------|
| Isolated | VMs can only communicate with the promiscuous port — not with each other |
| Community | VMs can communicate with each other and with the promiscuous port |
| Promiscuous | Can communicate with all secondary VLANs — typically used for the default gateway |

**Route Based on Physical NIC Load**
An advanced load balancing method that actively monitors the utilization of each VMNIC. If a VMNIC becomes overloaded, VMs are automatically moved to a less loaded VMNIC. This is a dynamic method unlike the Standard Switch methods which are static.

**LACP (Link Aggregation Control Protocol)**
Allows multiple physical adapters to be bonded together into a single logical link. LACP is an open standard and supports a variety of load balancing algorithms to distribute traffic across the bonded adapters.

---

> ### Lesson 13: Demo — Creating a vSphere Distributed Switch

#### Creating the vDS in vCenter

Access: vSphere Client → Networking icon → right-click Datacenter → Distributed Switch → New Distributed Switch

Configuration steps:

| Setting | Value / Notes |
|---------|--------------|
| Name | e.g. RickCrisciDemo |
| ESXi Version | Choose the latest version all hosts can support — newer = more features |
| Number of Uplinks | How many VMNICs per host can be dedicated to this vDS — default 4 |
| Network I/O Control | Leave enabled — used to prioritize different traffic types over physical adapters |
| Default Port Group | Optionally create one at the same time (e.g. RickCrisciDemoPG) |

After clicking Finish, the vDS exists in vCenter only. It has not yet been distributed to any ESXi hosts.

---

#### Adding Hosts to the vDS

Right-click the vDS → Add and Manage Hosts → Add Hosts

Steps:
1. Select which ESXi hosts to add
2. Assign VMNICs to the vDS uplinks — each host contributes its own physical adapters
   - In this lab: vmnic1 is available and unused on all three hosts → assigned as Uplink 1
3. Optionally migrate VMkernel ports to the vDS
4. Optionally migrate existing VMs to a vDS port group

**VMkernel port migration recommendation:** Do not migrate VMkernel ports immediately. Validate the vDS is working correctly first. Many organizations permanently leave VMkernel ports on a Standard Switch since they must be configured per host anyway.

---

#### Removing Hosts from the vDS

Right-click the vDS → Add and Manage Hosts → Remove Hosts

**Important:** A host cannot be removed from the vDS if any VM running on that host is connected to the vDS. The VM must be powered off or migrated to a different port group first.

---

#### Lab Reference

| Object | Detail |
|--------|--------|
| vDS name | RickCrisciDemo |
| Default port group | RickCrisciDemoPG |
| ESXi version | 8.0.0 |
| Uplinks per host | 4 configured, vmnic1 assigned as Uplink 1 on all three hosts |
| Hosts in vDS | ESXi(A), ESXi host 2, ESXi host 3 |

---

> ### Lesson 14: CDP and LLDP — Physical Switch Discovery

#### What Are CDP and LLDP?

Both protocols allow virtual switches to discover information about the physical switches they are connected to. This is useful for troubleshooting, documentation, and identifying which physical ports VMNICs are plugged into before making changes in the datacenter.

---

#### CDP — Cisco Discovery Protocol

| Property | Detail |
|----------|--------|
| Type | Cisco proprietary |
| Works with | Cisco hardware only |
| Supported on | vSphere Standard Switch and vSphere Distributed Switch |

Information discoverable via CDP:
- Which physical port on the Cisco switch each VMNIC is connected to
- Physical switch details

---

#### LLDP — Link Layer Discovery Protocol

| Property | Detail |
|----------|--------|
| Type | Open standard — not vendor specific |
| Works with | Any hardware that supports LLDP (e.g. Dell, HP, etc.) |
| Supported on | vSphere Distributed Switch only — NOT supported on vSphere Standard Switch |

Information discoverable via LLDP:
- IP address of the physical switch
- Which port each VMNIC is plugged into
- Hardware platform of the physical switch
- Software/firmware version of the physical switch

---

#### CDP vs LLDP Comparison

| Feature | CDP | LLDP |
|---------|-----|------|
| Standard | Cisco proprietary | Open standard |
| Hardware compatibility | Cisco only | Any vendor |
| Supported on vSS | Yes | No |
| Supported on vDS | Yes | Yes |

---

> ### Lesson 15: Demo — Basic vSphere Distributed Switch Configuration

#### Accessing vDS Settings

vSphere Client → Networking → right-click the vDS → Settings → Edit Settings

---

#### Configurable Settings

**MTU**
Same concept as on the Standard Switch. Default is 1500. Set to 9000 for Jumbo Frames. Must match the physical network configuration.

**Network I/O Control**
Enable or disable traffic prioritization across the physical uplinks. Covered in more detail in advanced lessons.

**Discovery Protocol**
Choose between CDP or LLDP for physical switch discovery:

| Protocol | Standard | Supported On |
|----------|----------|-------------|
| CDP | Cisco proprietary | vSS and vDS |
| LLDP | Open standard | vDS only |

Three operating modes are available for the selected protocol:
- **Listen** — the vDS learns information from the physical switch only
- **Advertise** — the vDS sends information about itself to the physical switch only
- **Both** — the vDS both learns from and advertises to the physical switch

**Uplinks**
View and modify the number of uplinks configured on the vDS. This was set during creation (4 uplinks in this lab). Uplinks not backed by a physical VMNIC are unused but do not cause issues.

Note: Modifying uplinks here changes the logical uplink count on the vDS — it does not add or remove physical VMNICs from the hosts.

---

#### Lab Reference

| Setting | Value |
|---------|-------|
| vDS name | RickCrisciDemo |
| MTU | 1500 |
| Uplinks configured | 4 (only Uplink 1 in use — vmnic1 on each host) |

---

> ### Lesson 16: Private VLANs (PVLANs)

#### What Are Private VLANs?

Private VLANs are a vSphere Distributed Switch exclusive feature — not available on the Standard Switch. They allow traffic isolation within a single VLAN by creating secondary VLANs inside a primary VLAN, each with different communication rules.

Use case: Multiple VMs share the same IP subnet and primary VLAN but belong to different departments or tenants that should not communicate freely with each other.

---

#### Structure

A Private VLAN has one **primary VLAN** containing multiple **secondary VLANs**:

| Secondary VLAN Type | Can communicate with | Cannot communicate with |
|--------------------|---------------------|------------------------|
| Isolated | Promiscuous only | Other isolated VMs, community VLANs |
| Community | Other VMs in the same community + promiscuous | Other communities, isolated VMs |
| Promiscuous | All secondary VLANs (isolated, community, other promiscuous) | Nothing — it can reach everything |

---

#### Example

Primary VLAN: 10 (all VMs in 10.1.1.0/24 subnet)

| Secondary VLAN | Type | Example Use |
|---------------|------|-------------|
| 110 | Isolated | Library computers — can only reach the proxy server |
| 111 | Community | Department A — members talk to each other and the gateway |
| 112 | Community | Department B — isolated from Department A |
| 10 | Promiscuous | Default gateway / proxy server — reachable by all |

- VM in VLAN 110 (isolated) → cannot talk to any other isolated VM or any community VM → can only reach VLAN 10 (promiscuous)
- VM in VLAN 111 (community) → can talk to other VLAN 111 VMs and VLAN 10 → cannot talk to VLAN 112 or VLAN 110
- VM in VLAN 10 (promiscuous) → can talk to everyone

---

> ### Lesson 17: Demo — Configuring Private VLANs on a vSphere Distributed Switch

Private VLANs must be configured at the vDS level first before they can be applied to any port group.

---

#### Step 1: Configure Private VLANs on the vDS

vSphere Client → Networking → right-click the vDS → Settings → Edit Private VLAN

Create the Primary VLAN first, then add Secondary VLANs within it:

| VLAN | Type | Behavior |
|------|------|----------|
| Primary VLAN 10 | Primary | Container for all secondary VLANs |
| Secondary VLAN 10 | Promiscuous | Can communicate with all secondary VLANs |
| Secondary VLAN 11 | Community | Members communicate with each other and promiscuous only |
| Secondary VLAN 12 | Community | Separate community — cannot communicate with VLAN 11 |
| Secondary VLAN 13 | Isolated | Cannot communicate with anyone — only the promiscuous VLAN |

Click OK to save.

---

#### Step 2: Apply a Private VLAN to a Port Group

Right-click the port group → Edit Settings → VLAN → select Private VLAN

Choose which secondary VLAN this port group belongs to. All VMs connected to this port group will be subject to that secondary VLAN's rules.

---

#### Real-World Analogy: Hotel Network

| Location | Secondary VLAN Type | Reason |
|----------|-------------------|--------|
| Guest rooms | Isolated | Guests should not communicate with each other — only reach the router |
| Conference room | Community | Attendees can communicate with each other and reach the router |
| Router / gateway | Promiscuous | Must be reachable by all guests and conference rooms |

---

> ### Lesson 18: NIC Teaming Methods on vSphere Distributed Switch

All Standard Switch NIC Teaming methods are also available on the vDS (Originating Port ID, Source MAC Hash, IP Hash). The vDS additionally offers two more advanced methods not available on the Standard Switch.

---

#### Method 4: Route Based on Physical NIC Load (vDS only)

Unlike the Standard Switch methods, this is an intelligent and dynamic load balancing method. It actively monitors VMNIC utilization and moves VMs between physical adapters when congestion is detected.

How it works:
- Each VM is assigned to one VMNIC at any given time
- If a VMNIC exceeds a utilization threshold (e.g. 75%), VMs are automatically moved to a less loaded VMNIC
- No VM uses multiple VMNICs simultaneously — they are just reassigned over time

Physical switch configuration: **No Port Channel, EtherChannel, or LACP** — same requirement as Port ID and MAC Hash methods, since each VM is still bound to one adapter at a time.

| Feature | Standard Switch methods | Route Based on Physical NIC Load |
|---------|------------------------|----------------------------------|
| Monitors utilization | No | Yes |
| Adjusts dynamically | No | Yes |
| VM uses multiple VMNICs at once | No | No |
| Available on vDS | Yes | Yes (vDS only) |

---

#### Method 5: LACP — Link Aggregation Control Protocol (vDS only)

LACP bonds multiple physical VMNICs together into a single logical pipe called a **Link Aggregation Group (LAG)**. Traffic is then load balanced across all physical adapters in the LAG simultaneously.

How it works:
1. Configure a LAG on the vDS — identify which VMNICs will participate
2. Configure a matching LAG on the physical switch — both sides must match
3. LACP load balances traffic across the bonded adapters using a selected algorithm

LACP load balancing algorithms (examples):

| Algorithm | Basis |
|-----------|-------|
| Destination IP | Traffic balanced by destination IP address |
| Destination MAC | Traffic balanced by destination MAC address |
| Source + Destination IP | Traffic balanced by both source and destination IP |
| Source IP + VLAN | Traffic balanced by source IP and VLAN tag |
| Source + Destination IP + VLAN | Most granular — combines IP addresses and VLAN |

Physical switch configuration: **Port Channel / LAG must be configured** on the physical switch — both sides must have matching LAG configuration.

LACP is an open industry standard — not VMware-specific — supported by a wide variety of physical switch vendors.

---

#### vDS NIC Teaming Methods Summary

| Method | Available On | VM Uses Multiple VMNICs? | Physical Switch Config |
|--------|-------------|--------------------------|----------------------|
| Originating Port ID | vSS + vDS | No | No Port Channel / LACP |
| Source MAC Hash | vSS + vDS | No | No Port Channel / LACP |
| IP Hash | vSS + vDS | Yes (per connection) | Port Channel / LACP required |
| Route Based on Physical NIC Load | vDS only | No (reassigned dynamically) | No Port Channel / LACP |
| LACP | vDS only | Yes (simultaneously) | LAG required on both sides |

---

> ### Lesson 19: Network I/O Control (NIOC)

#### What is Network I/O Control?

Network I/O Control (NIOC) is a vSphere Distributed Switch exclusive feature that enforces bandwidth controls on different types of network traffic sharing the same physical uplinks. It ensures critical traffic gets sufficient bandwidth during periods of contention.

---

#### NIOC Controls

| Control | Description |
|---------|-------------|
| Shares | Relative priority between traffic types — only enforced during contention |
| Limits | Hard cap on maximum bandwidth a traffic type can consume |
| Reservations | Guaranteed minimum bandwidth for a traffic type |

Example: VM traffic configured with twice as many shares as iSCSI traffic. During congestion, VM traffic gets double the bandwidth of iSCSI. During normal operation with no contention, shares have no effect.

---

#### Network Resource Pools

NIOC can also create network resource pools based on port groups. This allows per-port-group bandwidth allocation — controlling the relative priority of any specific group of VMs as their traffic leaves the vDS toward the physical network.

---

> ### Lesson 20: Demo — Configuring Network I/O Control

#### Enabling / Disabling NIOC

vSphere Client → Networking → select vDS → Configure → Properties → Edit → Enable or Disable Network I/O Control

---

#### Configuring System Traffic Allocations

Navigate to: Configure → Resource Allocation → System Traffic

This view shows all system traffic types and their current share values:

| Traffic Type | Default Shares | Priority |
|-------------|---------------|---------|
| Virtual Machine Traffic | 100 | High |
| vSAN | 50 (Normal) | Medium |
| Management | 50 (Normal) | Medium |
| vMotion | 50 (Normal) | Medium |
| iSCSI | 50 (Normal) | Medium |
| NFS | 50 (Normal) | Medium |

Share presets: Low, Normal, High, or Custom value.

Example: Changing vSAN from Normal (50) to a custom value of 75 gives vSAN higher priority than Management during contention but still lower than VM Traffic (100).

**Remember:** Shares are only enforced during contention. Under normal conditions with sufficient bandwidth, share values have no effect.

---

#### Configuring Reservations

Edit any traffic type → set a Reservation value in Mbps or Gbps.

Example: Setting a 1 Gbps reservation for Virtual Machine Traffic guarantees that 1 Gbps is always set aside for VM traffic — enforced 100% of the time regardless of contention, unlike shares.

| Control | Enforced When |
|---------|--------------|
| Shares | Only during bandwidth contention |
| Reservation | Always — guaranteed at all times |

---

#### Configuring Network Resource Pools

A Network Resource Pool allows you to allocate a portion of the VM traffic reservation to specific port groups.

**Prerequisite:** A bandwidth reservation must first be configured for Virtual Machine Traffic.

Steps:
1. Configure → Resource Allocation → Network Resource Pools → Add
2. Name the pool (e.g. High Priority)
3. Set a bandwidth allocation (e.g. 500 Mbps)
4. Associate distributed port groups with this pool

VMs connected to those port groups will benefit from the guaranteed bandwidth allocation of that pool.

---

> ### Lesson 21: Filtering and Tagging on vSphere Distributed Switch

#### What is Filtering and Tagging?

Filtering and Tagging is a vDS-exclusive feature similar to an Access Control List (ACL). It allows traffic rules to be applied to packets passing through the vDS — either allowing, dropping, or marking them for Quality of Service (QoS).

---

#### Filtering (Allow / Drop Rules)

Rules are created based on criteria such as:
- Source or destination IP address range
- TCP/UDP port number

When a packet matches a rule, the specified action is taken — allow or drop.

**Important:** Rules are enforced in order. The first matching rule is applied. Subsequent rules are not evaluated for that packet.

---

#### Tagging (QoS Marking)

Packets can be tagged before leaving the vDS so the physical network knows how to prioritize them. Two tag types are available:

| Tag Type | Layer | Purpose |
|----------|-------|---------|
| Class of Service (CoS) | Layer 2 | Prioritization at the physical switch level |
| DSCP (Differentiated Services Code Point) | Layer 3 | Prioritization at the router level |

Use case: Latency-sensitive traffic such as VoIP or streaming video can be marked with a high CoS or DSCP value. Once the traffic leaves the ESXi host, the physical network uses these tags to give it appropriate priority.

---

#### Where Rules Can Be Applied

Filtering and Tagging rules can be configured at three levels:
- Individual distributed port group
- Individual port on the distributed switch
- Uplink port or uplink group

---

> ### Lesson 22: NetFlow on vSphere Distributed Switch

#### What is NetFlow?

NetFlow is an industry-standard network traffic monitoring protocol that has been used in physical networks for many years. On the vDS, it collects summaries of all traffic flowing through the switch and sends them to a centralized NetFlow Collector for analysis.

NetFlow is a vDS-exclusive feature — not available on the Standard Switch.

---

#### How NetFlow Works

1. A VM sends traffic through the vDS
2. The vDS records a summary of that traffic including:
   - Source IP address and port
   - Destination IP address
   - Protocol and port number
   - Date, time, and amount of data sent
3. That summary is forwarded to a NetFlow Collector
4. The collector builds a historical record of all traffic flowing through the vDS

---

#### Use Cases

| Scenario | How NetFlow Helps |
|----------|------------------|
| Intermittent slowdowns at a specific time | Review traffic history from that time period after the fact |
| Physical adapters constantly overwhelmed | Identify which VMs are generating the most traffic and when |
| Security investigation | Track which VMs communicated with which destinations and on which ports |

---

#### NetFlow Collectors

The NetFlow Collector is not a vSphere product — it is a separate third-party tool. Common options include:

- SolarWinds
- Nagios

These tools visualize and analyze the NetFlow data sent by the vDS.

---

#### Demo: Configuring NetFlow on the vDS

Access: vSphere Client → Networking → select vDS → Configure → NetFlow → Edit

Configuration required:

| Setting | Description |
|---------|-------------|
| NetFlow Collector IP | IP address of the server that will receive and store the traffic flow records |
| vDS IP address | The vDS itself needs an IP address to communicate with the NetFlow Collector |
| Sampling rate | How frequently traffic samples are captured |
| Flow timeout | How long an inactive flow is tracked before the record is closed |

Once configured, the vDS will continuously send traffic flow summaries to the collector. The collector builds a searchable historical record that can be queried after the fact — for example, to investigate what traffic was occurring at 2:00 PM yesterday when users reported slowness.

---

> ### Lesson 23: Port Mirroring on vSphere Distributed Switch

#### What is Port Mirroring?

Port Mirroring duplicates traffic from one or more ports and sends a copy to another port or destination where a sniffer or analysis tool is running. It is a vDS-exclusive feature.

Example: All traffic destined for VM at 10.1.1.11 is copied and also sent to a sniffer VM at another IP address — allowing passive analysis without interrupting the original traffic flow.

---

#### Port Mirroring Session Types

| Session Type | Description |
|-------------|-------------|
| Distributed Port Mirroring | Mirrors traffic between ports on the same vDS — source and destination must be on the same host |
| Remote Mirroring Source | Captures packets from distributed ports and sends them to an uplink on another ESXi host |
| Remote Mirroring Destination | Mirrors packets from VLANs to distributed ports on the vDS |
| Encapsulated Remote Mirroring Source | Captures packets from distributed ports and mirrors them to the IP address of a remote agent (e.g. a sniffer VM) — most commonly used |

---

#### Demo: Configuring a Port Mirroring Session

Access: vSphere Client → Networking → select vDS → Configure → Port Mirroring → New

**Step 1: Choose session type**
Select from the available session types (Distributed Port Mirroring is most commonly used).

**Step 2: Configure session settings**

| Setting | Description |
|---------|-------------|
| Session name | A name to identify this mirroring session (e.g. RickDemo) |
| Status | Enabled or Disabled |
| Allow normal I/O on destination ports | Whether the destination port (sniffer VM) can also send and receive regular traffic alongside mirrored traffic |
| Sampling rate | Reduce the volume of mirrored data if needed |

**Step 3: Select source port(s)**
Choose the distributed port(s) whose traffic should be mirrored — typically the port the VM being monitored is connected to.

**Step 4: Select destination port**
Choose the port the sniffer VM is connected to — all mirrored traffic will be forwarded here.

Click Finish to activate the session.

> ### Lesson 24: Network Health Check on vSphere Distributed Switch

#### What is Network Health Check?

Network Health Check is a vDS-exclusive feature that validates the configuration of the vDS against the physical switch it is connected to. It is **disabled by default** and must be manually enabled.

Its purpose is to detect configuration mismatches between the virtual and physical network before they cause problems.

---

#### What Network Health Check Detects

| Configuration | What is Checked | Example Problem |
|---------------|----------------|-----------------|
| MTU | vDS MTU matches physical switch MTU | vDS set to 9000 (Jumbo Frames), physical switch set to 1500 — causes fragmentation |
| NIC Teaming | Physical switch is configured appropriately for the selected teaming method | vDS using IP Hash but physical switch has no Port Channel configured — incompatible |
| VLAN | VLANs configured on vDS port groups exist on the physical switch | vDS port group tagged VLAN 20 but VLAN 20 not configured on the physical switch |

---

#### Why It Matters

These mismatches are easy to introduce and can be difficult to diagnose without tooling. For example:
- A Jumbo Frame mismatch causes the physical switch CPU to fragment every oversized frame — major performance impact
- An IP Hash / Port Channel mismatch causes inconsistent or broken connectivity
- A missing VLAN causes all traffic on that port group to be silently dropped

Network Health Check surfaces all of these issues in one place so they can be identified and resolved.

---

#### Demo: Enabling and Using Health Check

Access: vSphere Client → Networking → select vDS → Configure → Health Check → Edit

Enable both health check services (MTU/VLAN and NIC Teaming/Failover).

**Security note:** Health Check introduces a minor security vulnerability when left running. Best practice is to enable it only when needed — for example after setting up new ESXi hosts or a new physical switch — then disable it again once validation is complete.

**What gets validated when enabled:**

| Check | What it validates |
|-------|-----------------|
| MTU | vDS MTU matches physical switch MTU |
| VLAN | VLANs on vDS port groups are configured on the physical switch |
| NIC Teaming / Failover | Physical switch Port Channel config is compatible with the vDS teaming method selected |

Example mismatches Health Check will flag:
- vDS configured for IP Hash → physical switch has no Port Channel → incompatible
- vDS port group using VLAN 10 → VLAN 10 not configured on physical switch → traffic dropped
- vDS MTU 9000 → physical switch MTU 1500 → fragmentation and performance degradation

---

#### vDS-Only Features Quick Reference

For exam preparation — features exclusive to the vSphere Distributed Switch:

| Feature | vSS | vDS |
|---------|-----|-----|
| Private VLANs | No | Yes |
| LLDP | No | Yes |
| NetFlow | No | Yes |
| Port Mirroring | No | Yes |
| Network Health Check | No | Yes |
| Network I/O Control | No | Yes |
| Route Based on Physical NIC Load | No | Yes |
| LACP | No | Yes |
| Filtering and Tagging | No | Yes |

> ### Lesson 25: Host Networking Rollback

#### What is Host Networking Rollback?

Host networking rollback is an automatic safety mechanism that triggers whenever a network configuration change breaks the connection between an ESXi host and vCenter. The problematic change is automatically undone and the host reconnects to vCenter without any manual intervention.

---

#### How It Works

1. An administrator makes a network configuration change on an ESXi host (e.g. changes the management VMkernel port IP address to an incorrect value)
2. The change breaks connectivity between the host and vCenter
3. The host detects the loss of vCenter communication
4. The change is automatically reverted
5. The host reconnects to vCenter

---

#### Why It Matters

Without rollback, a misconfigured management VMkernel port would require one of the following to recover:
- Remote console access to the ESXi host
- Physical access to the datacenter (keyboard, monitor, and mouse connected directly to the host)

Rollback eliminates the need for either by automatically restoring the last working configuration.

---

> ### Lesson 26: Distributed Switch Rollback

#### What is Distributed Switch Rollback?

Distributed switch rollback is a recovery mechanism for configuration changes made to the vDS itself that result in ESXi hosts losing connectivity to vCenter. Unlike host networking rollback which is fully automatic, distributed switch rollback requires manual intervention to resolve.

---

#### Differences from Host Networking Rollback

| Feature | Host Networking Rollback | Distributed Switch Rollback |
|---------|--------------------------|----------------------------|
| Trigger | Change to host-level network config breaks host-vCenter connection | Change to vDS config breaks host-vCenter connection |
| Recovery | Automatic — change is reverted without intervention | Manual — administrator must fix the issue |

---

#### What Can Trigger a Distributed Switch Rollback

- Changing the MTU of the vDS to an incompatible value
- Selecting an incompatible NIC Teaming method
- Configuring an incorrect VLAN on a port group

Example: Management VMkernel port is on VLAN 10. Administrator changes the vDS port group VLAN to 20. Physical switch is still configured for VLAN 10. ESXi hosts lose connectivity to vCenter.

---

#### Recovery Options

When a vDS configuration change disconnects hosts from vCenter, two options are available:

| Option | Description |
|--------|-------------|
| Fix the physical switch | Update the physical switch configuration to match the new vDS settings |
| Restore vDS from backup | Restore a saved vDS configuration backup to revert the vDS to its last working state |

---

#### Section 4 Review — vDS Feature Summary

| Feature | Available On | Purpose |
|---------|-------------|---------|
| CDP | vSS + vDS | Discover physical Cisco switch info (port, IP, software version) |
| LLDP | vDS only | Discover physical switch info from any vendor |
| NetFlow | vDS only | Send traffic flow summaries to a collector for historical analysis |
| Traffic Filtering | vDS only | Allow or deny traffic based on IP/port criteria |
| Traffic Marking (CoS/DSCP) | vDS only | Tag traffic for QoS prioritization in the physical network |
| Network I/O Control | vDS only | Prioritize traffic types using shares, limits, and reservations |
| Port Mirroring | vDS only | Duplicate traffic to a sniffer for monitoring and analysis |
| Host Networking Rollback | All | Automatically reverts host-level changes that disconnect host from vCenter |
| Distributed Switch Rollback | vDS only | Manual recovery when vDS config change disconnects hosts from vCenter |

---
## Section 5: Configure and Manage vSphere 8 Storage

This section covers vSphere storage concepts and configuration. Topics include virtual disks, datastores, VMFS, NFS, storage hardware types, and vSAN — VMware's software-defined storage solution that uses the local physical storage of ESXi hosts to create a shared datastore.

---

> ### Lesson 1: Introduction to Storage Virtualization

#### How Storage is Presented to a VM

A VM's guest operating system has no awareness that it is running on virtual hardware. It sees a virtual SCSI controller as if it were a real physical storage adapter. When the OS needs to read or write data, it sends SCSI commands to that virtual SCSI controller.

The flow of a storage operation:
1. Guest OS generates a SCSI command
2. SCSI command is sent to the virtual SCSI controller
3. The hypervisor intercepts the command and determines which datastore and VMDK file to direct it to
4. The hypervisor uses a physical storage adapter to send the command to the appropriate datastore

---

#### What is a Datastore?

A datastore is a chunk of storage capacity made available to virtual machines. VMDKs (virtual disk files) are stored on datastores. A datastore can be local to one host or shared across multiple hosts.

---

#### Virtual Disk Provisioning Types

**Thin Provisioned**
- Only the space actually used by data is consumed on the datastore
- Example: 80 GB virtual disk with 40 GB of data → only 40 GB consumed
- Benefit: Saves storage capacity
- Risk: No guarantee that space will be available when the VM needs to write more data

**Thick Provisioned (Lazy Zeroed)**
- The full allocated size is reserved on the datastore immediately
- Blocks are zeroed on first write (not up front)
- Example: 80 GB virtual disk → 80 GB reserved immediately, zeros written as data is written
- Risk: First write to each block is slightly slower due to zeroing

**Thick Provisioned (Eager Zeroed)**
- The full allocated size is reserved and all blocks are zeroed immediately at creation time
- Takes longer to create than lazy zeroed
- Best performance for storage-intensive workloads (e.g. databases) because zeroing is done up front — no zeroing penalty during writes
- Recommended for: high-performance or I/O-intensive VMs

---

#### Provisioning Types Comparison

| Type | Space Reserved | Zeroed | Creation Speed | Write Performance |
|------|---------------|--------|---------------|------------------|
| Thin | Only used space | On demand | Fast | Normal |
| Thick Lazy Zeroed | Full size | On first write | Medium | Slight penalty on new blocks |
| Thick Eager Zeroed | Full size | Up front at creation | Slow | Best — no zeroing penalty |

---

> ### Lesson 2: VMFS vs NFS Storage

vSphere datastores can be built on two different file system types — VMFS and NFS. The fundamental difference is who owns and manages the file system.

---

#### VMFS (Virtual Machine File System)

Used with block-based storage technologies:

| Storage Type | Connection Method |
|-------------|------------------|
| iSCSI | Ethernet network |
| Fibre Channel (FC) | Dedicated FC network |
| Fibre Channel over Ethernet (FCoE) | Ethernet network |
| Direct Attached Storage (DAS) | Local physical disks on the ESXi host |

**How VMFS works:**

1. The storage array aggregates physical disk capacity (using RAID) into one large pool
2. The storage administrator carves that pool into **LUNs (Logical Unit Numbers)** — raw, unformatted chunks of capacity
3. The ESXi host discovers available LUNs via **target discovery** (e.g. sending a "send targets" request to the iSCSI target IP)
4. The ESXi host formats the LUN with the **VMFS file system** — the ESXi host owns and manages this file system
5. The formatted LUN becomes a VMFS datastore

Key point: The storage array delivers raw, unformatted disk space. The ESXi host does the formatting.

---

#### NFS (Network File System)

NFS is fundamentally different — the ESXi host never formats anything.

- The NFS server owns and manages its own file system entirely
- The ESXi host connects to the NFS server over an Ethernet network using a VMkernel port
- Creating an NFS datastore is equivalent to creating a shared folder (called an **NFS export**) on the NFS server
- The ESXi host reads and writes files within that shared folder

---

#### VMFS vs NFS Comparison

| Feature | VMFS | NFS |
|---------|------|-----|
| File system owner | ESXi host formats and owns it | NFS server owns it |
| Storage types | iSCSI, FC, FCoE, DAS | NFS server over Ethernet |
| LUNs required | Yes | No |
| Boot from SAN | Yes | No |
| Raw Device Mapping (RDM) | Yes | No |
| ESXi formats the storage | Yes | No |
| Datastore type | Block-based | File-based (shared folder / export) |

---

#### Raw Device Mapping (RDM)

An RDM gives a VM direct access to a raw, unformatted LUN on a block storage device — bypassing the VMFS file system entirely. Only supported with iSCSI, FC, FCoE, and DAS. Not supported with NFS.

---

#### How NFS Storage Traffic Flows

1. Guest OS issues SCSI commands to the virtual SCSI controller
2. The ESXi storage adapter packages those commands in NFS format
3. The storage adapter sends the commands to a VMkernel port on the virtual switch
4. The VMkernel port pushes the traffic out a physical VMNIC toward the NFS server IP address
5. The NFS server receives the request and reads/writes data in the shared export folder

---

> ### Lesson 3: NFS Version 3 vs NFS Version 4.1

vSphere supports two versions of NFS for datastore connectivity. NFS 4.1 is a significant improvement over Version 3 in terms of security, load balancing, and authentication.

---

#### NFS Version 3

| Characteristic | Detail |
|---------------|--------|
| Encryption | None — all traffic is unencrypted |
| Network requirement | Trusted network only |
| Connections | Single connection for I/O |
| IP addresses per datastore | Single IP address only |
| Load balancing challenge | With IP Hash, all traffic goes to one destination IP → only one physical VMNIC is ever used |
| Authentication | Requires root-level access to the NFS server |
| Security concern | The NFS server must have "no root squash" enabled — not the default, and reduces NFS server security |

---

#### NFS Version 4.1

| Characteristic | Detail |
|---------------|--------|
| Encryption | Headers are signed and encrypted |
| Security improvement | Kerberos authentication supported |
| Connections | Multiple IP addresses per datastore (multipathing) |
| Load balancing | Multiple destination IPs → IP Hash can distribute traffic across multiple VMNICs |
| Authentication | No root access required — uses Kerberos credentials instead |
| Kerberos requirement | Credentials must match on all ESXi hosts using the same datastore |
| Directory integration | Uses Active Directory and a Key Distribution Center (KDC) for Kerberos |

---

#### NFS v3 vs NFS v4.1 Comparison

| Feature | NFS v3 | NFS v4.1 |
|---------|--------|----------|
| Traffic encryption | No | Yes (headers signed and encrypted) |
| Multipathing (multiple IPs) | No | Yes |
| Load balancing across VMNICs | Difficult — single IP | Effective — multiple IPs |
| Root access required | Yes (no root squash needed) | No — Kerberos used instead |
| Kerberos support | No | Yes |
| Active Directory integration | No | Yes |

---

> ### Lesson 4: Demo — Creating an NFS Datastore

#### Creating the NFS Datastore

Access: vSphere Client → Shortcuts → Storage → right-click the Datacenter → Storage → Add New Datastore

Steps:

1. Select NFS as the datastore type
2. Choose NFS version (3 or 4.1)
3. Configure the datastore:

| Setting | Description |
|---------|-------------|
| Name | A label for the datastore (e.g. NFS-Demo) |
| Folder / Export path | The NFS export path on the server (e.g. /Share) |
| NFS server address | IP address or hostname of the NFS server |
| Access mode | Read/Write (default) or Read/Only |

4. Select which ESXi hosts should have access to this datastore
5. Click Finish

Note: No disk size is specified and no formatting takes place. The ESXi host is simply mounting a shared folder on the NFS server.

---

#### What Happens Behind the Scenes

The NFS datastore is just a shared directory on the NFS server. When the datastore is created, its used space reflects the existing content in that shared folder — not anything created by vSphere.

NFS Version 3 requirements on the server:
- Root access must be allowed (no root squash must be disabled)
- Access should be restricted to specific IP address ranges for security
- No Kerberos authentication — NFS v3 does not support it

---

#### Mounting a Datastore on Additional Hosts

When a new NFS datastore is created it is mounted on the selected hosts only. To give access to additional hosts added later, the datastore must be mounted on those hosts separately.

---

#### Unmounting vs Deleting a Datastore

| Action | What Happens |
|--------|-------------|
| Unmount | Removes the datastore from the ESXi host — data on the NFS server is NOT deleted |
| Delete | Destroys the datastore and its contents |

Unmounting is simply detaching the shared folder from the host — the folder and its contents remain on the NFS server. Any VMs running on that datastore must be addressed before unmounting.

---

> ### Lesson 5: Fibre Channel Storage

Fibre Channel (FC) is a block-based storage technology that uses the VMFS file system — similar to iSCSI in concept but using a dedicated Fibre Channel network instead of Ethernet. All VMFS-specific features apply: boot from SAN, raw device mapping, and ESXi-formatted LUNs.

---

#### Fibre Channel Architecture

| Component | Description |
|-----------|-------------|
| Fibre Channel Switch Fabric | The dedicated FC network — switches connected via fibre optic cables (equivalent of the Ethernet network in iSCSI) |
| Storage Processors | The CPU/interface of the storage array — same concept as iSCSI |
| Disk Aggregate | Total pooled storage capacity of the array |
| LUNs | Logical Unit Numbers — raw, unformatted chunks carved from the aggregate |
| HBA (Host Bus Adapter) | The physical Fibre Channel adapter installed in the ESXi host — terminates the fibre optic connection |

---

#### How FC Storage Commands Flow

1. VM generates SCSI commands
2. Hypervisor forwards commands to the Fibre Channel HBA
3. HBA prepares the SCSI commands for transmission across the FC fabric
4. Commands travel through the Fibre Channel Switch Fabric to the storage array
5. Data is read from or written to the LUN

---

#### Multipathing in Fibre Channel

Connecting to two separate FC switches provides path redundancy:

| Failure Scenario | Result with Multipathing |
|-----------------|--------------------------|
| One FC port on ESXi host fails | Other port continues — no storage outage |
| One FC switch fails | Other switch provides a valid path |
| One storage processor fails | Other processor still reachable |

Multipathing modes:
- **Active/Active (Round Robin)** — both FC ports used simultaneously, load balanced
- **Active/Passive** — one port active, other on standby

---

#### HBA Redundancy Best Practice

A dual-port HBA (one card, two ports) is convenient but a single card failure can take down both paths. For true redundancy:

- Use **two single-port HBAs** (two separate cards, one port each)
- A single hardware failure then only affects one path — the other remains active

---

> ### Lesson 6: Fibre Channel Zoning and LUN Masking

Zoning and LUN Masking are two complementary methods for controlling which ESXi hosts can access which storage resources in a Fibre Channel environment.

---

#### Zoning

Zoning segments the Fibre Channel Switch Fabric to control which devices can communicate with each other — similar in concept to VLANs in Ethernet networking.

- Configured on the **Fibre Channel Switch Fabric** itself — not on the hosts or storage array
- Ports in the same zone can communicate; ports in different zones cannot
- Allows multiple teams or environments (e.g. Dev and Prod) to share the same FC fabric while being isolated from each other

Example:
- Prod ESXi Host port → Blue Zone → Production Storage Array
- Dev ESXi Host port → Green Zone → Dev Storage Array
- Dev host cannot reach the Production Storage Array because they are in different zones

---

#### LUN Masking

LUN Masking controls which hosts can access which specific LUNs on a storage array — even when those LUNs are on the same array (where Zoning alone cannot help).

- Configured on the **ESXi hosts and the Storage Array**
- Uses **World Wide Names (WWNs)** to identify each device in the FC network
- The storage array maintains an access list: which WWN can access which LUN

Example:
- Production LUN → accessible only by the Prod ESXi Host WWN
- Dev LUN → accessible only by the Dev ESXi Host WWN
- Both LUNs are on the same array — Zoning cannot separate them — LUN Masking handles this

---

#### Zoning vs LUN Masking

| Feature | Zoning | LUN Masking |
|---------|--------|-------------|
| Configured on | FC Switch Fabric | ESXi hosts and Storage Array |
| Uses | Port-based segmentation | World Wide Name (WWN) access lists |
| Comparable to | VLANs | ACLs / firewall rules |
| Separates | Hosts from entire storage arrays | Hosts from specific LUNs on the same array |

---

#### N-Port ID Virtualization (NPIV)

NPIV allows individual VMs to be assigned their own Fibre Channel World Wide Names and N-Ports — enabling LUN Masking at the per-VM level rather than the per-host level.

Benefits of NPIV:
- LUN Masking can be applied per VM (not just per host)
- QoS can be configured on a per-VM basis
- Storage array usage can be monitored per VM

Drawback:
- Requires a **Raw Device Mapping (RDM)** for each VM
- RDMs can interfere with Storage vMotion

---

#### Fibre Channel Summary

| Feature | Supported |
|---------|----------|
| VMFS file system | Yes |
| Boot from SAN | Yes |
| Raw Device Mapping | Yes |
| Software FC adapter | No — requires physical HBA |
| Zoning | Yes — on FC Switch Fabric |
| LUN Masking | Yes — on hosts and storage array |
| NPIV | Yes — per-VM WWN assignment |

---

> ### Lesson 7: Fibre Channel over Ethernet (FCoE)

FCoE is another block-based VMFS storage option. It is conceptually identical to Fibre Channel — same storage array architecture, same LUNs, same VMFS formatting — but uses Ethernet switches instead of a dedicated FC Switch Fabric for connectivity.

All VMFS features apply: Boot from SAN, Raw Device Mapping, and ESXi-formatted datastores.

---

#### FCoE Architecture

| Component | Description |
|-----------|-------------|
| Storage Array | Same as FC — aggregate, LUNs, storage processors |
| FCoE Switch | A special Ethernet switch that supports Fibre Channel over Ethernet — replaces the FC Switch Fabric |
| CNA (Converged Network Adapter) | A physical hardware card installed in the ESXi host — handles both storage and network traffic on a single interface |

---

#### How FCoE Separates Traffic

The CNA carries both storage traffic and regular network traffic on the same physical interface, separated by VLAN:

| VLAN | Traffic Type |
|------|-------------|
| Storage VLAN (e.g. Red) | FCoE storage commands |
| Network VLAN (e.g. Blue) | Regular VM network traffic |

The FCoE switch differentiates between them using the VLAN tag.

---

#### FCoE Connectivity Options

**Option 1: Hardware CNA (Converged Network Adapter)**
- A physical card installed in the ESXi host
- Handles all FCoE processing in dedicated hardware
- No additional CPU overhead on the ESXi host
- Specific to FCoE — cannot be used for FC or iSCSI

**Option 2: Software FCoE Adapter**
- A software-based storage adapter created on the ESXi host
- Uses a VMkernel port on a virtual switch with a physical VMNIC
- No dedicated hardware required — saves cost
- Downside: additional CPU overhead on the ESXi host due to software processing

---

#### FCoE Redundancy

For path redundancy, two VMkernel ports on two separate virtual switches (each with its own VMNIC) can be connected to two different FCoE switches. Round Robin load balancing distributes traffic across both paths.

Tolerates:
- Single VMNIC failure
- Single FCoE switch failure
- Single storage processor failure

---

#### FCoE vs FC vs iSCSI Comparison

| Feature | FC | FCoE | iSCSI |
|---------|----|------|-------|
| Network type | FC Switch Fabric | Ethernet (FCoE switch) | Ethernet |
| File system | VMFS | VMFS | VMFS |
| Boot from SAN | Yes | Yes | Yes |
| Raw Device Mapping | Yes | Yes | Yes |
| Hardware adapter | HBA | CNA (or software) | NIC (or software) |
| Software adapter option | No | Yes | Yes |

---

> ### Lesson 8: iSCSI Storage

iSCSI is a block-based VMFS storage technology that uses a standard Ethernet network to connect ESXi hosts to the storage array. Conceptually similar to Fibre Channel — same aggregate, LUN, and VMFS model — but uses Ethernet instead of a dedicated FC fabric.

All VMFS features apply: Boot from SAN, Raw Device Mapping, ESXi-formatted datastores.

---

#### iSCSI Initiator Options

There are three ways to connect an ESXi host to an iSCSI storage array:

**1. Software iSCSI Initiator**
- Storage adapter created entirely in software on the ESXi host
- Requires a VMkernel port on a virtual switch with a physical VMNIC
- No dedicated hardware required — lowest cost
- Downside: additional CPU overhead on the ESXi host

**2. Dependent Hardware iSCSI Initiator**
- A physical hardware card is installed in the ESXi host
- Still requires VMkernel ports to be configured — the hardware depends on the virtual switch
- Reduces some CPU overhead compared to software iSCSI

**3. Independent Hardware iSCSI Initiator**
- A physical card with built-in Ethernet ports and its own IP addresses
- Handles multipathing and all iSCSI functions entirely in hardware
- No VMkernel port or software adapter needed
- Highest cost but lowest CPU overhead on the ESXi host

---

#### iSCSI Initiator Comparison

| Feature | Software | Dependent Hardware | Independent Hardware |
|---------|----------|--------------------|---------------------|
| Hardware required | No | Yes | Yes |
| VMkernel port required | Yes | Yes | No |
| CPU overhead on ESXi host | High | Medium | Low |
| Cost | Lowest | Medium | Highest |

---

#### Redundancy with iSCSI

For path redundancy, configure multiple VMkernel ports on separate virtual switches, each with its own VMNIC connected to a separate Ethernet switch. Use Round Robin multipathing to distribute storage traffic across both paths.

Tolerates: single VMnic failure, single Ethernet switch failure, single storage processor failure.

---

#### Dynamic Discovery — How ESXi Finds LUNs

1. Configure the iSCSI target (IP address of the storage array) on the ESXi host
2. Perform a rescan on the storage adapter
3. The adapter sends a **Send Targets request** to the storage array
4. The storage array responds with a **Send Targets response** — a list of all available LUNs
5. LUNs already in use are filtered out automatically
6. Available LUNs can now be formatted with VMFS to create datastores

---

#### CHAP Authentication

iSCSI supports **CHAP (Challenge Handshake Authentication Protocol)** to verify that both the initiator (ESXi host) and the target (storage array) are legitimate devices.

**Important for exam:** iSCSI is the only vSphere storage technology that supports CHAP authentication.

---

> ### Lesson 9: Demo — Creating a Software iSCSI Initiator and VMFS Datastore

#### Step 1: Add a Software iSCSI Adapter

Access: vSphere Client → Hosts and Clusters → ESXi host → Configure → Storage Adapters → Add Software Adapter → Add Software iSCSI Adapter → OK

The new adapter (e.g. vmhba65) appears in the list. At this point it exists but cannot connect to anything yet.

---

#### Step 2: Configure Dynamic Discovery

1. Click the new iSCSI adapter → Properties → Dynamic Discovery → Add
2. Enter the IP address of the iSCSI target (the storage array)
3. Leave the default port (3260)
4. Click OK

This tells the ESXi host where to query for available LUNs.

---

#### Step 3: Rescan Storage

After configuring dynamic discovery, rescan the storage adapter. The ESXi host will:
1. Send a Send Targets request to the iSCSI target IP
2. Receive a list of available LUNs
3. Display discovered LUNs and any existing datastores under the adapter's Devices tab

---

#### Step 4: Create a VMFS Datastore on a LUN

Right-click the ESXi host → Storage → New Datastore

| Setting | Value |
|---------|-------|
| Type | VMFS |
| LUN | Select an available unformatted LUN |
| VMFS Version | VMFS 6 (recommended for all new datastores) |
| Partition configuration | Leave defaults unless specific block size or partition layout is needed |

Click Finish. The datastore appears in the Storage view and is ready to use.

---

#### Delete vs Unmount

| Action | What Happens |
|--------|-------------|
| Unmount (NFS) | Detaches the datastore from the host — data is preserved |
| Delete (VMFS) | Permanently destroys the datastore and all data on the LUN |

Deleting a VMFS datastore frees the LUN so it can be used to create a new datastore.

---

#### iSCSI Target Configuration (Server Side)

For iSCSI to work, the storage server must:
- Have iSCSI target services enabled
- Have the ESXi host's initiator added to the trusted/connected initiators list
- Present available virtual disks (LUNs) to the initiator

In this lab, Windows Server 2016 is used as the iSCSI target with iSCSI storage services enabled under Server Manager.

---

> ### Lesson 10: vSAN — Original Storage Architecture

#### What is vSAN?

vSAN (Virtual SAN) is VMware's software-defined storage solution that aggregates the local physical storage of ESXi hosts into a shared datastore accessible by all hosts in the cluster. This lesson covers the **Original Storage Architecture** — the traditional vSAN design using disk groups with cache and capacity tiers.

---

#### Prerequisites for vSAN

- Supported vSphere and vCenter version
- Supported hardware on each ESXi host
- An ESXi Host Cluster must be created first
- Each host must have a **VMkernel port tagged for vSAN traffic**
- Recommended: dedicated physical network for vSAN traffic with redundant switches

---

#### Network Design

Each ESXi host in the vSAN cluster should have:
- Two physical VMNICs (e.g. 10 GbE), each connected to a separate physical switch
- A VMkernel port with an IP address tagged for vSAN traffic
- Dedicated switches for vSAN traffic only (no other traffic on the vSAN network)

The vSAN VMkernel port handles all storage traffic flowing between hosts — reads, writes, and mirror operations.

---

#### Hybrid Configuration: Cache and Capacity Tiers

In the hybrid configuration, each ESXi host has two types of storage devices:

| Tier | Device Type | Purpose |
|------|------------|---------|
| Cache | SSD (flash) | Read cache (70%) and write buffer (30%) |
| Capacity | HDD (magnetic disks) | Bulk storage for VM data |

---

#### How Reads Work (Hybrid)

1. VM requests data via the vSAN VMkernel port
2. vSAN checks the **SSD read cache** first (70% of SSD dedicated to this)
3. **Cache hit:** Data is served from SSD — very fast
4. **Cache miss:** Data must be read from the HDD capacity tier — much slower

---

#### How Writes Work

vSAN mirrors VM data across multiple hosts for fault tolerance:

1. VM writes data
2. The write is sent simultaneously to two hosts (primary and mirror copy)
3. On each host, the write first hits the **SSD write buffer** — fast acknowledgment
4. In the background, vSAN destages the data from the write buffer to the HDD capacity tier

The write buffer analogy: dropping a library book at the front desk — the VM is done immediately, and vSAN handles shelving it afterward.

---

#### VM Object Storage

- A VM's active VMDK resides on one host
- A mirror copy resides on another host for fault tolerance
- If the primary host fails, the mirror ensures no data loss
- The vSAN VMkernel port facilitates traffic between the VM and wherever its VMDK physically resides

---

> ### Lesson 11: vSAN vs Traditional Storage Array

#### How Traditional Storage Works

In a traditional setup, a VM is presented with a virtual SCSI controller — the guest OS thinks it has real physical storage hardware. When the OS reads or writes data, it sends SCSI commands through that virtual controller to the hypervisor. The hypervisor then forwards those commands across a physical storage network (Ethernet for iSCSI, or Fibre Channel) to a dedicated physical storage array where the VM's files (VMDK, VMX, etc.) reside on a LUN formatted with VMFS.

---

#### Why Shared Storage Matters

Shared storage is what makes advanced vSphere features possible. When a datastore is accessible from multiple ESXi hosts simultaneously, it enables:

| Feature | Why it needs shared storage |
|---------|----------------------------|
| vMotion | VM moves to another host but still needs its files |
| High Availability | Failed VM restarts on another host — files must be reachable |
| Fault Tolerance | Zero-downtime failover requires both hosts to access the same data |
| DRS | VMs move between hosts automatically for load balancing |

Without shared storage, a VM's files would only be accessible from the one host they're physically attached to.

---

#### How vSAN Provides Shared Storage Differently

Instead of a dedicated external storage array, vSAN uses the local physical disks inside the ESXi hosts themselves. A VM's VMDK object is stored on the local storage of one host, but is accessible from all other hosts in the cluster via the vSAN VMkernel network. A mirror copy is also kept on another host for fault tolerance.

From the VM's perspective, nothing changes — it still sends SCSI commands through a virtual SCSI controller. It has no idea whether it's on a physical storage array, vSAN, or local disks.

---

#### vSAN vs Traditional Storage Array

| Aspect | Traditional VMFS/NFS | vSAN |
|--------|---------------------|------|
| Storage hardware | Dedicated physical storage array | Local disks inside ESXi hosts |
| Network | Dedicated storage network (FC, iSCSI, Ethernet) | vSAN VMkernel network between hosts |
| Shared storage | Yes — all hosts connect to the same array | Yes — created virtually across host local disks |
| Supports HA, vMotion, DRS, FT | Yes | Yes |
| VM awareness | VM does not know which storage it is on | VM does not know — same virtual SCSI abstraction |
| Cost model | Separate storage array purchase | Storage built into ESXi hosts |

---

> ### Lesson 12: vSAN Express Storage Architecture

#### Why a New Architecture?

The Original Storage Architecture was designed for hybrid environments — spinning HDD as the capacity tier with SSD as a cache in front of it. This made sense when HDDs were the most affordable storage option and SSDs were expensive. That two-tier approach is still supported in vSAN 8, but it is increasingly less relevant as all-flash configurations have become affordable.

The **vSAN Express Storage Architecture (ESA)** is VMware's modern vSAN design, built specifically for all-flash environments using fast NVMe-based TLC flash devices. These devices are significantly faster and provide much higher IOPS than spinning disks, and the original two-tier architecture was not designed to take full advantage of them.

---

#### Key Differences from the Original Architecture

**No More Disk Groups**

In the Original Storage Architecture, physical storage devices on each host were organized into disk groups — each group had a dedicated cache device (SSD) and one or more capacity devices (HDD). In the Express Storage Architecture, disk groups are gone entirely. There is no separate cache tier and no dedicated cache device.

**Log-Structured File System**

The Express Storage Architecture uses a new log-structured file system that changes how vSAN stores data internally. Because all storage devices are fast NVMe flash, there is no longer a need for a cache layer to buffer writes before destaging them to slower disks.

**Faster Network Required**

| Architecture | Recommended Network Speed |
|-------------|--------------------------|
| Original Storage Architecture | 10 GbE |
| Express Storage Architecture | 25 GbE |

The faster storage devices demand a faster network to avoid the network becoming the bottleneck.

---

#### Additional Enhancements in ESA

- More efficient snapshot mechanism — significantly improves performance compared to the original snapshot implementation
- Ability to use RAID 5 across a three-node cluster while achieving performance comparable to RAID 1
- Multiple other performance and efficiency improvements beyond the scope of this course

---

#### Which Architecture Should You Use?

The choice depends on the hardware available:

| Scenario | Recommended Architecture |
|----------|--------------------------|
| Hybrid environment with HDDs as capacity | Original Storage Architecture |
| All-flash environment with NVMe devices | Express Storage Architecture |

Both architectures can coexist — different clusters within the same environment can run different architectures.

---

> ### Lesson 13: Storage Configuration Maximums

The following storage maximums are from the VMware Configuration Maximums page for vSphere 8. These are exam-relevant values worth memorizing.

---

#### Virtual Machine Storage Maximum

| Item | Maximum |
|------|---------|
| Virtual disk size | 62 TB |

---

#### ESXi Host Storage Maximums

| Item | Maximum |
|------|---------|
| VMFS datastore size | 64 TB |
| Raw Device Mapping — Virtual Compatibility Mode | 62 TB |
| Raw Device Mapping — Physical Compatibility Mode | 64 TB |
| VMFS 5 and VMFS 6 block size | 1 MB |

---
## Section 6: Deploy and Administer vSphere 8 VMs

This section covers virtual machine administration in vSphere 8. Topics include creating VMs, installing VMware Tools, working with thin provisioned disks, VM templates, OVF files, and snapshots — all the critical skills needed to manage virtual machines and pass the VCTA exam.

---

> ### Lesson 1: Demo — Creating a New Virtual Machine

#### Starting the Wizard

Access: vSphere Client → Hosts and Clusters → right-click the ESXi host or cluster → New Virtual Machine → Create a new virtual machine

---

#### Step 1: Name and Location

Give the VM a name (e.g. RickServer2016) and choose the folder/datacenter to place it in.

---

#### Step 2: Compute Resource

Select which ESXi host or cluster the VM will run on.

---

#### Step 3: Storage

Select the datastore where all VM files (VMX, VMDK, etc.) will be stored.

---

#### Step 4: Compatibility

Choose the virtual hardware version. This determines which ESXi hosts the VM can run on.

| Rule | Detail |
|------|--------|
| Latest version | VM can run on the latest ESXi hosts only |
| Older version | VM can run on older and newer hosts |

Best practice: choose the version that matches the oldest ESXi host the VM might ever run on. If all hosts are on the latest version, select the latest.

---

#### Step 5: Guest Operating System

Select the OS you plan to install (e.g. Microsoft Windows Server 2016). This tells vSphere what type of virtual hardware to build — the hardware must be appropriate for the OS, just as a physical server's hardware must support the OS being installed on it.

---

#### Step 6: Customize Hardware

Key settings to configure:

| Setting | Notes |
|---------|-------|
| vCPUs | Number of virtual processors (e.g. 2) |
| Memory | Amount of vRAM (e.g. 4 GB) |
| Virtual disk size | e.g. 20 GB — if the datastore doesn't have enough space the field turns red |
| Disk provisioning | Thin or Thick — see below |
| Network adapter | Choose which port group to connect to (e.g. VM Network) |
| CD/DVD Drive | Set to Datastore ISO file — browse to and select the OS installation ISO |

Select **Connect At Power On** for the CD/DVD drive so the VM boots from the ISO on first power-on.

---

#### Thin vs Thick Provisioned Disk

| Type | Space Consumed | Best For |
|------|---------------|---------|
| Thin Provisioned | Only actual data used | Saving datastore space |
| Thick Provisioned Lazy Zeroed | Full size reserved immediately | General use with reserved space |
| Thick Provisioned Eager Zeroed | Full size reserved + pre-zeroed | High I/O workloads like SQL Server |

Example: A 20 GB thin provisioned disk with only 10 GB of data consumes only 10 GB on the datastore. A 20 GB thick provisioned disk consumes the full 20 GB immediately regardless of how much data is inside it.

---

#### Step 7: Review and Finish

Review all settings on the summary screen and click Finish. The VM is created and appears in the inventory.

---

#### Installing the Guest OS

After creation, power on the VM and open the web console. The VM will boot from the attached ISO file (similar to booting a physical machine from a DVD). Complete the OS installation, then disconnect the CD/DVD drive so the VM doesn't boot from the ISO again on subsequent restarts.

---

> ### Lesson 2: Demo — Installing VMware Tools

#### Why VMware Tools Matters

VMware Tools is a package of drivers and utilities installed inside the guest OS of a VM. It should be installed on every VM possible. While it does improve small things like mouse behavior in the console, its most important benefits are performance-related:

| Component | Benefit |
|-----------|---------|
| VMXNET3 NIC driver | The highest-performance virtual network adapter available — only usable after VMware Tools is installed |
| Memory Control Driver (Balloon Driver) | Enables memory ballooning — allows the ESXi host to reclaim unused memory from VMs |
| Paravirtual SCSI Controller driver | High-performance storage adapter driver |
| Mouse driver | Improved mouse behavior in the VM console |

Without VMware Tools, a VM cannot use the VMXNET3 NIC or support memory ballooning — both significant performance features.

---

#### Checking VMware Tools Status

In the vSphere Client, click the VM → Summary tab. The VMware Tools status shows whether it is installed and running. A newly created VM will show "Not installed."

---

#### Installing VMware Tools

1. Right-click the VM → Guest OS → Install VMware Tools
2. vSphere mounts a VMware Tools ISO image to the VM's CD/DVD drive
3. Open the VM console and log in
4. Browse to This PC — the CD/DVD drive will show the VMware Tools installer
5. Double-click the installer and follow the wizard:
   - **Typical** — installs the standard set of components (sufficient for most VMs)
   - **Complete** — installs everything
   - **Custom** — choose specific components to install
6. Click Next → Install
7. Click Finish when complete → restart the VM when prompted

---

#### After Installation

Once the VM restarts, the Summary tab in the vSphere Client will show VMware Tools as **Running** with the current version.

Also disconnect the CD/DVD drive from the Windows installation ISO (if one was attached from OS installation) before rebooting — otherwise the VM may boot back into the OS installer.

---

> ### Lesson 3: Demo — Upgrading VMware Tools

#### Method 1: Upgrade a Single VM Manually

Right-click the VM → Guest OS → Upgrade VMware Tools

This option is only available (not grayed out) when an update exists. If the VM is already on the latest version, the option will be grayed out.

---

#### Method 2: Auto-Upgrade on Power On

This is a convenient option because the VM already has downtime during a reboot — the upgrade can happen at the same time.

1. Right-click the VM → Edit Settings → VM Options → expand VMware Tools
2. Enable: **Upgrade VMware Tools before each power on**

Every time the VM is powered on, vSphere checks for a newer version and upgrades if one is available.

---

#### Method 3: Bulk Upgrade via Host or Container Object

Available in vSphere 7 and later. This allows upgrading VMware Tools on multiple VMs at once.

Access: vSphere Client → click the ESXi host (or datacenter/folder) → Updates → VMware Tools

Steps:
1. Select the VMs to upgrade (uncheck the VCSA — do not upgrade VMware Tools on vCenter itself)
2. Optionally set auto-update preferences for future upgrades
3. Click **Upgrade To Match Host**
4. Schedule when the upgrade runs:

| VM State | Schedule Option |
|----------|----------------|
| Powered On | Set a time (e.g. after hours) |
| Powered Off | Immediately or scheduled |
| Suspended | Immediately or scheduled |

---

#### Snapshot Rollback Option

Before performing a bulk upgrade, vSphere can automatically take a snapshot of each VM as a safety net:
- Enable automatic pre-upgrade snapshots
- Set an automatic deletion time (e.g. 48 hours) — recommended to avoid leaving snapshots running indefinitely
- Name the snapshots descriptively (e.g. "VMware Tools Update")

Leaving snapshots running long-term causes performance degradation and datastore space consumption — always set a deletion time.

---

#### Scope of Bulk Upgrades

The upgrade can be triggered from different levels of the vSphere inventory:

| Level | Scope |
|-------|-------|
| Individual ESXi host | All VMs on that host |
| Datacenter | All VMs in that datacenter |
| Folder | All VMs in that folder |
| Cluster | All VMs across all hosts in the cluster (requires a cluster to be configured) |

---

> ### Lesson 4: Demo — Inflating a Thin Provisioned Virtual Disk

#### What is Inflation?

Inflating a thin provisioned virtual disk converts it to a thick provisioned eager zeroed disk. This allocates the full disk size on the datastore and writes zeros to all blocks — the same process as creating a thick provisioned eager zeroed disk from scratch.

Use case: A VM was created with a thin provisioned disk to save space, but now needs the performance benefits of a thick provisioned eager zeroed disk (e.g. for a database workload).

---

#### Requirement: VM Must Be Powered Off

Inflation cannot be performed on a running VM. The VM must be powered off first.

A quick way to confirm a VM is powered off: check its files on the datastore. A **vSwap file** only exists when a VM is powered on — if there is no vSwap file, the VM is powered off.

---

#### Steps to Inflate a Thin Provisioned Disk

1. Power off the VM
2. vSphere Client → Storage → select the datastore → Files → find the VM's folder
3. Locate the VMDK file (a thin provisioned disk will show as very small — e.g. 0 KB — if no data has been written)
4. Select the VMDK → click **Inflate**
5. vSphere will allocate the full disk size and write zeros to all blocks
6. Monitor the file size increasing as the inflation progresses
7. Once complete, the VMDK type changes to Thick Provisioned Eager Zeroed

---

#### Verifying the Result

After inflation, right-click the VM → Edit Settings → Hard disk — the disk type will now show as **Thick Provisioned Eager Zeroed** instead of Thin Provision.

---

#### Remove from Inventory vs Delete from Disk

| Action | What Happens |
|--------|-------------|
| Remove from inventory | VM disappears from vCenter but all files remain on the datastore |
| Delete from disk | VM and all its files (VMDK, VMX, etc.) are permanently deleted from the datastore |

---

> ### Lesson 5: Demo — Creating a vApp

#### What is a vApp?

A vApp is a container object in vSphere that groups related virtual machines together. It is similar to a resource pool in that it can have CPU and memory shares, reservations, and limits applied to all VMs within it. The key additional feature of a vApp — beyond what a resource pool offers — is the ability to control **VM boot order and shutdown order**.

Use case: A three-tier application where a Database VM must start before an App Server, which must start before a Web Server.

---

#### Creating a vApp

Access: vSphere Client → Hosts and Clusters → right-click the ESXi host or cluster → New vApp

Steps:
1. Choose to create a new vApp (or clone an existing one)
2. Name the vApp (e.g. RickDemovApp) and choose the datacenter to place it in
3. Configure resource allocations if needed (CPU shares, memory reservations/limits) — leave at defaults if not needed
4. Click Finish

Once created, drag and drop VMs into the vApp from the inventory.

---

#### Configuring Boot Order

Right-click the vApp → Edit Settings → Start Order

VMs are organized into numbered groups. Group 1 starts first, then Group 2, then Group 3, and so on.

| Group | VM | Starts |
|-------|-----|--------|
| Group 1 | Database server | First |
| Group 2 | App server | After Group 1 delay |
| Group 3 | Web server | After Group 2 delay |

For each group, configure when the next group should start:

| Trigger Option | Description |
|---------------|-------------|
| After a delay (seconds) | Wait a fixed number of seconds before starting the next group |
| When VMware Tools are ready | Wait until VMware Tools reports the OS is fully up — requires VMware Tools to be installed |

**Important:** If using "VMware Tools ready" as the trigger, VMware Tools must be installed on the VMs in that group — otherwise the vApp will stall and never move to the next group.

---

#### Configuring Shutdown Order

In the same Start Order settings, each group also has shutdown actions:

| Option | Description |
|--------|-------------|
| Guest shutdown | Gracefully shuts down the OS — requires VMware Tools |
| Power off | Immediately powers off the VM |

A shutdown delay can also be configured between groups — allowing the Web Server to fully shut down before shutting down the App Server, and so on in reverse order.

---

#### Powering a vApp On and Off

- Right-click the vApp → Power → Power On — starts all VMs in the configured group order with the configured delays
- Right-click the vApp → Power → Power Off — stops all VMs according to the shutdown configuration

---

#### Deleting a vApp

| Action | What Happens |
|--------|-------------|
| Delete from disk | Deletes the vApp and permanently destroys all VMs and files inside it |
| Remove from inventory | Removes the vApp from vCenter — VMs and files remain on the datastore |

Deleting a vApp from disk is a convenient way to clean up an entire multi-VM application in one action.

---

> ### Lesson 6: Demo — VM Templates and Customization Specifications

#### What is a VM Template?

A VM Template is a master image of a configured virtual machine used to rapidly deploy multiple identical VMs. Once a VM has been set up exactly as desired — OS installed, updated, hardened, and configured — it can be converted or cloned into a Template. New VMs can then be created from that Template.

Templates cannot be powered on — they are not VMs. They exist solely as a source for cloning.

---

#### Creating a Template

There are two methods:

| Method | How | Result |
|--------|-----|--------|
| Convert to Template | Right-click VM → Template → Convert to Template | Original VM becomes a Template — very fast, no cloning |
| Clone to Template | Right-click VM → Clone → Clone to Template | A copy is made as a Template — original VM is preserved |

Cloning to a Template preserves the original VM but takes longer because it performs a full copy. Converting to a Template is nearly instant but the original VM no longer exists as a VM.

---

#### Template File: VMTX vs VMX

| File | Belongs To | Purpose |
|------|-----------|---------|
| VMX | Virtual Machine | Configuration file for a VM |
| VMTX | Template | Configuration file for a Template |

Converting a Template back to a VM simply changes the VMTX file to a VMX file — which is why it happens almost instantly.

---

#### Important: Renaming a Template

Renaming a Template or VM in the vSphere Client only changes the display name in the inventory. The underlying folder and files on the datastore keep their original names — they are not renamed.

---

#### Deploying a VM from a Template

Right-click the Template → New VM from this Template

Options during deployment:
- **Customize the guest OS** — apply a customization specification (recommended in production)
- **Customize the virtual hardware** — change CPU, memory, network adapters, etc. for this specific VM
- **Power on after creation** — automatically power on the new VM when creation completes

---

#### Customization Specifications

When deploying VMs from a Template, every VM will initially be an identical copy — same Windows SID, same computer name, same IP address. A Customization Specification (Customization Spec) fixes this by automatically modifying OS-level settings during deployment.

Access: vSphere Client → Shortcuts → VM Customization Specifications → New

Key settings a Customization Spec can configure:

| Setting | Purpose |
|---------|---------|
| New unique SID | Prevents SID conflicts between VMs |
| Computer name | Use VM name, specify a name, or append a unique numeric value |
| Windows license key | Apply per-VM OS licensing |
| Local administrator password | Set the admin password for the new VM |
| Time zone | Configure the correct time zone |
| Run commands on first boot | Execute scripts or commands after the OS boots |
| Network settings | Configure DHCP or static IP per NIC |
| Domain join | Automatically join the VM to an Active Directory domain |

Using a Customization Spec is critical in production — without it, all VMs cloned from the same Template will have identical SIDs, names, and IP addresses, which causes conflicts.

---

#### Templates in the vSphere Client

Templates are only visible in the **VMs and Templates** view — they do not appear in Hosts and Clusters. This is a common point of confusion.

---

#### Converting a Template Back to a VM

Right-click the Template → Template → Convert to Virtual Machine

This is nearly instant because only the VMTX file is changed to a VMX file. Use this when you need to boot the Template to make updates, then convert it back to a Template when done.

---

> ### Lesson 7: Demo — Cloning a Virtual Machine

#### Clone vs Template

| Method | Purpose |
|--------|---------|
| Template | Standardized golden image — used repeatedly to deploy many VMs consistently |
| Clone | A one-time copy of an existing VM — used for testing, experimentation, or quick duplication |

Cloning is best when you need a single copy of a specific VM as it exists right now — for example, to safely test a risky fix without touching the original.

---

#### How to Clone a VM

Right-click the VM → Clone → Clone to Virtual Machine

Steps:
1. Name the clone (e.g. RickTestFix)
2. Select the folder and datacenter
3. Choose the ESXi host to run the clone on
4. Choose the disk format and target datastore
5. Optionally apply a customization spec (see below)
6. Click Finish

---

#### Customization Spec

When cloning a VM, the new VM will initially be an identical copy — including the same computer name, IP address, and Windows license key. Running two VMs with the same identity on the same network causes conflicts.

A customization spec scrubs out and reassigns these settings:
- Windows computer name
- IP address configuration
- Windows license key
- Domain join settings
- Local administrator password

In production, always apply a customization spec when cloning. In a lab demo it can be skipped, but in a real environment it is essential.

---

#### What Happens After Cloning

Once the clone is created and powered on:
1. Windows runs SYSPREP automatically
2. The unique settings from the customization spec are applied
3. The result is an independent copy of the original VM with its own identity

---

> ### Lesson 8: Demo — Deploying a VM from an OVF Template

#### What is an OVF Template?

An OVF Template is a captured image of a virtual machine — its configuration, virtual hardware, and disk contents — packaged for portability. It allows a pre-built VM to be deployed into any vSphere environment without having to build it from scratch.

Common sources of OVF Templates:
- A vendor provides a URL to download their appliance (e.g. a network appliance or monitoring tool)
- A file downloaded locally and deployed from your workstation

---

#### OVF vs OVA

| Format | Description |
|--------|-------------|
| OVF | A set of multiple files (descriptor, disk image, manifest) |
| OVA | All OVF files combined into a single archive file |

Both are functionally the same — OVA is simply more convenient as a single download.

---

#### Deploying an OVF Template

Access: vSphere Client → Hosts and Clusters → right-click ESXi host or cluster → Deploy OVF Template

Steps:
1. Choose the source — URL or local file
2. Select the destination folder or resource pool
3. Choose the ESXi host to run the VM on
4. Review storage size (note: thick provisioned values shown may look large — can be changed)
5. Accept license agreements if required
6. Choose the deployment size/form factor (options vary by appliance — e.g. Tiny, Small, Large)
7. Select the network/port group to connect to
8. Fill in any customization parameters required by the appliance
9. Click Finish

vSphere deploys the OVF and creates a new VM. Once complete, power it on and it is ready to use.

---

> ### Lesson 9: VM Snapshots

#### What is a Snapshot?

A snapshot is a point-in-time picture of a virtual machine — capturing the exact state of both memory and the virtual disk at the moment it was taken. It creates a restore point you can revert to if something goes wrong.

**Important:** A snapshot is NOT a backup. It is a temporary restore point only.

---

#### What Happens When You Take a Snapshot

Two things occur simultaneously:

| Component | What Happens |
|-----------|-------------|
| Memory (vmsn file) | The current contents of memory are saved to a .vmsn file |
| Virtual disk (vmdk) | The original VMDK is locked and marked read-only |
| Delta disk (delta.vmdk) | A new delta disk is created — all writes from this point forward go here |

The VM continues running normally. All new data is written to the delta disk instead of the original VMDK.

---

#### Reverting a Snapshot

If something goes wrong and you need to roll back:

1. Memory contents are restored from the vmsn file
2. The delta disk is discarded — all changes since the snapshot are lost
3. The original VMDK is unlocked and returned to its pre-snapshot state

The VM is returned to exactly the state it was in when the snapshot was taken.

---

#### Deleting a Snapshot

Deleting a snapshot means you are committing to keeping the changes — you no longer need the restore point.

What happens during deletion:

1. The vmsn file (memory contents) is purged — no longer needed
2. All data in the delta disk is committed (merged) back into the original VMDK
3. The delta disk is deleted
4. The original VMDK is unlocked

**Critical warnings about snapshot deletion:**

- The commit process can take hours or even days if the delta disk is large
- The VM cannot be powered off or rebooted until the deletion completes
- Snapshot deletion cannot be cancelled once started
- During deletion, the VM temporarily consumes MORE space — not less

---

#### The Space Ballooning Problem

This is an important exam and real-world concept. During snapshot deletion, data must move from the delta disk into the original VMDK simultaneously. This means both files exist at the same time during the merge:

Example:
- Original VMDK: 100 GB
- Delta disk (3 months of changes): 100 GB
- Total before deletion: 200 GB
- **During deletion: up to 300 GB temporarily consumed**

If the datastore is already nearly full, deleting a large old snapshot can push it over the edge and fill it completely. Always check available datastore space before deleting old snapshots.

---

#### Snapshot Best Practices

| Rule | Reason |
|------|--------|
| Never keep a snapshot longer than a week | Delta disks grow continuously — large deltas cause performance issues and long deletion times |
| Never treat a snapshot as a backup | Snapshots are temporary restore points — use proper backup software for long-term protection |
| Delete snapshots as soon as changes are confirmed successful | The sooner you delete, the smaller the delta disk and the faster the commit |
| Check datastore free space before deleting old snapshots | Deletion temporarily increases space consumption before freeing it |

---

> ### Lesson 10: Demo — Working with Snapshots in vSphere

In this demo, snapshots are created, managed, and reverted on a running virtual machine in the vSphere Client. The goal is to observe the full snapshot workflow — from taking a snapshot and making changes, to reverting and examining the underlying files that are created and modified throughout the process.

---

#### Taking a Snapshot

To take a snapshot of a VM, right-click the VM → Snapshots → Take Snapshot.

| Option | Description |
|--------|-------------|
| Name | A label to identify the snapshot (e.g. RickSnap1) |
| Include VM memory | Captures the contents of RAM — allows reverting to an exact running state including open applications |

Without memory included, reverting restores the disk state but the VM will be powered off on revert.

---

#### Snapshot Manager

Once a snapshot is taken, it can be managed from: right-click the VM → Snapshots → Manage Snapshots.

This view shows all snapshots for the VM and marks the current state with **"You are here."** Any changes made after a snapshot was taken are not part of that snapshot — only what existed at the moment it was taken.

---

#### Reverting to a Snapshot

To revert, select the snapshot in Snapshot Manager → Revert.

All changes made after the snapshot was taken are discarded. If memory was included in the snapshot, the VM powers back on in the exact state it was in — open applications, console windows, and all.

---

#### Underlying Files

After taking a snapshot, browsing the VM's folder on the datastore reveals:

| File | Description |
|------|-------------|
| Original VMDK | Locked and preserved exactly as it was at snapshot time |
| Delta disk (e.g. -000003.vmdk) | Captures all new writes made after the snapshot |
| VMSD file | The snapshot database — lists all snapshots for the VM |
| vSwap file | Only present when the VM is powered on — useful for confirming VM state |

---

#### What Happens When Reverting Multiple Times

Each time a revert is performed, the current delta disk is discarded and a new one is created. The old delta disk is purged and replaced — only the most recent delta disk is active at any given time.

---

#### Deleting a Snapshot

To delete a snapshot, select it in Snapshot Manager → Delete.

All changes in the delta disk are committed back into the original VMDK and the delta disk is removed. The VM continues running normally — the only thing lost is the ability to revert to that point.

---

> ### Lesson 11: Demo — Managing VM Templates in a Content Library

#### Converting a VM to a Template

Right-click the VM → Template → Convert to Template

The VM is removed from the VMs list and becomes a Template object under VMs and Templates. Templates cannot be powered on — they exist solely as a source image for deploying new VMs.

---

#### Creating a Content Library

Shortcuts → Content Libraries → New Content Library

| Setting | Notes |
|---------|-------|
| Name | A descriptive name (e.g. TrainerTestsDemo) |
| vCenter association | The vCenter Server this library belongs to |
| Publishing | Allows other vCenter Servers to subscribe and access the library contents |
| Authentication | Requires subscribing vCenter Servers to provide a password |
| Datastore | Where all Templates, OVAs, OVFs, and ISO images will be stored |

A Content Library can store ISO images, OVF/OVA Templates, and VM Templates.

---

#### Cloning a Template into a Content Library

Right-click the Template → Clone to Library

A copy of the Template is uploaded into the selected Content Library. A descriptive name should be given (e.g. Server2016GoldenImage). After cloning, the Template appears under **OVF and OVA Templates** inside the library.

---

#### Deploying a VM from a Content Library Template

Content Library → select Template → Actions → Deploy

| Step | Detail |
|------|--------|
| Name and location | Name the VM and select the destination datacenter |
| Customization spec | Optionally scrub IP address, computer name, and SID |
| Compute resource | Select the ESXi host |
| Disk format | Thin Provisioned recommended to conserve space |
| Network | Select the port group |

---

#### Uploading ISO Images

Content Library → Other Types → Actions → Import Item → select a local file

ISO images can be stored in a Content Library for standardized access across all vCenter Servers in the environment.

---

#### VM Templates vs OVF/OVA Templates

| Method | Appears Under | Supports Check Out/In |
|--------|--------------|----------------------|
| Clone to Library (from Template) | OVF and OVA Templates | No |
| Clone as Template to Library (from VM) | VM Templates | Yes |

To use the Check Out/Check In workflow, the Template must be created by right-clicking a powered-off VM → Clone → Clone as Template to Library.

---

#### Check Out / Check In Workflow

**Check Out**
VM Templates → select Template → Actions → Check Out VM From This Template

A temporary VM is created and appears in VMs and Templates with a blue square indicator. The Template is locked while checked out.

**Making Changes**
Power on the temporary VM, make the required changes (e.g. Windows updates, software installs), then shut it down.

**Check In**
Content Library → VM Templates → select Template → Check In

Changes from the temporary VM are applied to the Template, creating a new version. Previous versions remain available and can be deleted from disk when no longer needed.

---

> ### Lesson 12: VM Configuration Maximums

Virtual machine configuration maximums for vSphere 8 are published at configmax.vmware.com. The following values are the most exam-relevant and worth memorizing.

---

#### Virtual Machine Maximums — Exam-Relevant Values

| Resource | Maximum |
|----------|---------|
| Virtual CPUs (vCPUs) per VM | 768 |
| Memory per VM | 24 TB |
| Virtual NICs (vNICs) per VM | 10 |
| Virtual disk size | 62 TB |

For a complete list of all vSphere 8 configuration maximums, refer to configmax.vmware.com.

---
## Section 7: Using vMotion to Migrate vSphere 8 VMs

This section covers vMotion — VMware's technology for migrating running virtual machines between ESXi hosts without any service interruption. Topics include how vMotion works, its prerequisites, compatibility requirements, and the different types of vMotion available in vSphere 8.

---

> ### Lesson 1: Introduction to vMotion

#### What is vMotion?

vMotion allows a running virtual machine to be migrated from one ESXi host to another with no downtime and no service interruption. The VM remains live throughout the entire process. This is the foundation that features like DRS rely on, and it is also used to manually evacuate hosts before performing maintenance.

---

#### Prerequisites

Three conditions must be met for a standard vMotion to work:

| Requirement | Detail |
|-------------|--------|
| Shared Storage | The VM's files must reside on a datastore accessible by both the source and destination hosts |
| Network Compatibility | The destination host must have a port group on the same VLAN so the VM's IP address still works after migration |
| vMotion VMkernel Port | Each host must have a VMkernel port tagged for vMotion traffic |

---

#### How vMotion Works

1. A copy of the running VM is built on the destination host over the vMotion VMkernel network
2. While the copy is being built, the source VM continues running — any memory changes are tracked in a **memory bitmap**
3. Once the copy is ready, the source VM is briefly halted
4. The memory bitmap is transferred to the destination host
5. The destination VM takes over and the source instance is removed

The brief halt in step 3 is why a single ping may timeout or respond slowly during vMotion — this is imperceptible to end users.

---

#### Compatibility Requirements

| Requirement | Detail |
|-------------|--------|
| CPU compatibility | Source and destination hosts must use the same CPU vendor — Intel to Intel, AMD to AMD |
| No local-only resources | A VM with a locally attached ISO image cannot be vMotioned — the destination host cannot access that file |
| Concurrent vMotion limits | 1 GbE: max 4 concurrent vMotions — 10 GbE: max 8 concurrent vMotions |

---

#### Types of vMotion

| Type | Description |
|------|-------------|
| Standard vMotion | Migrates a running VM between hosts in the same vCenter — requires shared storage |
| Cross-vCenter vMotion | Migrates a VM to a host on a different vCenter instance — both vCenters must be time-synced and in the same SSO domain |
| Long-Distance vMotion | Supports migration over networks with up to 150ms of latency |
| Cross-vSwitch vMotion | Migrates a VM from one virtual switch to another — port group names do not need to match but VLAN must be compatible |
| Storage vMotion | Migrates a running VM's storage from one datastore to another |
| Shared-Nothing vMotion | Migrates both the VM and its storage simultaneously — no shared storage required |

---

> ### Lesson 2: Demo — Configuring vMotion in vSphere

Before vMotion can be performed, two critical requirements must be configured: shared storage accessible by both hosts, and a VMkernel port tagged for vMotion on each host. This lesson walks through verifying and setting up both.

---

#### Step 1: Verify Shared Storage

Navigate to each ESXi host and check which datastores are available under the Datastores tab. Local datastores are accessible only to one host and cannot support vMotion. Shared datastores — such as iSCSI or NFS volumes — appear on both hosts and are what vMotion requires.

The VM being migrated must reside on a shared datastore so both the source and destination hosts can access its files throughout and after the migration.

---

#### Step 2: Configure a vMotion VMkernel Port

Each host needs at least one VMkernel port with the vMotion service enabled.

Navigate to the ESXi host → Configure → VMkernel Adapters → select the VMkernel port (e.g. vmk0) → Edit → enable the vMotion service → OK.

Repeat this on the destination host. Both VMkernel ports must be on the same subnet so they can communicate with each other during migration. A separate dedicated VMkernel port for vMotion can also be created if traffic isolation is needed.

---

#### VM Settings That Affect vMotion

Certain VM configurations can prevent a vMotion from completing:

| Setting | Impact |
|---------|--------|
| CPU Scheduling Affinity | Pins the VM to a specific physical CPU — makes vMotion impossible |
| Memory Reservation | The destination host must have enough free memory to satisfy the reservation |
| CD/DVD connected to a local ISO | The ISO exists only on the source host — the destination cannot access it |
| Virtual hardware version | Must be compatible with the ESXi version on the destination host |

---

#### Port Group Requirements

The VM's network connection must be valid on the destination host. The destination host must have a port group on the same VLAN and physical network as the source. Starting from vSphere 6, port group names no longer need to match — a manual network mapping can be configured during the migration wizard to redirect the VM to a differently named port group.

---

> ### Lesson 3: Demo — Performing a vMotion Migration in vSphere

This lesson walks through performing a live vMotion migration between two ESXi hosts, and demonstrates common configuration issues that can prevent a migration from succeeding.

---

#### Performing a Standard vMotion

To migrate a running VM using vMotion:

1. Right-click the VM → **Migrate**
2. Select **Change compute resource only** (standard vMotion — compute only, no storage move)
3. Choose the destination ESXi host
4. Verify that **compatibility checks succeed**
5. Select the destination network (port group mapping)
6. Choose the migration priority
7. Click **Finish**

The migration typically completes in seconds for most VMs.

---

#### Selecting Migration Priority

When initiating a vMotion, you are prompted to select a scheduling priority:

| Priority | When to Use |
|----------|-------------|
| **High Priority** | Use when migration speed is critical — vMotion gets more CPU resources |
| **Normal Priority** | Use when the source host is CPU-constrained — running VMs keep priority over vMotion |

---

#### Network Compatibility Requirement

Before the migration proceeds, vSphere checks that the VM's port group exists on the destination host. If the port group name does not match, a **compatibility error** will appear.

> **Example:** A VM connected to a port group called `VM Network` on Host A will fail compatibility if the equivalent port group on Host B is named `VM Network 2`. Renaming the port group on the destination host to match resolves the error.

This is one of the inherent limitations of **Standard Virtual Switches** — port groups must be manually maintained to remain consistent across all hosts. Distributed Virtual Switches eliminate this problem by managing port group configuration centrally.

---

#### Common vMotion Compatibility Issues

The table below summarizes the most frequent causes of vMotion failures and how to resolve them:

| Problem | Cause | Resolution |
|---------|-------|------------|
| Port group mismatch | Destination host has a differently named port group | Rename the port group to match, or remap during the migration wizard |
| CD/DVD connected to a local ISO | The ISO file exists only on the source host's local datastore | Disconnect the CD/DVD drive or mount an ISO from a shared datastore |
| Inconsistent VLAN configuration | VM is connected to a port group on a VLAN that does not exist on the destination host | Ensure matching VLAN configurations exist on both hosts |
| CPU affinity set | VM is pinned to a specific physical CPU | Remove the CPU affinity setting |

---

#### The Risk of Migrating to an Incompatible Network

Even when vSphere allows a migration with a network remapping, moving a VM to a port group on a **different VLAN** can cause network isolation. The VM retains its original IP address, which may not route correctly through the new VLAN's gateway — effectively taking the VM offline after migration. Always verify that the destination port group uses the same VLAN before proceeding.

---

#### Best Practices for a vMotion-Ready Environment

Building a consistent environment across all ESXi hosts is the foundation of reliable vMotion operations. The actual migration takes seconds — but the configuration work that makes it possible requires planning up front.

- **Consistent port groups** — all hosts should have the same port groups with identical names and VLAN tags
- **Consistent datastores** — all hosts should have access to the same shared datastores
- **No local-only resources** — avoid mounting ISO images from local datastores; use shared storage instead
- **No CPU affinity** — avoid pinning VMs to specific physical CPUs unless absolutely required
- **Design with vMotion in mind** — these requirements become critical when clusters running DRS are introduced, where vMotion occurs automatically and continuously

> **Note:** When vMotion runs automatically via DRS, any configuration inconsistency across hosts will quickly surface as a migration failure. It is far easier to build the environment correctly from the start than to troubleshoot failures later.

---

> ### Lesson 4: Demo — Configure a Scheduled vMotion in vSphere

A Scheduled vMotion allows you to plan a VM migration in advance — useful for maintenance windows or off-hours migrations where you want to minimize risk and manual involvement.

---

#### Setting Up a Scheduled Task

Navigate to the VM → **Configure** tab → **Scheduled Tasks** → **New Scheduled Task** → select **Migrate**.

From here you can configure:

| Option | Description |
|--------|-------------|
| Task Name | Pre-populated by default — can be customized |
| Frequency | Once, hourly, daily, weekly — choose based on need |
| Date & Time | Set the exact time for the migration to occur |
| Email Notification | If email is configured in vCenter, you can receive a completion notification |

---

#### Completing the Migration Wizard

After setting the schedule, the standard vMotion wizard continues as normal:

1. Select migration type — **Change compute resource only** for a standard vMotion
2. Choose the destination ESXi host
3. Configure network mapping if needed
4. Set migration priority (High or Normal)
5. Click **Finish** — the task is saved and will execute at the scheduled time

---

#### Running a Scheduled Task Immediately

If needed, a scheduled task can be triggered right away without waiting for the scheduled time — click the **Run** button next to the task. This is useful for testing or when plans change.

---

#### Verifying Completion

After the scheduled time passes, the task entry updates with:
- **Status** — Completed
- **Start time**
- **Duration**
- **End time**

A completion record remains visible under the VM's Scheduled Tasks so you can confirm the migration succeeded.

---

#### When to Use Scheduled vMotion

- **Maintenance windows** — migrate VMs off a host during planned downtime without being present
- **Off-hours migrations** — schedule sensitive migrations overnight, then verify in the morning
- **Risk reduction** — gives confidence that the migration will happen at a controlled, low-traffic time

---

> ### Lesson 5: Storage vMotion Concepts

Storage vMotion migrates a running VM's files from one datastore to another — no shutdown required and no service interruption.

---

#### Use Cases

- A datastore is running low on free space
- A datastore is experiencing high latency or performance issues
- A storage array needs to be taken offline for maintenance
- Manual storage load balancing across multiple datastores

---

#### How Storage vMotion Works

Unlike standard vMotion (which moves compute), Storage vMotion moves the VM's files — its `.vmx`, `.vmdk`, `.vswp`, and other associated files — from the source datastore to the destination datastore.

Both datastores must be accessible to the same ESXi host. The VM continues running throughout the entire process.

```
ESXi Host 1
│
├── Datastore 1 (source)   ──→   Datastore 2 (destination)
│       └── VM files                    └── VM files (copy in progress)
│
└── VM (running the whole time)
```

---

#### Mirroring During Migration

Because the VM keeps running while its files are being copied, data can change mid-migration. Storage vMotion handles this using **mirroring**:

- Any writes made to the VM's files during the migration are written to **both** the source and destination datastores simultaneously
- This keeps both copies in sync throughout the process
- When the migration completes, no additional data needs to be transferred — the destination is already up to date

> **Note:** Storage vMotion can be time-consuming. Large `.vmdk` files mean a large amount of data must be copied, and the process can take hours depending on datastore size and I/O throughput.

---

#### Shared-Nothing vMotion

A **Shared-Nothing vMotion** is used when no shared storage exists between two ESXi hosts — for example, when both hosts use only local datastores.

Since standard vMotion requires shared storage, Shared-Nothing vMotion solves this by running **vMotion and Storage vMotion simultaneously**:

| Operation | What Happens |
|-----------|-------------|
| vMotion | Copies the running VM state (memory) to the destination host |
| Storage vMotion | Copies the VM's files to the destination host's local datastore |

Both operations run at the same time. Once complete, the VM cuts over to the destination host and its local datastore in a single step.

```
ESXi Host 1 (local DS)     ──→     ESXi Host 2 (local DS)
  VM running                          VM copy (memory)
  VM files                            VM files (copy)
        ↘ cutover when both copies are ready ↗
```

This allows VMs to be migrated between hosts that have no shared storage — something a standard vMotion cannot do alone.

---

> ### Lesson 6: Demo — Performing a Storage vMotion in vSphere

---

#### Steps to Perform a Storage vMotion

1. Right-click the VM → **Migrate**
2. Select **Change storage only**
3. Click **Next**
4. Optionally change the virtual disk provisioning format (Thin ↔ Thick)
5. Select the destination datastore
6. Click **Next** → **Finish**

The VM remains running throughout. Progress is visible in Recent Tasks.

---

#### Changing Disk Format During Migration

Storage vMotion provides an opportunity to change the disk provisioning format at no extra cost:

| Format | Description |
|--------|-------------|
| **Thin Provisioned** | Disk grows on demand — saves space initially |
| **Thick Provisioned** | Full disk space is allocated upfront |

Simply select the desired format in the migration wizard before confirming the destination datastore.

---

#### Migration Duration

The time a Storage vMotion takes depends entirely on the size of the VM's files — primarily the `.vmdk` virtual disk:

| File | Notes |
|------|-------|
| `.vmdk` | Usually the largest file — can be hundreds of GB |
| `.vswp` | Swap file — equal in size to the VM's memory allocation |
| `.vmx` | VM configuration file — very small |

A VM with a 500 GB virtual disk requires all 500 GB to be transferred between datastores. Depending on the speed of the source and destination storage, this can take hours.

---

#### Scheduling a Storage vMotion

For large VMs, it is strongly recommended to schedule the Storage vMotion as a **Scheduled Task** rather than running it immediately:

VM → **Configure** → **Scheduled Tasks** → **New Scheduled Task** → **Migrate** → select **Storage only**

Set the task to run during a **maintenance window or off-hours** to minimize the impact on production workloads. Storage vMotion generates significant storage I/O and should not compete with active workloads during business hours.

---

> ### Lesson 7: Demo — Shared-Nothing vMotion in vSphere

A Shared-Nothing vMotion migrates a running VM to a different host **and** a different datastore simultaneously — even when no shared storage exists between the two hosts.

---

#### Setting Up the Scenario

To demonstrate a true Shared-Nothing vMotion, the VM's files are first moved to a **local datastore** on the source host using a Storage vMotion. This ensures the destination host has no access to the VM's storage — which is exactly the condition that requires a Shared-Nothing vMotion.

Additionally, a new port group (`Not Shared`, VLAN 25) is created on the destination host only — making the network also non-shared across the two hosts.

This means the migration must move:
- The VM's **compute** (from one host to another)
- The VM's **storage** (from a local datastore to the destination)
- The VM's **network** (to a port group that only exists on the destination)

---

#### Performing a Shared-Nothing vMotion

1. Right-click the VM → **Migrate**
2. Select **Change both compute resource and storage**
3. Choose the destination ESXi host
4. Select the destination datastore
5. Optionally keep or change the disk format
6. Map the source network to the destination port group
7. Set migration priority → **Finish**

> **Important:** The word "vMotion" always implies the VM is **powered on**. Migrating a powered-off VM is possible but is simply called a migration — not a vMotion.

---

#### What Happens During the Migration

Shared-Nothing vMotion runs two operations in parallel:

| Operation | What It Does |
|-----------|-------------|
| vMotion | Copies the running VM's memory state to the destination host |
| Storage vMotion | Copies all VM files to the destination datastore simultaneously |

Once both copies are complete, the VM cuts over to the destination host and datastore in a single step — all while remaining powered on.

---

#### Resource Considerations

Just like a standard Storage vMotion, a Shared-Nothing vMotion is resource-intensive:

- All VM files — including the `.vmdk` — must be transferred to the destination datastore
- The larger the virtual disk, the longer the operation takes
- For large VMs, using a **Scheduled Task** to run the migration during off-peak hours is strongly recommended

---

> ### Lesson 8: vMotion Configuration Maximums

Each ESXi host has **16 units** available for migration operations. Different migration types consume a different number of units, which determines how many can run simultaneously.

---

#### Units per Migration Type

| Migration Type | Units Consumed | Max Simultaneous per Host |
|----------------|---------------|--------------------------|
| vMotion | 2 units | **8 concurrent vMotions** |
| Storage vMotion | 8 units | **2 concurrent Storage vMotions** |

---

#### Limits per Datastore

| Migration Type | Max Simultaneous per Datastore |
|----------------|-------------------------------|
| vMotion | 128 |
| Storage vMotion | 8 |

---

#### Limits by Network Interface Speed

| NIC Speed | Max Concurrent vMotions |
|-----------|------------------------|
| 1 GbE | 4 |
| 10 GbE | 8 |
| 25 GbE | 8 |

---

> **Exam Tip:** Memorize the per-host, per-datastore, and per-NIC limits. These are the most likely values to appear on the certification exam.

---

> ### Lesson 9: vMotion Enhancements in vSphere 8

> **Note:** These enhancements are unlikely to appear on the certification exam but are worth knowing as improvements introduced in vSphere 8.

---

#### Enhancement 1: Application Notification Before Migration

vSphere 8 introduces the ability to **notify applications** that a vMotion is about to occur. This allows the application to prepare itself before the migration begins — for example:

- Stopping certain background services
- Quiescing a database
- Flushing in-memory data to disk

This improves compatibility with applications that historically had issues with vMotion, resulting in smoother and more reliable live migrations.

---

#### Enhancement 2: Unified Data Transport for Cold Migrations

A **cold migration** moves a powered-off VM from one location to another. Previously, cold migrations used the **Network File Copy (NFC)** protocol, which is significantly slower than the vMotion protocol.

In vSphere 8, a new protocol called **Unified Data Transport (UDT)** replaces NFC for cold migrations. UDT combines the strengths of both NFC and vMotion, delivering faster and more efficient cold migration performance.

| Protocol | Used For | Performance |
|----------|----------|-------------|
| NFC (Network File Copy) | Cold migrations (pre-vSphere 8) | Slower |
| Unified Data Transport (UDT) | Cold migrations (vSphere 8+) | Faster |

---
## Section 8: Administer Availability in vSphere 8

This section covers the availability features in vSphere 8, focused on three primary areas:

- **vSphere High Availability (HA)** — protects virtual machines from host failures by automatically restarting them on surviving hosts
- **vCenter High Availability (vCHA)** — makes the vCenter Server itself highly available, along with manual and scheduled backup options
- **Fault Tolerance (FT)** — provides zero-downtime failover for critical VMs by maintaining a live shadow copy at all times

---

> ### Lesson 1: Basic vSphere High Availability Architecture

#### What is vSphere High Availability (HA)?

vSphere HA protects virtual machines from failures by automatically restarting them on other hosts within the same cluster. It is important to understand what HA is — and what it is not:

| Feature | Downtime? | Uses vMotion? |
|---------|-----------|---------------|
| **vSphere HA** | Yes — typically 5–10 minutes per VM | ❌ Never |
| **Fault Tolerance (FT)** | No — zero downtime | ❌ No |
| **DRS** | No | ✅ Yes |

> **Key Point:** HA never uses vMotion. HA involves downtime. vMotion does not.

---

#### What HA Protects Against

- **Host failure** — if an ESXi host goes down, all VMs on it are restarted on surviving hosts in the cluster
- **VM-level failure** — individual VMs that crash can be restarted on the same host

---

#### Requirements

- An **ESXi cluster** (HA is a cluster-level feature)
- **vCenter** — required to configure HA (but not required to keep it running once configured)
- **Shared storage** — VM files must reside on a datastore accessible by all hosts in the cluster

---

#### How HA Works

When an ESXi host fails, the VMs running on it go down. However, because the VM files are stored on **shared storage**, any surviving host in the cluster can access those files and boot the VMs up again.

```
ESXi-01 (fails)          ESXi-02 / ESXi-03 / ESXi-04
  └── VM1 (goes down)  →  surviving host grabs VM1 files
                           from shared datastore → restarts VM1
```

---

#### Cluster Roles: Primary and Secondary Hosts

When HA is enabled on a cluster, an election is held to determine which host becomes the **Primary**. All other hosts become **Secondary** hosts.

The election is based on:
- The number of datastores each host has access to
- The host UUID as a tiebreaker

| Role | Responsibility |
|------|---------------|
| Primary | Sends heartbeats to all secondary hosts; monitors the cluster |
| Secondary | Sends heartbeats to the primary; responds to HA events |

> Secondary hosts do **not** send heartbeats to each other — only to the primary.

---

#### Fault Domain Manager (FDM)

When HA is enabled, a module called the **Fault Domain Manager (FDM)** is instantiated on every host in the cluster. FDM is responsible for running HA operations on each host.

vCenter pushes all cluster configuration to each host's FDM when the cluster is created. This means **HA continues to function even if vCenter goes down** — the FDM has everything it needs to operate independently.

---

#### Heartbeats and the Management Network

Heartbeats are sent between hosts over the **management network** (VMkernel port `vmk0` tagged for management traffic). They serve as a "health check" — as long as heartbeats are flowing, a host is considered alive.

---

#### Host Isolation

A **host isolation** occurs when a host loses connectivity to the management network but is otherwise still running. From the primary's perspective, the host looks like it has failed because heartbeats have stopped.

The risk: the primary may incorrectly trigger an HA response and restart VMs that are actually still running — causing an unnecessary outage.

---

#### Redundant Heartbeat Networks

To reduce false positives from host isolation events, best practice is to configure **two redundant heartbeat networks**:

- If a host cannot send heartbeats over either network, it is much more likely that the host has truly failed
- If only one path is down, the primary can still receive heartbeats over the second network and avoid triggering a false HA response

> **Best Practice:** Always configure at least two management network paths in an HA cluster to distinguish between true host failures and simple network isolation events.

---

> ### Lesson 2: HA Datastore Heartbeats and Host Isolation

#### The Problem: False Host Failure Detection

When an ESXi host loses management network connectivity, the primary host stops receiving heartbeats from it. The primary then tries to ping the isolated host — but if the management network is down, the ping also fails. At this point, the primary cannot tell whether the host has truly failed or is simply network-isolated.

Incorrectly treating an isolated-but-running host as failed would trigger unnecessary VM restarts — causing an outage that didn't need to happen.

---

#### Datastore Heartbeats

To solve this problem, HA uses **Datastore Heartbeats** as a secondary detection mechanism. Each host creates and maintains a **lock file** on a set of shared datastores. The rules are simple:

- A host holds an exclusive lock on its own file as long as it is **up and running**
- If a host goes down, it releases the lock — and the file becomes accessible to other hosts

When the primary cannot reach a host via the management network, it checks the lock file on the shared datastore:

| Lock File State | What the Primary Concludes |
|-----------------|---------------------------|
| File is **locked** | Host still has storage access → host is **isolated, not dead** |
| File is **accessible** | Host has released the lock → host is **truly down** |

This gives HA a reliable second source of truth beyond network heartbeats alone.

---

#### How the Isolated Host Detects Its Own Isolation

An isolated host also runs its own check. When it stops receiving heartbeats from the primary, it attempts to ping a configured **isolation address** (typically a default gateway or router). If that ping fails too, the host concludes it is isolated and invokes its configured **isolation response**.

```
Secondary host stops receiving heartbeats
        ↓
Tries to ping isolation address
        ↓
Ping fails → Host confirms: "I am isolated"
        ↓
Invokes host isolation response
```

---

#### Host Isolation Response Options

Once a host confirms it is isolated, it applies one of three configurable responses:

| Response | What Happens |
|----------|-------------|
| **Do Nothing** | VMs keep running on the isolated host — no action taken |
| **Shut Down and Restart VMs** | VMs are gracefully shut down, then restarted on other hosts using files from shared storage |
| **Power Off and Restart VMs** | VMs are hard-powered off (no graceful shutdown), then restarted on other hosts |

---

#### Choosing the Right Isolation Response

The correct response depends on the network architecture:

- If VM traffic shares the **same physical adapters and switches** as management traffic → management network down likely means VMs are also unreachable → **Shut Down or Power Off** is appropriate
- If VMs use **separate physical adapters and switches** → VM traffic may still be working fine despite the management network being down → **Do Nothing** may be the better choice

> **Best Practice:** Design your network with separate physical adapters for management traffic and VM traffic. This gives you more flexibility in choosing a safe isolation response.

---

> ### Lesson 3: HA Failure Scenarios

#### Scenario 1: Secondary (Slave) Host Failure

When a secondary host goes down, the primary detects the failure through a three-step verification process:

| Step | Check | Result |
|------|-------|--------|
| 1 | Heartbeats stop arriving from the host | ❌ No heartbeat |
| 2 | Primary tries to ping the host | ❌ No response |
| 3 | Primary checks the datastore lock file | ❌ File is unlocked (host released it) |

All three checks failing confirms the host is **truly down**. The primary then begins restarting all VMs that were running on the failed host on the remaining cluster hosts.

> **Restart Priority:** If configured, the most critical VMs are restarted first.

---

#### Scenario 2: Primary (Master) Host Failure

When the primary host fails, all secondary hosts stop receiving heartbeats from it. The secondaries then:

1. Begin communicating with each other
2. Hold a new **election** — the host with access to the most datastores wins; UUID is the tiebreaker
3. The newly elected primary takes over and verifies whether the old primary truly failed (ping + datastore heartbeat check)
4. Once confirmed down, all VMs from the failed primary are restarted on surviving hosts

---

#### Scenario 3: vCenter Running as a VM Inside the Cluster

If vCenter is running as a VM on a host that fails, HA handles it like any other VM — it simply restarts on another host in the cluster. Since each host runs its own **Fault Domain Manager (FDM)**, HA continues to operate independently of vCenter.

- vCenter is needed to **configure** HA
- vCenter is **not needed** to keep HA running once it is enabled
- If vCenter fails, HA still operates — but configuration changes cannot be made until vCenter is restored

---

#### Scenario 4: Individual VM Failure (VM Monitoring)

HA can also protect against individual VM crashes — not just host failures. This requires:

- **VMware Tools** installed on the VM
- **VM Monitoring** enabled on the cluster

VMware Tools sends regular heartbeats from the VM to the host's FDM. If the VM crashes and heartbeats stop, HA automatically **reboots the VM on the same host** — no host migration involved.

```
VM crashes → heartbeats stop → FDM detects failure → VM rebooted on same host
```

> ### Lesson 4: Demo — Basic High Availability Configuration in vSphere

> **Important:** This lesson covers **vSphere HA** — High Availability for virtual machines running on ESXi hosts. This is completely separate from **vCenter High Availability (vCHA)**, which is covered later in this section.

---

#### Creating an HA Cluster

A cluster is a container object that groups multiple ESXi hosts together. To create one:

1. Right-click the datacenter or folder → **New Cluster**
2. Give the cluster a name
3. Optionally enable **DRS**, **HA**, or a **Cluster Image** (ensures all hosts run an identical ESXi image)
4. Click **Next** → **Finish**
5. Add ESXi hosts to the cluster by clicking **Add** or dragging and dropping them in

Once the cluster exists, HA is configured under:
**Cluster → Configure → vSphere Availability → Edit**

---

#### HA Configuration Options

**1. Host Monitoring**

Enables the cluster to monitor ESXi hosts for failures and isolation events. Should always be left enabled.

---

**2. Host Failure Response**

Defines what happens to VMs when the host they are running on fails:

| Option | Behavior |
|--------|----------|
| Disabled | No action taken — VMs are not restarted |
| Restart VMs (Default) | VMs are restarted on surviving hosts in the cluster |

---

**3. VM Restart Priority**

When a host fails and its VMs need to restart, priority controls the order:

| Priority | Use Case |
|----------|----------|
| Highest / High | Critical VMs — restart first |
| Medium (default) | Standard VMs — equal priority |
| Low / Lowest | Non-critical VMs — restart last |

The cluster moves to lower-priority VMs after one of the following conditions is met for higher-priority VMs (configurable):

- Resources have been allocated
- VMs are **powered on**
- Guest heartbeats are detected

An additional **delay** (e.g. 60 seconds) and a **timeout** (e.g. 10 minutes) can also be configured before moving on to the next priority tier.

---

**4. Host Isolation Response**

Defines what happens to VMs on a host that is isolated from the management network but is still running:

| Option | Behavior |
|--------|----------|
| Disabled (Default) | VMs are left running on the isolated host |
| Shut Down and Restart | VMs are gracefully shut down and restarted on other hosts |
| Power Off and Restart | VMs are hard-powered off and restarted on other hosts |

The right choice depends on whether VM traffic shares the same network path as management traffic.

---

**5. Permanent Device Loss (PDL) and All Paths Down (APD)**

These cover storage connectivity failures:

| Condition | Description |
|-----------|-------------|
| Permanent Device Loss (PDL) | A physical storage device has failed and sent a sense code confirming it is down |
| All Paths Down (APD) | The ESXi host has lost all connectivity to a storage system |

For both conditions, options include doing nothing, generating events only, or powering off VMs and restarting them on hosts that still have storage access.

---

**6. VM Monitoring**

Monitors individual VMs using VMware Tools heartbeats:

| Setting | Behavior |
|---------|----------|
| Disabled | No action if a VM stops sending heartbeats |
| VM Monitoring | VM is rebooted on the **same host** if heartbeats stop |
| VM and Application Monitoring | Also monitors specific applications — resets the VM if an app stops responding |

> **Note:** VM Monitoring reboots the VM on the same host — it does not migrate it. The host itself is healthy; only the individual VM has failed.

---

> ### Lesson 5: vCenter High Availability (vCHA) for vSphere 8

> **Exam Warning:** Do not confuse **vCenter High Availability (vCHA)** with **vSphere High Availability (HA)**. vSphere HA protects VMs running in a cluster. vCHA protects the vCenter Server Appliance itself.

---

#### How vCenter High Availability Works

vCHA uses three nodes to keep vCenter available:

| Node | Role |
|------|------|
| **Active** | The live vCenter instance — all management traffic connects here |
| **Passive** | A full clone of the Active node — constantly synchronized over the vCHA network — takes over if the Active fails |
| **Witness** | A lightweight clone — acts as a tiebreaker to prevent split-brain scenarios |

The **Active** node replicates its database **synchronously** to the **Passive** node at all times. If the Active fails, the Passive assumes the Active's identity — including its IP address — and takes over immediately.

> **Recovery Time Objective (RTO): 5 minutes** — this is the expected downtime window in the event of a vCenter failover.

---

#### Split-Brain Protection

A split-brain situation occurs when both Active and Passive nodes believe they are the Active vCenter simultaneously. The **Witness** node provides a third vote (quorum) to prevent this, ensuring only one node can hold the Active role at any time.

---

#### Platform Services Controller (PSC) — Historical Context

In **vCenter 6.7 and earlier**, the Platform Services Controller could be deployed in two ways:

| Topology | Description |
|----------|-------------|
| **Embedded PSC** | PSC installed inside the vCenter Server Appliance |
| **External PSC** | PSC deployed as a separate standalone component, shared across multiple vCenter instances |

With an external PSC, the vCHA topology was more complex — the PSC existed outside of the Active/Passive/Witness nodes and had to be managed separately.

---

#### Embedded Platform Services Controller (PSC)

As of vSphere 7 (and continuing in vSphere 8), the PSC is always **embedded** inside the vCenter Server Appliance — there is no longer an option to deploy an external PSC. When vCHA is configured, the PSC is included in the clone, so the Passive node has a fully ready PSC if failover occurs.

Other changes introduced in vCenter 7 that remain in vSphere 8:
- No external PSC deployment option
- No vCenter installer for Windows — VCSA only

---

#### Automatic vs. Manual Configuration

| Method | What It Handles |
|--------|----------------|
| **Automatic** (recommended for most environments) | Adds a second NIC to vCenter, clones Active to create Passive, creates the Witness VM, configures DRS anti-affinity rules |
| **Manual** | Administrator handles all of the above manually |

**When to use manual configuration:**
- vCenter runs in a management cluster in a different SSO domain
- vCHA nodes need to be split across multiple sites

---

> ### Lesson 6: Demo — Deploy vCenter High Availability (vCHA)

#### Prerequisites

Before configuring vCHA, the following must be in place:

- **Three ESXi hosts** — one for the Active node, one for the Passive node, one for the Witness
- **Identical networking** on all three hosts — a Management Network and a dedicated vCHA network (port group)

---

#### Step 1: Configure Networking on All Three Hosts

Create a dedicated port group for vCHA replication traffic on every host:

**Host → Configure → Virtual Switches → Add Networking → Port Group**

- Port group name: `vCHA`
- VLAN: `20` (or any VLAN that works in your environment)

Repeat on all three hosts so the networking configuration is identical across the cluster. This is a mandatory prerequisite regardless of whether automatic or manual configuration is used.

---

#### Step 2: Initiate vCHA Setup

Navigate to the **vCenter Server object** (not a host) → **Configure** tab → **vCenter HA** → **Set Up vCenter HA**

The wizard will ask:
1. Which network to use as the **vCHA network** for the Active node — select the `vCHA` port group
2. Whether to **automatically create clones** for the Passive and Witness nodes — leave this checked for automatic configuration

---

#### Step 3: Configure Passive and Witness Nodes

For each node (Passive and Witness), configure:

| Setting | Passive Node | Witness Node |
|---------|-------------|--------------|
| Folder | Choose a VM folder | Choose a VM folder |
| Compute Resource | Second ESXi host | Third ESXi host |
| Datastore | Different datastore from Active (recommended) | Any available datastore |
| Management Network | Required | Not required |
| vCHA Network | Required | Required |

> **Note:** The Witness node does not need a Management Network — it is never used for vSphere Client access or management tasks.

---

#### Step 4: Assign vCHA Network IP Addresses

All three nodes communicate over the vCHA network using a private IP range. No default gateway is needed since all nodes are on the same port group.

Example configuration:

| Node | IP Address |
|------|-----------|
| Active | 10.1.1.101 |
| Passive | 10.1.1.102 |
| Witness | 10.1.1.103 |

This is an isolated private network used exclusively for replication between the three vCHA nodes.

---

#### Step 5: Finish

Click **Finish**. vCenter will automatically:
- Clone the Active node to create the Passive and Witness VMs
- Configure replication between Active and Passive
- Set up DRS anti-affinity rules

Progress is visible in **Recent Tasks**. Once complete, vCenter High Availability is fully configured.

---

> ### Lesson 7: Demo — vCenter Manual Backup to an SMB Share

vCenter Server Appliance backups are managed through the **VAMI** (vCenter Server Appliance Management Interface), accessible at:

```
https://<vCenter-IP>:5480
```

Log in with **root** credentials.

---

#### Step 1: Create a Shared Folder on the Backup Target

On the Windows server that will receive the backup:

1. Create a new folder (e.g. `vcsabkup`)
2. Right-click → **Share with** → **Specific people**
3. Add the **Administrator** account with **Read/Write** access
4. Click **Share**

---

#### Step 2: Perform a Manual Backup

In the VAMI:

**Backup** → **Backup Now**

Configure the following:

| Setting | Description |
|---------|-------------|
| **Backup Location** | `smb://<Windows-Server-IP>/vcsabkup` |
| **Username / Password** | Credentials for the shared folder (e.g. Administrator) |
| **Encryption Password** | Optional but recommended — encrypts the backup files |
| **Statistics, Events & Tasks** | Optional — include historical vCenter data in the backup |

> The vCenter inventory and configuration are always included automatically — no need to select them separately.

Click **Start** to initiate the backup.

---

#### Backup File Structure

Each backup is stored in its own timestamped folder, making it easy to identify when each backup was taken:

```
vcsabkup/
├── 2023-03-06_10-30-00/   ← backup taken March 6, 2023
├── 2023-03-06_14-15-00/   ← second backup same day
```

As more manual backups are taken, additional timestamped folders are created in the same share.

---

> ### Lesson 8: Demo — vCenter Scheduled Backups and Retention Policies

#### Supported Backup Destinations

The VAMI supports the following protocols for file-based vCenter backups:

`FTP` · `FTPS` · `SFTP` · `HTTP` · `HTTPS` · `NFS` · `SMB`

---

#### Configuring a Scheduled Backup

In the VAMI: **Backup** → **Configure** (next to Backup Schedule)

| Setting | Description |
|---------|-------------|
| **Backup Location** | Same SMB/NFS/FTP path used for manual backups |
| **Credentials** | Username and password with access to the backup destination |
| **Scheduled Time** | Time of day the backup runs — choose an off-hours window |
| **Encryption Password** | Encrypts the backup files |
| **Retention Policy** | Number of scheduled backups to keep — older ones are automatically purged |

---

#### Retention Policy Behavior

The retention policy applies **only to scheduled backups** — manual backups are never affected by it.

| Backup Type | Prefix | Affected by Retention Policy? |
|-------------|--------|-------------------------------|
| Manual | `M` | ❌ No — kept indefinitely |
| Scheduled | `S` | ✅ Yes — purged when limit is exceeded |

For example, setting a retention of **2** means only the two most recent scheduled backups are kept. When a new scheduled backup completes, the oldest scheduled backup is automatically deleted from the share.

---

#### Backup Folder Structure

Both manual and scheduled backups coexist in the same share, each in their own timestamped folder:

```
vcsabkup/
├── M-2023-03-06_10-30-00/   ← manual backup
├── M-2023-03-06_14-15-00/   ← manual backup
└── S-2023-03-07_15-15-00/   ← scheduled backup
```

---

> ### Lesson 9: Zero-Downtime Failover with Fault Tolerance (FT)

#### Fault Tolerance vs. High Availability

| Feature | vSphere HA | Fault Tolerance (FT) |
|---------|-----------|----------------------|
| Scope | Entire cluster | Individual VMs |
| Downtime on failure | Yes — 5–10 min | No — zero downtime |
| Data/connection loss | Possible | None |
| Uses vMotion | No | Yes (for migration) |
| Storage vMotion supported | Yes | ❌ No |
| Snapshots supported | Yes | ❌ No |

---

#### How Fault Tolerance Works

FT keeps a **Secondary VM** running on a separate ESXi host, perfectly synchronized to the **Primary VM** at all times using a mechanism called **checkpointing**. The Secondary VM's files are stored on a separate datastore.

```
Primary VM (ESXi-01)          Secondary VM (ESXi-02)
  └── Files on Datastore 1  ←→  └── Files on Datastore 2
         ↕ FT Logging Network (10 GbE)
```

Any change in the Primary — including mouse movements — is instantly replicated to the Secondary. Both VMs are identical at all times.

---

#### Failover Behavior

When the host running the Primary VM fails:

1. The Secondary VM **immediately becomes the new Primary** — zero downtime
2. A **new Secondary VM** is automatically spawned on another host to re-protect the new Primary

> **Important:** FT does **not** protect against OS-level failures (e.g. blue screen, accidental data deletion, virus). Any changes in the Primary OS — including bad ones — are replicated to the Secondary in real time.

---

#### Prerequisites

- The host cluster must have **vSphere HA enabled** — FT cannot be configured without it
- Each host needs a **VMkernel port tagged for FT Logging traffic**
- The FT Logging network must be **10 GbE** — a dedicated network is strongly recommended
- **Snapshots must be removed** from the VM before enabling FT

---

#### Configuration Limits (vSphere 8 — same as vSphere 7)

| Limit | Default Value |
|-------|--------------|
| Max FT VMs per host | 4 |
| Max FT vCPUs per host | 8 (total across all FT VMs) |
| Max vCPUs per FT VM — Standard / Enterprise | 2 |
| Max vCPUs per FT VM — Enterprise Plus | 8 |

> These limits can be overridden through advanced host configuration settings.

**vCPU limit examples:**
- 1 VM with 8 vCPUs → only that VM can have FT on that host
- 4 VMs with 2 vCPUs each → all four can have FT simultaneously

---

#### Supported Disk Formats

Unlike older versions of vSphere that required Thick Provisioned disks, FT in vSphere 8 supports:

- Thin Provisioned
- Thick Provisioned Eager Zeroed
- Thick Provisioned Lazy Zeroed

---

#### Supported and Unsupported Features

| Feature | Supported with FT? |
|---------|-------------------|
| vMotion | ✅ Yes |
| DRS (automatic migration) | ✅ Yes |
| Snapshots | ❌ No — must be removed before enabling FT |
| Storage vMotion | ❌ No — disable FT first, then perform Storage vMotion |
| Virtual Volumes (vVols) datastores | ❌ No (unless using vSAN) |
| Storage Policy-Based Management | ❌ No (unless using vSAN) |

---

> ### Lesson 10: Demo — Fault Tolerance in vSphere

#### Step 1: Enable HA on the Cluster

FT cannot be enabled without HA. Navigate to the cluster → **Configure** → **vSphere Availability** → enable High Availability.

---

#### Step 2: Enable FT Logging on VMkernel Ports

On every host in the cluster:

**Host → Configure → VMkernel Adapters → Edit → enable Fault Tolerance Logging**

Repeat for each host. A dedicated VMkernel port strictly for FT Logging traffic is recommended in production — FT generates significant network traffic.

---

#### Step 3: Enable Fault Tolerance on a VM

Unlike HA (which is cluster-wide), FT is enabled per individual VM:

Right-click the VM → **Fault Tolerance** → **Turn On Fault Tolerance**

The wizard will prompt for:
- **Datastore for Secondary VM** — must be different from the Primary VM's datastore
- **ESXi Host for Secondary VM** — must be different from the host running the Primary

> **Best Practice:** Place the Secondary VM's files on a completely separate storage array from the Primary to maximize availability.

---

#### Compatibility Checks

Before FT is enabled, vSphere runs compatibility checks. Common issues include:

| Issue | Fix |
|-------|-----|
| VM connected to a network that only exists on one host | Move the VM to a port group available on all hosts |
| CPU hotplug enabled | Disable CPU hotplug on the VM |
| Snapshots present | Remove or commit all snapshots |
| Other VM-specific settings | Review and resolve per the compatibility warning |

Work through each compatibility issue until all checks pass, then click **Finish**.

---

#### What Happens After FT is Enabled

- A Secondary VM is automatically created on the chosen host and datastore
- The Primary and Secondary are kept in perfect sync over the FT Logging network
- VM icons change in the vSphere Client to indicate FT protection:
  - **Primary VM** — solid blue outlined icon
  - **Secondary VM** — faded/transparent icon

---

#### Testing a Host Failure

Simulating a host failure (hard power off — no maintenance mode) on the Primary VM's host:

1. Primary VM goes down with the host
2. Secondary VM **immediately takes over** as the new Primary — zero downtime
3. A new Secondary is spawned on another available host to re-protect the new Primary
4. While no Secondary exists, a **red indicator** appears on the VM warning that FT protection is temporarily lost

When the failed host is restored and comes back online, the Secondary VM is automatically re-instantiated on it and FT protection resumes.

---

## Section 9: Administer Resource Management Features in vSphere 8

This section covers resource management in vSphere 8, focused on two primary areas:

- **Resource Pools** — container objects that group virtual machines together and control how resources are shared among them
- **Distributed Resource Scheduler (DRS)** — automates vMotion across ESXi hosts in a cluster to balance workloads and improve overall VM performance

---

> ### Lesson 1: Resource Pools

#### What is a Resource Pool?

A Resource Pool is a container object that groups virtual machines together and controls how CPU and memory resources are distributed among them. It allows resource management to be applied to a group of VMs rather than configuring each VM individually.

A Resource Pool can contain:
- Virtual machines
- Other Resource Pools (creating a hierarchy)
- vApps

---

#### Use Cases

- **Isolate critical workloads** — give a production Resource Pool higher shares or reservations to guarantee resources for important VMs
- **Cap non-essential workloads** — apply limits to a development Resource Pool so it cannot consume more than a set amount of resources
- **Centralize permissions and alarms** — any permissions or alarms configured on a Resource Pool automatically apply to all objects inside it

---

#### Shares, Limits, and Reservations

These three mechanisms control how resources are allocated within and across Resource Pools:

| Mechanism | Behavior | Enforced? |
|-----------|----------|-----------|
| **Shares** | Define proportional entitlement to resources relative to other pools or VMs | Only during contention |
| **Limits** | Set a hard ceiling on resource usage — never exceeded, even if resources are available | Always |
| **Reservations** | Guarantee a minimum amount of resources — immediately consumed at boot | Always |

> **Key Point:** Shares are the most flexible option — they only kick in when the host is running low on resources. Limits and reservations are strictly enforced at all times.

---

#### Shares Example

A host has 48 GB of memory. Two Resource Pools are configured:

| Resource Pool | Shares | Memory Entitlement (during contention) |
|---------------|--------|----------------------------------------|
| Development | 1 part | 16 GB (one third) |
| Production | 2 parts | 32 GB (two thirds) |

When the host has plenty of free memory, both pools can use as much as they need. When the host runs low, the share structure is enforced and the Production pool is guaranteed twice as much memory as the Development pool.

---

#### Reservations and Expandable Reservations

When a VM has a memory reservation, it can only boot if its parent Resource Pool has enough reserved memory to satisfy it. If the pool cannot satisfy the reservation, the VM will not start.

**Expandable Reservations** solve this by allowing a VM to look beyond its immediate parent pool:

- If the Resource Pool cannot satisfy the reservation, the VM checks the next level up (parent pool or the root — the ESXi host itself)
- If that level can satisfy the reservation, the VM is allowed to boot

This provides flexibility without requiring the Resource Pool itself to hold a large reservation at all times.

---

#### Child Resource Pools

A Resource Pool can contain other Resource Pools, forming a hierarchy. A child pool competes for resources from its parent pool, just as VMs compete for resources within a pool. Any resources assigned to a child pool come out of the parent pool's allocation.

---

#### Root Resource Pool

The Root Resource Pool is the ESXi host (or DRS cluster) itself — it is the top-level resource boundary that all child pools and VMs ultimately draw from.

---

#### vApp

A vApp is essentially a Resource Pool with one additional capability — **power-on order control**:

| Feature | Resource Pool | vApp |
|---------|--------------|------|
| Shares, Limits, Reservations | ✅ | ✅ |
| Container for VMs and pools | ✅ | ✅ |
| Permissions and alarms | ✅ | ✅ |
| Control power-on/off order | ❌ | ✅ |

A vApp is typically used when VMs have a dependency on each other — for example, ensuring a database server always starts before an application server, which starts before a web server.

---

> ### Lesson 2: Distributed Resource Scheduler (DRS)

#### What is DRS?

DRS is automated vMotion for the purposes of load balancing. It continuously monitors resource usage across all hosts in a cluster and automatically migrates VMs from one host to another to keep workloads balanced.

> **Core concept:** DRS = automated vMotion. It works only as well as vMotion works within the cluster.

---

#### Host Cluster Features

A cluster is a logical grouping of ESXi hosts that unlocks features unavailable on standalone hosts:

| Feature | What It Does |
|---------|-------------|
| **High Availability (HA)** | Automatically restarts VMs on surviving hosts if a host fails |
| **DRS** | Automatically migrates VMs between hosts for load balancing |
| **vSAN** | Pools local host storage into a shared datastore across the cluster |

---

#### How DRS Works

DRS monitors the resource utilization of all hosts in the cluster. When imbalances are detected, it uses vMotion to migrate VMs to less loaded hosts — all automatically, without manual intervention.

Because DRS relies entirely on vMotion, anything that breaks vMotion also breaks DRS.

---

#### Common DRS / vMotion Blockers

| Issue | Impact |
|-------|--------|
| VM connected to a local ISO image | Destination host cannot access the file — vMotion blocked |
| Incompatible CPU vendors (Intel vs. AMD) | CPUs are not compatible across hosts — vMotion blocked |
| CPU affinity configured on a VM | VM is pinned to a specific physical CPU — vMotion blocked |
| 1 GbE network between hosts | vMotion limited to 4 simultaneous migrations |
| 10 GbE network between hosts | vMotion supports up to 8 simultaneous migrations |

---

#### Best Practices for a DRS-Ready Cluster

- Ensure all hosts use the same CPU vendor (Intel or AMD — not mixed)
- Avoid connecting VMs to local ISO images — use shared datastores instead
- Avoid CPU affinity settings on VMs unless absolutely necessary
- Use at least a 10 GbE network between hosts for better concurrent migration throughput
- Configure consistent port groups and datastores across all hosts in the cluster

---

> ### Lesson 3: DRS Enhancements in vSphere 7 and 8

#### DRS Before vSphere 7: Cluster-Wide Deviation Model

Prior to vSphere 7, DRS focused on keeping all hosts in a cluster **equally utilized**. Every 5 minutes, DRS would measure host utilization across the cluster and trigger vMotion operations to minimize the deviation between hosts — moving high-resource VMs off busy hosts to less loaded ones.

The goal was simple: keep host utilization as even as possible across the cluster.

---

#### DRS in vSphere 7+: VM-Centric Model

In vSphere 7, the focus shifted from the cluster level down to the individual VM level. Instead of just balancing host utilization, DRS now calculates a **VM DRS Score** for each virtual machine — a value out of 100 representing how efficiently that VM is executing on its current host.

A higher score means the VM is running well. A lower score means it could potentially perform better on a different host.

The score is based on several factors:

| Factor | Description |
|--------|-------------|
| CPU Percent Ready | How often the VM is waiting for CPU time — high values indicate CPU contention |
| CPU Cache | How effectively the VM is using CPU cache |
| Memory Swap Activity | Memory swapping indicates the host is under memory pressure |
| Host Headroom | Whether the host has enough spare resources to handle the VM's expanding workload |
| vMotion Cost | The CPU and network overhead of performing a vMotion — factored into whether migration is worth it |

> **Key Point:** The VM DRS Score is not a health score — it measures **execution efficiency**. A low score means the VM could run better on a different host, and DRS will migrate it if the improvement justifies the cost of the vMotion.

---

#### Cluster DRS Score

The **Cluster DRS Score** is an aggregate of all individual VM DRS scores within the cluster. It gives an at-a-glance view of how well DRS is performing across the entire cluster. This score is visible in the vSphere Client under:

**Cluster → Summary → vSphere DRS**

From there, clicking **View All VMs** shows the individual DRS score for each VM in the cluster.

---

#### What Low DRS Scores Indicate

| Symptom | Likely Cause |
|---------|-------------|
| High CPU Ready values | VMs are waiting for CPU — host is CPU-constrained |
| Memory swapping activity | Host is running low on physical memory |
| Low host headroom | Host cannot accommodate the VM's growing resource needs |

When these conditions are detected, DRS will evaluate whether migrating the VM to another host would improve its score, and perform the vMotion if it would.

---

> ### Lesson 4: DRS Affinity Rules

DRS Affinity Rules allow you to override default DRS behavior for specific VMs — either keeping certain VMs together on the same host or forcing them apart onto different hosts.

---

#### Anti-Affinity Rules — Keeping VMs Apart

**Use case:** Two domain controllers serving the same role. If DRS migrates both onto the same host and that host fails, both go down simultaneously.

An **anti-Affinity Rule** tells DRS to keep specified VMs on different hosts at all times, preventing this scenario.

| Rule Type | Behavior | Can HA Override It? |
|-----------|----------|---------------------|
| **Preferential** | VMs *should* run on different hosts | ✅ Yes — HA can boot both on the same host if no other hosts are available |
| **Mandatory** | VMs *must* run on different hosts | ❌ No — if only one host is available, the second VM will not boot |

---

#### Affinity Rules — Keeping VMs Together

**Use case:** A Reporting server constantly queries a Database server. If they run on different hosts, all query traffic crosses the physical network unnecessarily. Placing them on the same host keeps that traffic internal.

An **Affinity Rule** tells DRS to keep specified VMs on the same host, reducing unnecessary network traffic between tightly coupled VMs.

---

#### Group Affinity Rules — Scaling to Host and VM Groups

For larger environments, individual VM rules may not be enough. DRS supports **VM Groups** and **Host Groups** that allow rules to be applied at scale.

**Example:**
- Create a **Production Hosts** group and a **Development Hosts** group within the same cluster
- Create a **Production VMs** group containing all production virtual machines
- Apply a Group Affinity Rule: *Production VMs should run on Production Hosts*

| Rule Type | Behavior | HA Behavior on Failure |
|-----------|----------|------------------------|
| **Preferential** | Production VMs run on Production Hosts under normal conditions | HA can restart them on Dev Hosts if all Production Hosts fail |
| **Required (Mandatory)** | Production VMs must run on Production Hosts | HA will not restart them on Dev Hosts — VMs remain down |

> **Best Practice:** Use Preferential rules in most cases. Mandatory rules provide stricter control but can result in VMs not recovering during a major failure.

---

#### Where to Configure Affinity Rules in vSphere

Navigate to: **Cluster → Configure tab**

| Section | What to Configure |
|---------|------------------|
| **VM/Host Rules** | Create Affinity or anti-Affinity rules for individual VMs or groups |
| **VM/Host Groups** | Define VM groups and Host groups for use in Group Affinity Rules |

---

> ### Lesson 5: DRS, HA, and vCenter Running as a VM

#### vCenter's Role in DRS

vCenter is required for DRS to function — it monitors cluster performance, determines which VMs need to be migrated, and executes the vMotion operations. Without vCenter, there is no DRS.

---

#### vCenter Running Inside the Cluster

vCenter can run as a VM within the same DRS cluster it manages. This works fine under normal conditions — vCenter controls DRS even while being a member of the cluster, and may itself be migrated by DRS to different hosts over time.

The problem arises when **vCenter fails**. If vCenter goes down, the vSphere Client becomes unavailable. To troubleshoot or reboot vCenter, you must connect directly to an ESXi host using the **Host Client** — but only if you know which host vCenter is currently running on.

---

#### Solutions for Tracking vCenter's Location

**Option 1: VM Override — Manual DRS Mode**

Configure a VM override on the vCenter VM to set its DRS automation level to **Manual**:

- vCenter stays on its current host unless manually approved to move
- If a host needs to enter maintenance mode, DRS generates a recommendation — but the migration only happens after manual approval
- You always know exactly which host vCenter is on

DRS can also be **fully disabled** for a specific VM using a VM override, while leaving DRS enabled for all other VMs in the cluster.

**Option 2: Group Affinity Rule**

Create a Host Group (e.g. ESXi01 and ESXi02) and apply a **Preferential Affinity Rule** specifying that vCenter should run on that group of hosts. This limits vCenter's possible locations to a known subset of hosts without preventing failover.

> Use a **Preferential** (not Mandatory) rule so that if all hosts in the group fail, HA can still restart vCenter on another available host.

---

#### HA Behavior with vCenter as a VM

HA handles vCenter like any other VM. If the host running vCenter fails, HA will automatically restart it on another host — no special configuration required. HA uses the **Fault Domain Manager** on each host and does not depend on vCenter to operate.

---

#### Where to Configure VM Overrides

Navigate to: **Cluster → Configure → VM Overrides**

From here, individual VMs can have their DRS automation level set independently from the rest of the cluster.

---

> ### Lesson 6: DRS and Maintenance Mode

#### What is Maintenance Mode?

Maintenance Mode ensures no VMs are running on an ESXi host, allowing it to be safely patched, updated, or serviced. A host cannot enter Maintenance Mode while VMs are still running on it.

There are two ways to clear a host before maintenance:
- **Power off all VMs** — causes downtime
- **Migrate VMs with DRS** — zero downtime

---

#### DRS + Maintenance Mode Workflow

When a host in a DRS cluster is placed into Maintenance Mode, DRS automatically detects this and uses vMotion to migrate all running VMs off that host to other hosts in the cluster — before the host actually enters Maintenance Mode. The VMs experience no downtime throughout the process.

```
Host enters Maintenance Mode request
        ↓
DRS detects VMs must be evacuated
        ↓
vMotion migrates all VMs to other cluster hosts
        ↓
Host enters Maintenance Mode (no VMs running)
        ↓
Patches/updates applied → host exits Maintenance Mode
        ↓
VMs can migrate back
```

---

#### DRS + Update Manager

Update Manager (built into the vCenter Server Appliance) handles ESXi host patching. When combined with DRS, it can roll through an entire cluster one host at a time with no VM downtime:

1. Hosts are attached to a **patch baseline**
2. Update Manager scans hosts and identifies which patches are needed
3. For each host: DRS evacuates VMs → host enters Maintenance Mode → patches applied → host exits Maintenance Mode
4. Process repeats for the next host

This allows patching to be performed even during business hours, since no VMs are ever powered off.

> **Key Benefit:** DRS + Update Manager together enable zero-downtime rolling maintenance across an entire cluster.

---

> ### Lesson 6: DRS and Maintenance Mode

#### What is Maintenance Mode?

Maintenance Mode ensures no VMs are running on an ESXi host before maintenance (such as patching or hardware work) is performed. A host cannot enter Maintenance Mode while VMs are still running on it.

There are two ways to clear VMs off a host before entering Maintenance Mode:

| Method | Impact |
|--------|--------|
| Power off all VMs manually | Causes downtime for all VMs on the host |
| DRS automatic vMotion | Zero downtime — VMs are live-migrated to other hosts |

---

#### DRS + Maintenance Mode Workflow

When a host in a DRS cluster is placed into Maintenance Mode, DRS automatically detects this and migrates all VMs off the host using vMotion before the host fully enters Maintenance Mode. This means:

- No VMs are powered off
- No service interruption occurs
- The host is fully evacuated and ready for maintenance

Once maintenance is complete, the host exits Maintenance Mode and VMs can migrate back to it.

---

#### DRS + Update Manager (Patching Workflow)

vSphere Lifecycle Manager (formerly Update Manager) is built into the vCenter Server Appliance and is used to apply patches and updates to ESXi hosts. When combined with DRS, patching can be performed with zero downtime using a rolling approach:

1. Attach hosts to a patch baseline and scan for required updates
2. Update Manager places **ESXi02** into Maintenance Mode → DRS migrates its VMs to ESXi01
3. Patches are applied to ESXi02 → host exits Maintenance Mode → VMs migrate back
4. Update Manager moves on to **ESXi01** → repeats the same process

This rolling update pattern allows patching to be completed during business hours without any VM downtime — one host at a time is taken offline while the rest of the cluster absorbs the workload.

---

> ### Lesson 7: DRS Automation Levels

DRS supports three automation levels that control how much autonomy DRS has over VM placement and migration decisions.

---

#### Automation Levels Comparison

| Level | Initial Placement | Ongoing Migrations |
|-------|------------------|--------------------|
| **Manual** | DRS recommends a host — admin must approve | DRS provides recommendations only — no automatic migrations |
| **Partially Automated** | DRS automatically places the VM on the best host | DRS provides recommendations only — no automatic migrations |
| **Fully Automated** | DRS automatically places the VM on the best host | DRS automatically vMotions VMs to balance workloads |

---

#### Manual Mode

DRS analyzes the cluster and generates recommendations but never acts on them automatically. All changes require manual approval. This is the recommended starting point when first enabling DRS — it allows you to observe what DRS would do before giving it full control.

---

#### Partially Automated Mode

Identical to Manual mode except that **initial placement** is handled automatically. When a VM is powered on, DRS selects the best host without prompting. Ongoing load-balancing migrations still require manual approval.

---

#### Fully Automated Mode

DRS has full control — it places VMs automatically at power-on and continuously migrates VMs via vMotion whenever it detects a workload imbalance. This is the "autopilot" mode.

**Migration Sensitivity** can be adjusted in Fully Automated mode:

| Sensitivity | Behavior |
|-------------|----------|
| High (aggressive) | Migrates VMs even for minor performance improvements |
| Default (recommended) | Migrates VMs only when there is a meaningful performance improvement |
| Low (conservative) | Migrates VMs only when forced — e.g. Affinity Rules or Maintenance Mode |

> **Best Practice:** Leave sensitivity at the default level. Every vMotion consumes CPU resources and network bandwidth on the host. Migrating VMs for marginal gains wastes resources and creates unnecessary overhead.

---

> ### Lesson 8: Resource Fragmentation and HA + DRS

#### What is Resource Fragmentation?

Resource Fragmentation occurs when a cluster has enough total resources to run a VM, but those resources are spread across multiple hosts — so no single host has enough free capacity to accommodate the VM on its own.

**Example:** A large VM needs 32 GB of free memory. The cluster has 40 GB free in total, but it is spread as 10 GB across four hosts. No single host can run the VM, even though the cluster technically has enough resources.

---

#### The Problem: HA Without DRS

When a host fails and HA tries to restart a large VM, it needs a host with sufficient free resources. If no single host qualifies due to fragmentation, HA on its own cannot resolve the situation — it cannot move other VMs around to free up space.

---

#### The Solution: HA + DRS Together

When DRS is enabled alongside HA, the response to a host failure becomes much more intelligent:

1. Host fails → HA triggers a restart for the affected VMs
2. DRS evaluates available resources across the cluster
3. DRS uses vMotion to shuffle smaller VMs to other hosts, freeing up enough resources on one host
4. The large VM boots up on that now-available host
5. Once the failed host is repaired and returns, DRS rebalances the cluster again

> **HA + DRS = the recommended combination.** HA handles recovery. DRS handles intelligent resource allocation. Together they provide both resilience and performance.

---

#### Recommended Configuration

Start with DRS in **Manual mode** to observe recommendations and build confidence. The end goal should be:

- **Fully Automated DRS** + **vSphere HA** enabled on the same cluster

This combination ensures that when a failure occurs, the cluster can not only recover VMs but also make the most efficient use of whatever resources remain after the failure.

---

> ### Lesson 9: Demo — Configuring DRS in vSphere

#### Enabling DRS on a Cluster

Navigate to: **Cluster → Configure → vSphere DRS → Edit**

Toggle the slider to enable DRS, then configure the following options:

---

#### Automation Level and Migration Threshold

| Setting | Behavior |
|---------|----------|
| **Manual** | Recommendations only — no automatic migrations |
| **Partially Automated** | Auto-places VMs at power-on — migrations still require approval |
| **Fully Automated** | Auto-places VMs and automatically migrates for performance (recommended) |

**Migration Threshold (Sensitivity):**

| Level | When DRS Migrates |
|-------|------------------|
| Aggressive | Even for minor performance improvements |
| Default (recommended) | Only for meaningful improvements |
| Conservative | Only for Affinity Rules or Maintenance Mode |

---

#### Additional Options

| Option | Description |
|--------|-------------|
| **VM Distribution** | Equalizes the number of VMs per host — reduces impact of a single host failure |
| **CPU Over-Commitment** | Sets a maximum ratio of vCPUs to physical CPUs in the cluster |
| **Scalable Shares** | Controls how shares scale in Resource Pools (covered separately) |
| **Distributed Power Management (DPM)** | Powers down hosts that are not needed to save electricity — useful for off-peak hours |

---

#### VM Overrides

VM Overrides allow individual VMs to have a different DRS automation level from the rest of the cluster.

Navigate to: **Cluster → Configure → VM Overrides → Add**

**Example:** Set the vCenter Server Appliance to **Manual** mode so it stays on a known host. If vCenter fails, you need to know which host it is on in order to console into it via the Host Client.

---

#### Configuring Affinity and Anti-Affinity Rules

Navigate to: **Cluster → Configure → VM/Host Rules → Add**

| Rule Type | Purpose | Example |
|-----------|---------|---------|
| **Affinity** | Keep specified VMs on the same host | App server + Database server that communicate heavily |
| **Anti-Affinity** | Keep specified VMs on different hosts | Two domain controllers that must not share a host |

---

#### VM/Host Groups and Group Rules

For larger-scale control, create groups first, then apply rules to those groups.

Navigate to: **Cluster → Configure → VM/Host Groups → Add**

1. Create a **VM Group** — add the VMs that share a placement requirement
2. Create a **Host Group** — add the target ESXi hosts
3. Navigate to **VM/Host Rules → Add** and create a group rule

| Rule Wording | Type | HA Can Override? |
|-------------|------|-----------------|
| VMs **must** run on hosts in group | Required / Mandatory | ❌ No |
| VMs **should** run on hosts in group | Preferential | ✅ Yes |

> **Important:** DRS only optimizes for performance. It has no awareness of what your VMs actually do. It is your responsibility to configure overrides, affinity rules, and group rules to prevent DRS from making changes that could cause problems in your environment.

---
