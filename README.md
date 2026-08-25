# netlab-demo

Multi-layer network lab examples using [netlab](https://netlab.tools/), with a [Nornir MCP server](https://github.com/sydasif/nornir-napalm-mcp) for AI-assisted network automation.

## Labs

| Lab                 | Description                                                               |
| ------------------- | ------------------------------------------------------------------------- |
| [lab-01](./lab-01/) | Pure Containerlab lab — all devices run as containers                     |
| [lab-02](./lab-02/) | Hybrid lab — libvirt VMs (IOSv/IOSvL2) mixed with containers (IOL, Linux) |

## Nornir MCP Server

A Model Context Protocol (MCP) server that exposes network devices to AI assistants via [Nornir](https://nornir.readthedocs.io/) + [NAPALM](https://napalm.readthedocs.io/).

**Source:** [github.com/sydasif/nornir-napalm-mcp](https://github.com/sydasif/nornir-napalm-mcp)

### Available Tools

| Tool                      | Description                                             |
| ------------------------- | ------------------------------------------------------- |
| `nornir_list_inventory`   | List all devices in the inventory                       |
| `nornir_get_facts`        | Get device facts (vendor, model, OS, serial)            |
| `nornir_get_config`       | Retrieve running/startup configuration                  |
| `nornir_run_getter`       | Run any NAPALM getter (interfaces, routes, VLANs, etc.) |
| `nornir_run_cli`          | Execute CLI commands on devices                         |
| `nornir_ping`             | Send ICMP ping from a device                            |
| `nornir_list_getters`     | List available NAPALM getters per platform              |
| `nornir_reload_inventory` | Reload inventory from disk                              |

### Connecting

In Claude Code, run `/mcp` to reconnect the server. The MCP config is in `.mcp.json`.

### Credentials

One shared Nornir inventory (`inventory/`) serves whichever lab is currently up — there is a single `config.yaml` and `.mcp.json`. Credentials differ per lab because the device platforms differ:

| Lab | Devices | Username | Password |
| --- | ------- | -------- | -------- |
| lab-01 (all IOL containers) | R1, D1, D2, S1, S2 | `admin` | `admin` |
| lab-02 (IOSv/IOSvL2 VMs)    | R1, D1, D2          | `vagrant` | `vagrant` |
| lab-02 (IOL containers)     | S1, S2              | `admin` | `admin` |

When switching labs, update `inventory/groups.yaml` (per-group `username`/`password`) and `inventory/hosts.yaml` (management IPs from `netlab status`) to match the active lab.

### Inventory Structure

Device details live in [inventory/hosts.yaml](./inventory/hosts.yaml), group definitions in [inventory/groups.yaml](./inventory/groups.yaml), and NAPALM connection options in [inventory/defaults.yaml](./inventory/defaults.yaml).

## Prerequisites

- Ubuntu 22.04+
- KVM/libvirt
- Vagrant + vagrant-libvirt (hybrid labs)
- Containerlab
- netlab
- Ansible
- [uvx](https://docs.astral.sh/uv/) (for MCP server)

See each lab's documentation for detailed setup and verification steps.
