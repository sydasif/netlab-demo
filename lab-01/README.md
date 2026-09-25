# OSPF Lab

## Purpose

`lab-01` is a containerlab-based OSPF lab built with netlab. It uses six Cisco IOL routers and one passive Linux host and can render a styled Graphviz topology image.

The topology source is `topology.yml`. Generated files must not be treated as hand-authored source.

## Design

- Provider: `clab`
- Router image: `asifsyd/cisco_iol:17.12.01`
- OSPF process: 1, IPv4
- Area 0: authenticated Ethernet backbone
- Area 1: NSSA
- Area 3: totally stubby (`inter_area: false`)
- Expected Area 0 election: `R02` is DR and `R14` is BDR
- Graph renderer: native Graphviz

The authentication value is configured in `topology.yml` and is intentionally not repeated in this document.

## Devices

| Device | Platform | OSPF role                      |
| ------ | -------- | ------------------------------ |
| `R01`  | IOL      | Area 0 backbone and Area 3 ABR |
| `R02`  | IOL      | Area 0 backbone and Area 1 ABR |
| `R10`  | IOL      | Area 1 NSSA router             |
| `R12`  | IOL      | Area 3 totally stubby router   |
| `R13`  | IOL      | Area 3 totally stubby router   |
| `R14`  | IOL      | Area 0 backbone and Area 3 ABR |
| `L01`  | Linux    | Passive host                   |

Important segments:

- The authenticated Area 0 LAN connects `R01`, `R14`, and `R02`.
- The Area 1 NSSA link connects `R02` to `R10`.
- Area 3 point-to-point links connect `R12` and `R13` to both `R01` and `R14`.
- The passive stub LAN connects `R10` to `L01`.

## Files

Hand-authored files:

```text
topology.yml    netlab topology and graph styling
README.md       lab design and operating notes
```

Typical generated files:

```text
clab.yml
config/
hosts.yml
netlab.snapshot.pickle
ospf.dot
ospf.png
```

Do not hand-edit generated files. Do not overwrite a reference image named `lab.png` if one is added later.

## Generate and validate

Run these commands from the lab directory:

```bash
cd /home/zulu/Projects/netlab-demo/lab-01

netlab create
netlab initial --output config --clean
containerlab deploy -t clab.yml --dry-run
```

These commands generate deployment artifacts without starting the lab.

## Start and inspect the lab

Starting or restarting the lab is an explicit operational action:

```bash
cd /home/zulu/Projects/netlab-demo/lab-01

netlab up
netlab status
```

After startup, verify:

- All intended OSPF adjacencies are `FULL`.
- Authentication is enabled on the Area 0 interfaces without exposing its value.
- `R02` is DR and `R14` is BDR.
- `R12` receives the expected inter-area default route.
- `L01` remains passive.

## Generate the topology image

Always render from the source topology explicitly so graph metadata cannot be read from an older snapshot:

```bash
cd /home/zulu/Projects/netlab-demo/lab-01

netlab graph \
  --topology topology.yml \
  --type topology \
  --title "OSPF Lab" \
  ospf.png
```

Inspect both `ospf.dot` and `ospf.png` after rendering. Durable graph styling remains under `defaults.outputs.graph.styles` in `topology.yml`.

## Stop the lab

```bash
cd /home/zulu/Projects/netlab-demo/lab-01

netlab down
```

Avoid `netlab down --cleanup` unless generated runtime artifacts should also be removed.

## Caveats

- This topology has no `validate:` section, so `netlab validate` cannot provide validation-test coverage.
- `netlab graph` can prefer `netlab.snapshot.pickle`; pass `--topology topology.yml` after source or graph-style changes.
- The Linux host image comes from the installed netlab device defaults rather than an image pinned in this topology.
- This lab is not included in the repository's shared Nornir inventory, which currently serves `lab-02`.
