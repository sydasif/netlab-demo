# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Build / Deploy

| Action | Command | Notes |
|--------|---------|-------|
| Deploy lab-01 (pure Containerlab) | `netlab up` | All devices as containers: IOL (S1, S2) + Linux (H1–H4). No VMs required. |
| Deploy lab-02 (hybrid) | `netlab up` | Mix of libvirt VMs (R1, D1, D2 = IOSv/IOSvL2) + Containerlab (S1, S2, H1–H4). Must use `netlab up`; running `vagrant up` or `containerlab deploy` alone leaves the lab half-connected. |
| Destroy lab | `netlab down --cleanup` | Removes all containers/VMS and stale lock files. |
| Check status | `netlab status` | Shows all devices, providers, and uptime. |
| Connect to device via SSH | `netlab connect <hostname>` | Opens SSH to management IP. Works for both VM and container nodes. |
| Run CLI command on device | `netlab connect <hostname> --show "show version"` | Execute a single show command. |

### Nornir / MCP Tools

| Action | Command | Notes |
|--------|---------|-------|
| List all network devices | `/mcp` then `nornir_list_inventory` | Requires MCP server to be active. |
| Get device facts | `/mcp` then `nornir_get_facts` then `name: <hostname>` | Returns vendor, model, OS version, serial, uptime, interfaces. |
| Get running config | `/mcp` then `nornir_get_config` then `name: <hostname>` | Returns running (and candidate if present) config. |
| Run NAPALM getter | `/mcp` then `nornir_run_getter` then `getter: <name>` then `name: <hostname>` | E.g. `interfaces`, `vlans`, `arp_table`, `routes`. |
| Execute CLI command | `/mcp` then `nornir_run_cli` then `commands: ["show version"]` then `name: <hostname>` | e.g. `show ip ospf neighbor`, `show vrrp brief`. |
| Reload inventory from disk | `/mcp` then `nornir_reload_inventory` | Pick up changes to inventory/hosts.yaml etc. |
| List available NAPALM getters per platform | `/mcp` then `nornir_list_getters` | Returns 21 getters for ios: arp_table, interfaces, vlans, etc. |

### Git

| Action | Command | Notes |
|--------|---------|-------|
| Check working tree | `git status` | Shows modified files, staging area. |
| View recent commits | `git log --oneline -10` | Shows last 10 commits. |
| Show diff of staged changes | `git diff --cached` | Before committing. |

## High-Level Architecture

This repository demonstrates **network lab automation** using two complementary toolchains:

### Lab-01 (Pure Containerlab)
- All 5 network devices + 4 hosts run as Docker containers via Containerlab
- Topology: `lab-01/topology.yml` defines nodes, VLANs, links, and groups
- Device roles:
  - **R1**: Core router (cisco_iol, CORE group)
  - **D1, D2**: Distribution switches with IRB & VRRP (DIST group)
  - **S1, S2**: Access switches, L2 forwarding only (ACCESS group)
  - **H1–H4**: Alpine Linux hosts on VLAN 10/20
- Configuration is **pre-generated** (Ansible templates + static configs in `configs/`)
- `netlab up` generates Containerlab config, starts containers, applies config via Ansible
- Nornir MCP server (`.mcp.json` + `config.yaml`) provides AI-assisted network automation on top

### Lab-02 (Hybrid: libvirt VMs + Containerlab)
- Topology: `lab-02/topology.yml` — mixes Cisco IOSv/IOSvL2 VMs (via Vagrant/libvirt) with IOL containers + Linux containers
- Key constraint: **primary provider must be `libvirt`**; `clab` can only be a secondary (per-node) provider
- Startup order is critical: `vagrant up` first (boots VMs, discovers bridge names), then `containerlab deploy` (boots containers, attaches to same bridges)
- See `lab-02/README.md` for full deployment and verification steps

### Inventory & Credentials

- **`inventory/hosts.yaml`** — Nornir inventory read by MCP server (NORNIR_CONFIG → `config.yaml`)
  - One shared inventory serves whichever lab is up; management IPs and credentials must match the active lab
  - lab-01 (all IOL): every device uses `admin/admin` (netlab IOL default, `group_vars/iol/topology.json`)
  - lab-02: R1/D1/D2 (IOSv/IOSvL2 VMs) use `vagrant/vagrant`; S1/S2 (IOL containers) use `admin/admin` — current `groups.yaml` matches this lab
- **`inventory/groups.yaml`** — group definitions (CORE, DIST, ACCESS) with per-group usernames/passwords/platform
- **`inventory/defaults.yaml`** — Napalm connection extras (no agent, no key lookup)

### Key Design Decisions (from netlab)

1. **Provider per device type**: IOSv/IOSvL2 → libvirt/Vagrant; IOL/Alpine → Containerlab/clab. This avoids running heavy VMs for lightweight nodes and excluding NOSes without container images.

2. **OVS bridges for multi-access links**: `defaults.providers.clab.bridge_type: ovs-bridge` in `lab-01/clab.yml`. Standard Linux bridges filter STP/LLDP frames, disrupting switching features. OVS passes them through transparently.

3. **VRRP for gateway redundancy**: Distribution switches D1/D2 use VRRP with priorities (110 = primary, 100 = backup) across VLAN 10/VLAN 20. Custom templates in `configs/` (D1.cfg, D2.cfg) supplement Ansible-generated config.

4. **Single source of truth**: `topology.yml` (lab-02) and `topology.yml` (lab-01) define the entire lab — device types, images, VLANs, groups, links, modules. netlab generates Vagrantfile, clab.yml, Ansible inventory, and the snapshot pickle from this one file.

### Common Gotchas

| Issue | Fix |
|-------|-----|
| `NORNIR_CONFIG` points at directory | Set to absolute path of `config.yaml` in `.mcp.json` |
| IOL devices fail auth | IOL containers use `admin/admin`; IOSv/IOSvL2 VMs use `vagrant/vagrant` — match creds to the active lab in `groups.yaml` |
| MCP server won't start after config change | Kill any `uvx nornir-napalm-mcp` processes and reconnect `/mcp` |
| `netlab up` fails on OVS | Ensure `openvswitch-switch` is installed: `sudo apt install openvswitch-switch` |
| Host key changed SSH warning | SSH host keys auto-trusted via `ansible.cfg` (`UserKnownHostsFile=/dev/null`) — if MCP still fails, reload inventory (`nornir_reload_inventory`) |

### What's Already in This Repo (do not duplicate)

- `README.md` — high-level project overview, prerequisites, setup, topology, verification, cleanup
- `lab-01/README.md` — Containerlab-specific deployment and verification
- `lab-02/README.md` — hybrid lab (VM + container) guide with full deployment sequence
- `config.yaml` — Nornir MCP server config (do not change `NORNIR_CONFIG` to a directory!)
- `inventory/` — Nornir inventory files (hosts.yaml, groups.yaml, defaults.yaml)
- `.mcp.json` — MCP server environment config
- Two topology files: `lab-01/topology.yml` (pure container) and `lab-02/topology.yml` (hybrid)
- Device configs in `lab-01/configs/` (D1.cfg, D2.cfg — VRRP templates)
- `lab-01/clab.yml` and `lab-01/clab-campus/` — Containerlab and Ansible wiring
- Nornir MCP tool definitions (all 9 tools listed in `README.md`)