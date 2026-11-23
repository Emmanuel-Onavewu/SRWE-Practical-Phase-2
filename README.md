# Lab 2: SRWE Practice Part 2

## Objective
Prototype a small enterprise network with static routing (including
floating backups), switch security, DHCP, and a wireless LAN controller.
Only four devices were configurable: Central, Branch-101, S1-1 and WLC-10.


## Design summary
| Area | Design |
|------|--------|
| Routing | IPv4 and IPv6 default routes, Ethernet as primary, serial as floating backup (AD 10) |
| Branch-101 | Subinterface for VLAN 10, DHCP pool `WLAN-hosts`, DHCP client toward the ISP |
| S1-1 | VLANs 10/999, port security, DHCP snooping, DAI, PortFast and BPDU guard, secured unused ports |
| WLC-10 | WLAN `SSID-10` with WPA2 and 802.1X, RADIUS, DHCP scope, SNMP trap receiver |

## Routing logic (floating static routes)
A floating static route is a backup with a higher administrative
distance (AD) than the primary. Here the primary uses the Ethernet link
and the backup uses the serial link with AD 10, so the backup stays out
of the routing table until the primary fails.

Branch-101, as configured:
```
ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/0/1
ip route 0.0.0.0 0.0.0.0 Serial0/1/0 10
ipv6 route ::/0 2001:DB8:ACAD:C::1
ipv6 route ::/0 2001:DB8:ACAD:B::1 10
```
`show ip route static` listed only the Ethernet default (`S*`), and
`show ipv6 route` listed only the primary `::/0` via
`2001:DB8:ACAD:C::1`. The AD 10 backups were configured but hidden, as
designed. IPv4 routes use exit interfaces and IPv6 routes use next-hop
addresses, as the lab required.

## What I configured
- **Branch-101:** subinterface `G0/0/0.10` (router-on-a-stick), DHCP
  pool `WLAN-hosts` with exclusions, `G0/0/1` as a DHCP client, floating defaults
- **Central:** IPv4 and IPv6 defaults with floating backups, route to the
  branch WLAN network, and host routes to Server-O1
- **S1-1:** port security (max 4, restrict, sticky, 10-minute aging),
  DHCP snooping with rate limit 5, DAI, PortFast, BPDU guard, unused
  ports in VLAN 999 and shut down
- **WLC-10:** `WLAN 10` interface, RADIUS server, WLAN `SSID-10`
  (WPA2, 802.1X, FlexConnect local switching), DHCP scope `Wired_Admin`,
  SNMP trap receiver
- **Laptop:** wireless profile `work net` with enterprise authentication

## Verification
| Test | Result |
|------|--------|
| Branch-101 `ping 203.0.113.25` | 100% (5/5) on the second run; the first run lost 2 packets while ARP resolved |
| Branch-101 `ping 2001:db8:cafe::25` | 100% (5/5) |
| Laptop sees `SSID-10` | Yes, channel 1, 79% signal |
| Laptop `ping 192.168.10.1` | 4/4 replies |

The Central and S1-1 verification outputs and the final Check Results
score were not captured, so they are not listed here.

## Troubleshooting story
**Wrong device, right config.** I pasted the Central routing block onto
Branch-101. The routes pointed at the wrong interfaces (`g0/0/2`,
`s0/1/1`) and the wrong next hops. I removed them with `no ip route ...`
and `no ipv6 route ...` and confirmed the clean state with
`show running-config | include route`. One `no` command also removed a
backup route that was correct, so I re-added
`ip route 0.0.0.0 0.0.0.0 s0/1/0 10` and checked again.

**A command that doesn't exist.** `show ipv6 route static` returned
"Invalid input" because this Packet Tracer version treated `static` as a
hostname and attempted a DNS lookup. `show ipv6 route` worked.

**IPv6 ping from the wireless laptop failed.** The WLAN is IPv4-only: the
DHCP pool gives IPv4 only and VLAN 10 has no IPv6 address in the
addressing table, so the laptop has no IPv6 path. IPv6 was verified from
Branch-101 instead.

## Lessons learned
1. Check the CLI prompt before pasting. Each device needs its own block.
2. Read `show running-config | include route` after any undo. A `no`
   command can remove more than you intended.
3. Floating static routes stay hidden until the primary path fails.
4. Test IPv6 only from hosts that have IPv6 addresses.
5. WLC pages use separate fields. Paste values one at a time.

## Files
- [`configs/`](configs/): device configurations (secrets redacted)
- [`verification/`](verification/): show command output
- [`screenshots/`](screenshots/): evidence from Packet Tracer
