# Building Hybrid Network Labs with netlab: Combining Containerlab and Vagrant

## 1. Introduction

Not every network operating system ships in the same format. Cisco IOSv
and IOSvL2 are distributed as virtual machine images and require KVM —
there are no container images for them. On the other end of the spectrum,
Cisco IOL and Alpine Linux run natively in containers, needing only a
few megabytes of overhead.

A hybrid lab lets you run each device in its native environment: virtual
machines for the NOSes that need them, containers for everything else.
netlab orchestrates both backends from a single topology file, so you
don't have to manage Vagrant and Containerlab separately.

By the end of this guide you'll have built a working campus topology
that mixes both providers:

| Role         | Nodes          | Device | Provider |
| ------------ | -------------- | ------ | -------- |
| Core router  | R1             | IOSv   | libvirt  |
| Distribution | D1, D2         | IOSvL2 | libvirt  |
| Access       | S1, S2         | IOL-L2 | clab     |
| Hosts        | H1, H2, H3, H4 | Linux  | clab     |

All nine nodes are defined in one `topology.yml` and deployed with a
single `netlab up` command — netlab handles the cross-provider wiring,
bridge discovery, and configuration deployment automatically.

---

## 2. Why Build a Hybrid Lab?

Cisco distributes its network operating systems in different formats:

- **IOSv and IOSvL2** are QEMU/KVM virtual machine images. There are no
  native container images for them — they require a full VM, consuming
  around 1 GB of RAM each.
- **IOL (IOS On Linux)** is a binary that runs directly on the Linux
  kernel. It doesn't need a hypervisor, so it uses less CPU and memory
  than the equivalent IOSv VM. Cisco's documentation notes that IOL nodes
  "consume much less CPU and memory than an equivalent IOSv or CSR 1000v
  node in your lab."
- **Alpine Linux** is a few megabytes as a container image, yet netlab
  would allocate 1 GB of RAM to it if you ran it as a VM — just for
  `ping` and `bash`.

Building a lab with a single provider forces you to pick the least common
denominator: run everything as VMs (wasting resources on lightweight
nodes) or run everything as containers (excluding devices that have no
container image). A hybrid lab avoids both tradeoffs.

| Device    | Best Backend    | Why                           |
| --------- | --------------- | ----------------------------- |
| IOSv      | Vagrant/libvirt | No native container image     |
| IOSvL2    | Vagrant/libvirt | No native container image     |
| Cisco IOL | Containerlab    | Runs as a native Linux binary |
| Linux     | Containerlab    | Runs as a native Alpine image |

> Run every device in its native environment.

---

## 3. Prerequisites

You need a Linux server (Ubuntu recommended) with the following toolchain
installed. Verify each component before proceeding:

```bash
# KVM hardware support
kvm-ok

# libvirt
virsh list --all

# Vagrant + vagrant-libvirt plugin
vagrant --version
vagrant plugin list

# Docker + Containerlab
docker --version
containerlab version

# netlab
pip3 show networklab | grep Version

# User group membership — must include libvirt, vagrant, docker
groups
```

If any check fails, see the [netlab Ubuntu installation guide](https://netlab.tools/install/ubuntu/) for setup steps.

---

## 4. Understanding netlab Providers

A virtualization provider is the backend that runs your lab devices. netlab supports three of them:

| Provider | Backend                          | Use case                    |
| -------- | -------------------------------- | --------------------------- |
| libvirt  | Vagrant + KVM/QEMU               | VM-based NOS images         |
| clab     | Containerlab + Docker            | Container-native devices    |
| external | Physical or pre-existing devices | Hardware or unmanaged nodes |

### Provider Resolution

netlab decides which provider to use for each node by checking these locations in order:

1. **Per-node** `provider` attribute — highest precedence
2. **Group** `provider` attribute — applied to all group members
3. **Topology-level** `provider` — the default for all nodes
4. **System default** — `libvirt` if nothing else is set

In our topology, `provider: libvirt` is set at the top level. Nodes without a `provider` attribute (R1, D1, D2) inherit it. Nodes with `provider: clab` (S1, S2, H1–H4) override it.

### Hybrid Mode Constraint

When mixing providers, the primary (topology-level) provider must be `libvirt`. `clab` can only be a secondary (per-node) provider. This is a hard constraint — netlab needs to orchestrate the startup sequence (Vagrant first, then Containerlab) and bridge the two environments.

### Provider-Specific Images

Each device type can use different images depending on the provider. Set them in `defaults.devices`:

```yaml
defaults:
  devices:
    iosv.libvirt.image: cisco/iosv:15.9.M3
    ioll2.clab.image: asifsyd/cisco_iol:l2-17.12.01
```

The `.libvirt.` variant is used when the node runs on libvirt; the `.clab.` variant is used when it runs on Containerlab.

---

## 5. Building the Topology

Here is the complete `topology.yml` file. It defines a campus network with a core router, two distribution switches, two access switches, and
four hosts — each running on its native provider.

```yaml
---
name: campus
provider: libvirt
version: 1.0

defaults:
  devices:
    iosv.libvirt.image: cisco/iosv:15.9.M3
    iosv.warnings.paramiko: false
    iosvl2.libvirt.image: cisco/iosvl2:15.2
    iosvl2.warnings.paramiko: false
    ioll2.clab.image: asifsyd/cisco_iol:l2-17.12.01
    linux.clab.image: alpine:latest

nodes:
  R1:
    device: iosv
    module: [ospf]
  D1:
    device: iosvl2
  D2:
    device: iosvl2
  S1:
    device: ioll2
    provider: clab
  S2:
    device: ioll2
    provider: clab
  H1:
    device: linux
    provider: clab
  H2:
    device: linux
    provider: clab
  H3:
    device: linux
    provider: clab
  H4:
    device: linux
    provider: clab

vlans:
  VLAN_10:
    id: 10
    links: [H1-S1, H4-S2]
  VLAN_20:
    id: 20
    links: [H2-S2, H3-S1]

groups:
  DISTRIBUTION:
    members: [D1, D2]
    module: [vlan, ospf]
    config: [template]
    vlan:
      mode: irb
  ACCESS:
    members: [S1, S2]
    module: [vlan]
    vlan:
      mode: bridge
  VLAN_10:
    members: [H1, H4]
    device: linux
    module: [routing]
    role: host
    routing.static:
      - ipv4: 0.0.0.0/0
        nexthop.ipv4: 172.16.0.254
  VLAN_20:
    members: [H2, H3]
    device: linux
    module: [routing]
    role: host
    routing.static:
      - ipv4: 0.0.0.0/0
        nexthop.ipv4: 172.16.1.254

links:
  - R1-D1
  - R1-D2
  - group: access_trunk
    vlan.trunk: [VLAN_10, VLAN_20]
    members: [D1-S1, D1-S2, D2-S1, D2-S2]
```

### Top-Level Attributes

- `name: campus` — used by netlab to name Linux bridges and container prefixes. Containers will be created as `clab-campus-<node>`.
- `provider: libvirt` — the primary virtualization provider. Nodes without their own `provider` attribute inherit this.
- `version: 1.0` — the minimum netlab version required. If an older version tries to load this file, netlab will refuse.

### defaults.devices

This block sets provider-specific container images and device-level settings. The syntax is `device.<type>.<provider>.attribute`:

```yaml
iosv.libvirt.image: cisco/iosv:15.9.M3
iosv.warnings.paramiko: false
```

The `.libvirt.` variant tells netlab which Vagrant box to use when the node runs on libvirt. The `.clab.` variant tells it which Docker image to use when the node runs on containerlab.

The `warnings.paramiko: false` lines suppress Ansible paramiko warnings that Cisco IOS devices trigger during configuration deployment.

### nodes

All nine lab devices are defined as a dictionary. Each node has a `device` type and an optional list of configuration `module`s.
Nodes running on the secondary provider must include `provider: clab`:

| Node | Device | Provider | Module       |
| ---- | ------ | -------- | ------------ |
| R1   | iosv   | libvirt  | ospf         |
| D1   | iosvl2 | libvirt  | (from group) |
| D2   | iosvl2 | libvirt  | (from group) |
| S1   | ioll2  | clab     | (from group) |
| S2   | ioll2  | clab     | (from group) |
| H1   | linux  | clab     | (from group) |
| H2   | linux  | clab     | (from group) |
| H3   | linux  | clab     | (from group) |
| H4   | linux  | clab     | (from group) |

### vlans

Two VLANs are defined globally. The `links` attribute within each VLAN is a shortcut — it tells netlab which switch ports are access members of that VLAN, saving you from writing out `vlan.access` on each link:

- **VLAN 10** — Hosts H1 and H4 connected to switches S1 and S2. IP subnet is auto-assigned from the default LAN pool.
- **VLAN 20** — Hosts H2 and H3 connected to switches S2 and S1.

### groups

Groups apply shared settings to multiple nodes at once:

- **DISTRIBUTION** — D1 and D2 run the `vlan` and `ospf` modules, use IRB mode (integrated routing and bridging), and pull custom configuration from the `template/` directory (Jinja2 files for VRRP configuration).
- **ACCESS** — S1 and S2 run the `vlan` module in pure bridging mode. They forward traffic without routing.
- **VLAN_10** and **VLAN_20** — Host groups that set `device: linux`, add a static default route pointing at the VRRP virtual IP, and mark the nodes as `role: host` (no loopback address).

### links

Two point-to-point links connect the core router to the distribution switches:

```yaml
- R1-D1
- R1-D2
```

A link group applies trunk attributes to all four distribution-to-access links at once:

```yaml
- group: access_trunk
  vlan.trunk: [VLAN_10, VLAN_20]
  members: [D1-S1, D1-S2, D2-S1, D2-S2]
```

This is equivalent to writing four separate link definitions with `vlan.trunk` on each one.

---

## 6. Assigning Providers

Every node in the topology uses the provider that best matches its operating system's distribution format.

### Provider per Node

| Node | Device | Provider | Reason                                        |
| ---- | ------ | -------- | --------------------------------------------- |
| R1   | iosv   | libvirt  | IOSv is a KVM image — no container exists     |
| D1   | iosvl2 | libvirt  | Same as above                                 |
| D2   | iosvl2 | libvirt  | Same as above                                 |
| S1   | ioll2  | clab     | IOL is a Linux binary packaged as a container |
| S2   | ioll2  | clab     | Same as above                                 |
| H1   | linux  | clab     | Alpine runs natively as a Docker container    |
| H2   | linux  | clab     | Same as above                                 |
| H3   | linux  | clab     | Same as above                                 |
| H4   | linux  | clab     | Same as above                                 |

R1, D1, and D2 have no `provider` attribute — they inherit `provider: libvirt` from the topology level. S1, S2, and H1–H4 override with `provider: clab`.

### Cross-Provider Links

When a link connects two libvirt nodes (e.g., R1–D1), netlab uses a UDP point-to-point tunnel — fast and transparent. When a link connects a libvirt node to a clab node (e.g., D1–S1), netlab automatically detects the mismatch and replaces the tunnel with a Linux bridge. Containerlab can then attach the container's vEth pair to that bridge. This happens transparently — you just define the link as `D1-S1` and netlab does the rest.

### Deployment Sequence

Because cross-provider links require bridge name coordination, the startup order is critical:

1. `vagrant up` — boots the libvirt VMs (R1, D1, D2)
2. netlab inspects the libvirt networks to discover Linux bridge names
3. `containerlab deploy` — boots the containers (S1, S2, H1–H4), wiring them to the same bridges the VMs are connected to

This is why you **must** use `netlab up` and `netlab down` for hybrid labs. Running `vagrant up` or `containerlab deploy` alone would leave the lab half-connected and hard to clean up.

---

## 7. Deploying the Lab

With the topology file ready, one command creates, starts, and configures the entire lab:

```bash
netlab up
```

### What Happens Step by Step

**1. Topology Validation and Configuration Generation**

`netlab up` calls `netlab create` to read `topology.yml`, merge it with system defaults, and validate the data model. It then generates:

- **Vagrantfile** — describes the libvirt VMs (R1, D1, D2) with their box images, memory, and network interfaces
- **clab.yml** — describes the containerlab topology (S1, S2, H1–H4) with their Docker images and links
- **hosts.yml** + **ansible.cfg** — Ansible inventory and configuration for initial device configuration
- **netlab.snapshot.pickle** — the transformed topology snapshot used by all subsequent netlab commands

**2. VM Deployment**

netlab runs `vagrant up` to boot the libvirt nodes. Vagrant:

1. Downloads the Vagrant boxes (`cisco/iosv`, `cisco/iosvl2`) if needed
2. Creates KVM domains for R1, D1, and D2
3. Creates the management network (`vagrant-libvirt`) and data-plane bridges
4. Powers on the VMs

**3. Bridge Discovery**

With the VMs running, netlab inspects the libvirt networks and discovers the Linux bridge names for each data-plane link. These bridge names are then injected into the containerlab configuration — this is the key step that makes cross-provider links work.

**4. Container Deployment**

netlab runs `sudo containerlab deploy -t clab.yml` to start the container nodes. Containerlab:

1. Pulls the Docker images (`asifsyd/cisco_iol`, `alpine:latest`) if needed
2. Creates containers for S1, S2, and H1–H4
3. Attaches each container's vEth interfaces to the same Linux bridges that the VM interfaces are connected to
4. Creates the containerlab management network (`clab`)

**5. Initial Configuration**

Finally, netlab runs `netlab initial` to deploy device configurations via Ansible. This is a five-step process:

1. **Wait for readiness** — netlab polls SSH ports until all devices accept connections
2. **Initial config** — hostname, interface IP addresses, descriptions, LLDP, and basic system settings
3. **Normalize** — device-specific cleanup (e.g., removing default VLANs from IOSvL2)
4. **Module config** — OSPF, VLANs, VRRP, and static routes as defined in the topology
5. **Custom templates** — any templates specified in `config:` group attributes (for example, VRRP configuration from the `template/` directory)

### After Deployment

Once `netlab up` finishes, all nine devices are running, wired, and configured. The topology is ready for traffic — hosts can ping the router, OSPF adjacencies are established, and VRRP provides gateway redundancy.

---

## 8. Verifying the Hybrid Lab

Once `netlab up` finishes, confirm that all VMs and containers are running and the network functions correctly.

### Check the VMs

```bash
virsh list
```

The libvirt domain names use the lab name (`campus`) as a prefix:

```
 Id   Name         State
---------------------------
 3    campus_R1    running
 6    campus_D1    running
 7    campus_D2    running
```

### Check the Containers

```bash
docker ps --format "table {{.Names}}"
```

Containerlab uses the `clab-<labname>-<nodename>` convention:

```
Names
clab-campus-S1
clab-campus-S2
clab-campus-H1
clab-campus-H2
clab-campus-H3
clab-campus-H4
```

### Check Everything with netlab

```bash
netlab status
```

This shows all nine devices in one table:

```
Lab default in /path/to/lab-02
  status: started
  provider(s): libvirt, clab
┏━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━┓
┃ node ┃ device ┃ image          ┃ mgmt IPv4       ┃ provider ┃ status    ┃
┡━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━┩
│ R1   │ iosv   │ cisco/iosv     │ 192.168.121.101 │ libvirt  │ Up 5 min  │
│ D1   │ iosvl2 │ cisco/iosvl2   │ 192.168.121.102 │ libvirt  │ Up 5 min  │
│ D2   │ iosvl2 │ cisco/iosvl2   │ 192.168.121.103 │ libvirt  │ Up 5 min  │
│ S1   │ ioll2  │ cisco_iol      │ 192.168.121.104 │ clab     │ Up 5 min  │
│ S2   │ ioll2  │ cisco_iol      │ 192.168.121.105 │ clab     │ Up 5 min  │
│ H1   │ linux  │ alpine:latest  │ 192.168.121.106 │ clab     │ Up 5 min  │
│ H2   │ linux  │ alpine:latest  │ 192.168.121.107 │ clab     │ Up 5 min  │
│ H3   │ linux  │ alpine:latest  │ 192.168.121.108 │ clab     │ Up 5 min  │
│ H4   │ linux  │ alpine:latest  │ 192.168.121.109 │ clab     │ Up 5 min  │
└──────┴────────┴────────────────┴─────────────────┴──────────┴───────────┘
```

### Verify Connectivity

From the core router, check OSPF adjacencies:

```bash
netlab connect R1
R1# show ip ospf neighbor
```

From a distribution switch, verify VLANs and VRRP:

```bash
netlab connect D1
D1# show vlan brief
D1# show vrrp brief
```

From a host, confirm reachability across the campus. For example,
ping H2 (VLAN 20) from H1 (VLAN 10):

```bash
docker exec -it clab-campus-H1 ping -c 3 172.16.1.7
```

A successful reply means the Linux bridges, VLAN trunks, OSPF routing,
and VRRP gateway are all working correctly across both providers.

---

## 9. Connecting to Devices

netlab provides a single command to connect to any lab device,
regardless of provider.

### netlab connect

```bash
netlab connect R1       # IOSv router — SSH
netlab connect D1       # IOSvL2 distribution switch — SSH
netlab connect S1       # IOL-L2 access switch — SSH
netlab connect H1       # Alpine host — docker exec
```

`netlab connect` reads the lab snapshot to determine the correct
connection method. For network devices (R1, D1, D2, S1, S2) it
opens an SSH session to the management IP. For Linux containers
(H1–H4) it uses `docker exec`.

You can also execute a single command:

```bash
netlab connect R1 --show "show ip route"
netlab connect H1 --show "ifconfig"
```

### Direct docker exec

Linux containers can be reached directly with Docker:

```bash
docker exec -it clab-campus-H1 /bin/sh
```

For IOL containers (S1, S2), docker exec gives you a Linux shell inside the container — not the IOS CLI. Use `netlab connect S1`
for the standard IOS command-line experience.

### Manual SSH

For VM-based nodes, SSH directly to the management IP:

```bash
ssh cisco@192.168.121.101
```

Management IPs are shown by `netlab status`. Default credentials are defined by the Vagrant box — typically `cisco`/`cisco` for
IOSv boxes.

---

## 10. Best Practices

- **Use libvirt as the primary provider** in hybrid labs. clab can only be a secondary provider — this is a hard netlab constraint.
- **Use Containerlab only for container-native NOSes.** IOL, Alpine Linux, FRR, and cEOS have native container images that run efficiently under Docker. IOSv and IOSvL2 do not — they must run as VMs.
- **Always use `netlab up` and `netlab down`** for hybrid labs. Running `vagrant up` or `containerlab deploy` alone will leave bridges disconnected and be difficult to clean up.
- **Set provider-specific images explicitly** in `defaults.devices`. This makes the topology portable and documents exactly which image each device expects.
- **Use groups to reduce duplication.** The DISTRIBUTION and ACCESS groups in this topology apply VLAN mode, configuration modules, and custom templates to multiple nodes with a single block of YAML.
- **Keep topology files in version control.** A declarative topology file coupled with netlab's automation means anyone can reproduce the exact same lab from source.
- **Use custom templates for complex configurations.** The `config: [template]` group attribute pulls Jinja2 files from the `template/` directory — useful for VRRP, interface descriptions, or any vendor-specific feature netlab does not generate natively.

---

## 11. Conclusion

You've built a working hybrid campus lab that mixes Cisco IOSv VMs, IOSvL2 VMs, IOL-L2 containers, and Alpine Linux containers — all
defined in a single `topology.yml` and deployed with one command.

The key takeaway is simple: run every device in its native environment. Virtual machines for the NOSes that need them, containers for everything else, and let netlab handle the wiring between the two.

This foundation can be extended in many directions: add more vendors (Juniper vSRX, Arista cEOS, Nokia SR Linux), introduce EVPN/VXLAN fabric modules, or connect the lab to external networks. Whatever path you take, the hybrid approach ensures you get the best mix of performance, resource efficiency, and device fidelity.
