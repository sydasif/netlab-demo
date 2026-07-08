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

### Inventory Structure

See [inventory/README.md](./inventory/README.md) for device details, credentials, and how to add new devices.

## Prerequisites

- Ubuntu 22.04+
- KVM/libvirt
- Vagrant + vagrant-libvirt (hybrid labs)
- Containerlab
- netlab
- Ansible
- [uvx](https://docs.astral.sh/uv/) (for MCP server)

See each lab's documentation for detailed setup and verification steps.
