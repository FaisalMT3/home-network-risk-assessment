# Assessment report

| | |
|---|---|
| Scope | One home network, `192.168.1.0/24`: ISP fibre gateway, four mesh nodes, one workstation, one laptop, two phones |
| Out of scope | Anything beyond the gateway's WAN port; denial-of-service testing; password guessing against the gateway (it locks after three attempts) |
| Authorisation | Given by the network owner before each pass |
| Passes | Configuration review 2026-09-26 · LAN scan 2026-10-06 · Re-test 2026-11-21 |
| Tools | nmap 7.991 (`-sS -sV -p-`, `vulners` and `vuln` scripts without DoS), nuclei 3.11.1 without DoS and fuzzing templates, curl for UPnP SOAP calls, miniupnpc |

## Method

1. **Configuration review.** Every settings page of the gateway was saved before any change, so each change could be reverted.
2. **Discovery.** ARP sweep, because several devices ignore ping.
3. **Full TCP scan with service detection** of every live host.
4. **Vulnerability scripts** against the open ports, with denial-of-service checks excluded. Consumer router firmware crashes under them, and taking the household offline is not an acceptable price for a finding.
5. **Manual verification** of anything a tool reported, before it went into the register.
6. **Risk scoring** on a 5 × 5 likelihood and impact scale, then a treatment decision per finding.

Scale used:

| Score | Likelihood | Impact |
|---|---|---|
| 1 | Needs physical access or an unlikely chain | Nuisance |
| 2 | Needs an attacker already on the LAN and a specific flaw | One device degraded |
| 3 | Any compromised device on the LAN could do it | One device or account compromised |
| 4 | Reachable from the internet | Several devices or all traffic exposed |
| 5 | Being exploited now | Lasting loss of data or money |

## Findings

### F1. DMZ exposed the workstation to the internet (High, closed)

The gateway's DMZ option was enabled and pointed at the workstation. A DMZ host receives every inbound connection that no other rule claims, so the only thing between the internet and that machine was its own host firewall. That firewall allowed a game-streaming service (TCP 47984 to 48010) and a file-sharing service (TCP 53317) from any address, which made both reachable from the internet.

- **Evidence:** gateway DMZ page, saved before the change.
- **Treatment:** DMZ disabled on 2026-09-26 and confirmed off on the gateway's settings page afterwards.
- **Lesson:** a host firewall rule written for "the LAN" silently becomes an internet rule when the perimeter changes. Rules should name the source network.

### F2. UPnP accepts unauthenticated port-forwarding requests (High, open)

The gateway runs a UPnP InternetGatewayDevice service on a high TCP port. It serves its device description and answers SOAP calls with no credentials. See [evidence/upnp-probe.txt](../evidence/upnp-probe.txt).

The same interface offers `AddPortMapping`. Any device on the LAN can ask the gateway to forward a port from the internet to any internal address.

- **How it was found:** SSDP discovery returned nothing, and the first draft of this report recorded UPnP as clean. The port sweep had listed an unknown service on port 49652. Requesting `/gatedesc.xml` from that port showed what it was. Where a discovery probe and a port disagree, the port is right.
- **Proof of impact:** during the first two passes only read-only calls were made, on the principle that a scan should not write firewall rules. On 2026-11-21, in an authorised honeypot exercise, a laptop on the LAN created 37 inbound port mappings with one unauthenticated request each. The gateway accepted them and refused only a few reserved ports (for example 22, 23, 80 and 443). All mappings were deleted when the exercise ended and the ports were checked closed from outside. Details: [honeypot-attack-analysis](https://github.com/FaisalMT3/honeypot-attack-analysis).
- **Risk:** malware on any household device can open its own path in from the internet. The gateway currently has no other mappings, so nothing is exposed today.
- **Treatment:** disable UPnP on the gateway. It stays open while the lab exercises use it, and is the first item to close afterwards.

### F3. Android Debug Bridge reachable over Wi-Fi (High, closed)

A phone had ADB listening on TCP 5555, left over from an earlier remote-debugging setup. Android asks the user to approve an unknown host key, so this was not an open shell, but port 5555 is a standard target for worms on shared networks.

- **Treatment:** switched ADB back to USB-only during the assessment and confirmed the port closed.

### F4. Unsupported DNS server on the mesh nodes (Medium, open)

Three mesh nodes answer DNS with `dnsmasq 2.78`, released in October 2017. The version matches the "DNSpooq" group of vulnerabilities fixed in 2.83, which include cache poisoning. See [evidence/dnsmasq-version-match.txt](../evidence/dnsmasq-version-match.txt).

- **What is proven:** the nodes report a version that has been out of support for years.
- **What is not proven:** that they are exploitable. Vendors often backport fixes and leave the version string alone. Confirming it would need the vendor changelog or a proof-of-concept run against a node, which was not attempted.
- **Exposure:** LAN only.
- **Treatment:** transfer. The nodes are ISP equipment and their firmware cannot be updated by the owner. Action: ask the ISP for current firmware or replacement units.

### F5. Wi-Fi accepted WPA with TKIP (Medium, closed)

Both bands were set to mixed WPA/WPA2 with TKIP and AES. TKIP has been deprecated since 2012.

- **Treatment:** changed to WPA2 with AES only on 2026-09-26. Every device reconnected, so nothing in the house needed the old mode.

### F6. Management frames are not protected (Medium, open)

The access point advertises RSN capabilities `0x000c`: protected management frames (802.11w) are neither required nor offered. Deauthentication frames are therefore accepted from anyone who forges the access point's address.

- **Proof of impact:** on 2026-11-21, 31 seconds of forged deauthentication frames from a low-cost microcontroller board, aimed at one of the owner's own laptops, disconnected it within a second. An interrupted handshake was then treated as a wrong password, and the unattended laptop stayed offline for 18 minutes until its network service was restarted. Method and logs: [wifi-intrusion-detector](https://github.com/FaisalMT3/wifi-intrusion-detector).
- **Risk:** loss of availability for any wireless device, and the opening move of an evil-twin attack. The Wi-Fi password is not exposed by this alone.
- **Treatment:** accept and monitor for now. A detector for deauthentication floods and evil twins was built and tested as a compensating control. Still to check: whether the gateway offers a protected-management-frames or WPA3 option to the owner's limited account.

### F7. Unsupported web server on the mesh admin panels (Medium, open)

The mesh nodes serve their admin pages with `lighttpd 1.4.20`, released in 2008. nmap's cross-site scripting and request-forgery scripts found nothing on the pages that load before login, and every other page redirects to the login form.

- **Treatment:** transfer, with F4. Same owner, same action.

### F8. Unidentified services (Low, open)

One mesh node listens on TCP 6100 and the gateway on TCP 27998. Neither answered any probe with a recognisable banner.

- **Treatment:** investigate. An asset inventory is not complete while a listening port has no explanation.

### F9. The ISP controls the gateway (Medium, open)

The owner has a limited account on the gateway. The ISP holds the administrator account and controls firmware and remote management. Findings F4 and F7 cannot be fixed without the ISP, and the owner cannot see what remote management is enabled.

- **Treatment:** accept. The alternative is putting an owner-controlled router behind the gateway, which is a cost decision. Recorded so the dependency is visible.

## Checks that came back clean

- No port is forwarded from the internet (after F1).
- The workstation exposes nothing to the LAN.
- The gateway's login page leaks no information before authentication and locks after three failed attempts.
- No default credentials were tried, by design (lockout risk).

## Recommendations in order

| Priority | Action | Owner |
|---|---|---|
| 1 | Disable UPnP on the gateway once the lab exercises end (F2) | Network owner |
| 2 | Ask the ISP for updated mesh firmware or replacement units (F4, F7) | ISP |
| 3 | Enable protected management frames, or WPA3, if the gateway offers it (F6) | Network owner |
| 4 | Identify the two unknown services (F8) | Network owner |
| 5 | Repeat the scan every six months and after any gateway change | Network owner |

## Commands

```bash
sudo nmap -sn -PR 192.168.1.0/24                       # ARP discovery
sudo nmap -sS -sV -p- --open -T4 <hosts>               # full TCP scan with service versions
sudo nmap -sV --script "vuln and not dos" -p <ports> <hosts>
nuclei -l targets.txt -et dos,fuzzing -rl 20           # web templates, destructive ones excluded
curl http://<gateway>:<port>/gatedesc.xml              # UPnP device description
```
