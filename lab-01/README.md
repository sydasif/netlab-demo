# Netlab Demo Project

This project sets up a multi-layer network lab using Netlab tool (Containerlab), demonstrating a robust and redundant network architecture. It includes core, distribution, access, and host layers, configured with VLANs, VRRP for gateway redundancy, and OSPF routing.

## Table of Contents

- [Netlab Demo Project](#netlab-demo-project)
  - [Table of Contents](#table-of-contents)
  - [Project Overview](#project-overview)
  - [Prerequisites](#prerequisites)
  - [Setup Instructions](#setup-instructions)
  - [Network Topology](#network-topology)
    - [Open vSwitch Bridges](#open-vswitch-bridges)
    - [Custom Configuration Templates](#custom-configuration-templates)
  - [Verification Steps](#verification-steps)
  - [Troubleshooting Tips](#troubleshooting-tips)
  - [Cleanup](#cleanup)

## Project Overview

This Containerlab-based network lab simulates a typical enterprise network design. It features:

- **Core Layer**: A single router (R1) responsible for external connectivity (if extended).
- **Distribution Layer**: Two Layer 3 switches (D1, D2) providing routing and switching functions, including Integrated Routing and Bridging (IRB) for VLAN interfaces and VRRP for gateway redundancy.
- **Access Layer**: Two Layer 2 switches (S1, S2) connecting end-user devices (hosts) to the network.
- **Host Layer**: Four Linux hosts (H1, H2, H3, H4) connected to the access layer switches.

The lab utilizes Cisco IOL images for network devices and Alpine Linux for hosts. Ansible is used for configuration management, with Jinja2 templates generating device-specific configurations.

Multi-access links use Open vSwitch bridges instead of standard
Linux bridges to ensure layer-2 control protocols (STP, LLDP) pass
through without interference.

## Prerequisites

To deploy and run this lab, you need the following installed on your system:

- **Containerlab**: A tool for orchestrating and managing container-based networking labs.
  - [Installation Guide](https://containerlab.dev/install/)
- **Docker**: Containerization platform used by Containerlab.
  - [Installation Guide](https://docs.docker.com/engine/install/)
- **Ansible**: For configuration management (though configurations are pre-generated in this demo).
  - [Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html)
- **Netlab Tool** A virtual networking labbing tool
  - [Installation Guide](https://netlab.tools/install/)

Verify each component before proceeding:

```bash
# Docker
docker --version

# Containerlab
containerlab version

# netlab
pip3 show networklab | grep Version

# Ansible
ansible-playbook --version
```

## Setup Instructions

1.  **Clone the Repository**:

    ```bash
    git clone https://github.com/sydasif/netlab-demo.git
    cd netlab-demo
    ```

2.  **Deploy the Lab**:

    ```bash
    netlab up
    ```

    This command generates the Containerlab configuration, creates
    Linux bridges for multi-access links, deploys the containers, and
    applies initial device configurations (hostname, IP addresses, OSPF,
    VLANs, VRRP) via Ansible.

Because this lab uses OVS bridges (see Network Topology below),
Open vSwitch must be installed for `netlab up` to succeed.

3.  **Access Devices**:
    You can access the devices via SSH. Containerlab will provide connection details after deployment. For example, to access R1:
    ```bash
    netlab connect R1
    ```

## Network Topology

The lab consists of the following nodes and their roles:

- **R1 (Core Router)**: Cisco IOL router.
- **D1, D2 (Distribution Switches)**: Cisco IOL L2 switches with IRB.
- **S1, S2 (Access Switches)**: Cisco IOL L2 switches.
- **H1, H2, H3, H4 (Hosts)**: Alpine Linux containers.

**VLANs and IP Addressing:**

- **VLAN 10**: Network `172.16.0.0/24`
  - Gateway: `172.16.0.254` (VRRP virtual IP)
  - Hosts: H1 (`172.16.0.6`), H4 (`172.16.0.9`)
- **VLAN 20**: Network `172.16.1.0/24`
  - Gateway: `172.16.1.254` (VRRP virtual IP)
  - Hosts: H2 (`172.16.1.7`), H3 (`172.16.1.8`)
- **Point-to-Point Links**: `10.1.0.0/16`
- **Loopbacks**: `10.0.0.0/24`
- **Management Network**: `192.168.121.0/24`

**VRRP Configuration:**

- **VLAN 10 Gateway (172.16.0.254)**:
  - D1: VRRP priority 110 (Primary)
  - D2: VRRP priority 100 (Backup)
- **VLAN 20 Gateway (172.16.1.254)**:
  - D1: VRRP priority 100 (Backup)
  - D2: VRRP priority 110 (Primary)

All IP addressing is built-in. See [Using Built-In Address Pools](https://netlab.tools/example/addressing-tutorial/)

### Open vSwitch Bridges

The topology uses OVS bridges for multi-access links instead of
standard Linux bridges:

```yaml
defaults:
  providers.clab.bridge_type: ovs-bridge
```

OVS passes layer-2 control protocols (STP, LLDP, LACP) without
interference. Standard Linux bridges filter some of these frames,
which can disrupt switching features in lab scenarios.

### Custom Configuration Templates

The DISTRIBUTION group pulls VRRP configuration from the `configs/`
directory:

```yaml
groups:
  DISTRIBUTION:
    config: [configs]
```

Files in `configs/` (`D1.cfg`, `D2.cfg`) contain static VRRP config
deployed by Ansible during `netlab up`. D1 is primary for VLAN 10,
D2 is primary for VLAN 20.

## Verification Steps

After deploying the lab, you can verify its functionality:

1.  **Check VRRP Status**:
    Log in to D1 and D2 and check the VRRP status for VLAN 10 and VLAN 20 interfaces.

    ```bash
    # On D1 or D2
    show vrrp brief
    ```

    You should see D1 as Master for VLAN 10 and D2 as Master for VLAN 20.

2.  **Ping between Hosts in the Same VLAN**:
    Ping from H1 to H4 (VLAN 10) and from H2 to H3 (VLAN 20).

    ```bash
    # On H1
    ping 172.16.0.9

    # On H2
    ping 172.16.1.8
    ```

3.  **Ping between Hosts in Different VLANs**:
    Ping from H1 (VLAN 10) to H2 (VLAN 20). This traffic should be routed via the VRRP gateways on D1/D2 and then through R1.

    ```bash
    # On H1
    ping 172.16.1.7
    ```

4.  **Check OSPF Neighbors**:
    Log in to R1, D1, and D2 to verify OSPF neighbor relationships.
    ```bash
    # On R1, D1, or D2
    show ip ospf neighbor
    ```

## Troubleshooting Tips

- **Containerlab Issues**: If containers fail to start, check `netlab status`
  for error details. Ensure Docker is running and Containerlab is correctly
  installed with `containerlab version`.

- **OVS Bridge Errors**: If `netlab up` fails with OVS-related errors,
  verify Open vSwitch is installed (`ovs-vsctl --version`). On Ubuntu:
  `sudo apt install openvswitch-switch`.

- **Container Images Not Found**: Verify the required Docker images are
  present (`docker image ls`). If missing, netlab will attempt to pull
  them automatically, but this may fail without internet access or
  appropriate credentials.

- **Container Name Conflicts**: If you see container name conflicts, another
  lab instance may be running. Stop it with `netlab down --cleanup` in its
  directory, or use `docker rm -f clab-campus-<name>` to remove stale
  containers.

- **netlab Lock Errors**: If you see "lab instance already running" or
  a stale `netlab.lock` file, run `netlab down --cleanup` in the lab
  directory to release the lock.

## Cleanup

To destroy the lab and remove all containers:

```bash
netlab down --cleanup
```
