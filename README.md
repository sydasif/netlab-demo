# netlab-demo

Network lab examples using [Netlab.tools](https://netlab.tools/), with a [Nornir MCP](https://github.com/sydasif/nornir-napalm-mcp) for AI-assisted network automation.

## Labs

| Lab                 | Description                                                               |
| ------------------- | ------------------------------------------------------------------------- |
| [lab-01](./lab-01/) | OSPF lab — six Cisco IOL routers, one Linux host, and Graphviz output     |
| [lab-02](./lab-02/) | Hybrid lab — libvirt VMs (IOSv/IOSvL2) mixed with containers (IOL, Linux) |

## Nornir MCP Server

A Model Context Protocol (MCP) server that exposes network devices to AI assistants via [Nornir](https://nornir.readthedocs.io/) + [NAPALM](https://napalm.readthedocs.io/).

**Source:** [github.com/sydasif/nornir-napalm-mcp](https://github.com/sydasif/nornir-napalm-mcp)

### Connecting

In `Claude Code Cli`, run `/mcp` to reconnect the server. The MCP config is in `.mcp.json`.

### Credentials

The Nornir inventory (`inventory/`) currently serves `lab-02`; `lab-01` is not included. There is a single `config.yaml` and `.mcp.json`. Credentials differ by provider and lab type, as shown below:

| Provider | Username | Password |
| ------- | -------- | -------- |
| Libvirt | `vagrant` | `vagrant` |
| Clab | `admin` | `admin` |

Ensure `inventory/groups.yaml` (per-group `username`/`password`) and `inventory/hosts.yaml` (management IPs from `netlab status`) match these device credentials.

### Inventory Structure

Device details live in [inventory/hosts.yaml](./inventory/hosts.yaml), group definitions in [inventory/groups.yaml](./inventory/groups.yaml), and NAPALM connection options in [inventory/defaults.yaml](./inventory/defaults.yaml).

## Prerequisites

- Ubuntu `22.04+`
- KVM/libvirt
- Vagrant + vagrant-libvirt (hybrid labs)
- Containerlab
- netlab

See each lab's documentation for detailed setup and verification steps.
