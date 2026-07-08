# Nornir Inventory

Nornir inventory files for the [nornir-napalm-mcp](https://github.com/sydasif/nornir-napalm-mcp) server. These files map the lab devices deployed by `netlab up` (lab-02) into a format Nornir can query via NAPALM.

## Device Table

| Device | Hostname/IP     | Platform | Group  | OS Version              |
| ------ | --------------- | -------- | ------ | ----------------------- |
| R1     | 192.168.121.101 | IOSv     | CORE   | IOS 15.9(3)M3           |
| D1     | 192.168.121.102 | IOSvL2   | DIST   | IOS 15.2 (Experimental) |
| D2     | 192.168.121.103 | IOSvL2   | DIST   | IOS 15.2 (Experimental) |
| S1     | 192.168.121.104 | IOL-L2   | ACCESS | IOS-XE 17.15.1          |
| S2     | 192.168.121.105 | IOL-L2   | ACCESS | IOS-XE 17.15.1          |

## Credential Matrix

| Group  | Username | Password | Devices |
| ------ | -------- | -------- | ------- |
| CORE   | vagrant  | vagrant  | R1      |
| DIST   | vagrant  | vagrant  | D1, D2  |
| ACCESS | admin    | admin    | S1, S2  |

## File Structure

```
inventory/
├── hosts.yaml      # Device definitions (hostname, groups)
├── groups.yaml     # Group definitions (credentials, platform)
├── defaults.yaml   # Connection options (NAPALM driver settings)
└── README.md       # This file
```

## How Nornir Resolves Values

Nornir uses an inheritance model: `defaults.yaml` → `groups.yaml` → `hosts.yaml`.

- **defaults.yaml** — connection options (napalm driver, SSH settings)
- **groups.yaml** — credentials and platform per group
- **hosts.yaml** — per-device overrides (only hostname and group membership here)

## Adding a New Device

1. Add the device to `hosts.yaml`:

```yaml
NEW_DEVICE:
  hostname: 192.168.121.XX
  groups:
    - CORE # or DIST / ACCESS
```

2. If the device needs a new group, add it to `groups.yaml`:

```yaml
NEW_GROUP:
  hosts:
    NEW_DEVICE:
  username: your_user
  password: your_pass
  platform: ios # or eos, nxos, junos, etc.
```

3. If using a new NAPALM driver, update `defaults.yaml` connection options.

4. Reload inventory in Claude Code: `/mcp` → `nornir_reload_inventory`

## Supported NAPALM Getters (IOS)

`arp_table`, `bgp_config`, `bgp_neighbors`, `bgp_neighbors_detail`, `config`, `environment`, `facts`, `firewall_policies`, `interfaces`, `interfaces_counters`, `interfaces_ip`, `ipv6_neighbors_table`, `lldp_neighbors`, `lldp_neighbors_detail`, `mac_address_table`, `network_instances`, `ntp_peers`, `ntp_servers`, `ntp_stats`, `optics`, `probes_config`, `probes_results`, `route_to`, `snmp_information`, `users`, `vlans`

## Notes

- All devices are on the `192.168.121.0/24` management network (Vagrant/libvirt)
- IOSv devices (R1, D1, D2) may timeout under heavy parallel NAPALM calls — use sequential queries
- The `connection_options.napalm.platform` in `defaults.yaml` is the NAPALM driver, not the Nornir host platform (that's set in `groups.yaml`)
