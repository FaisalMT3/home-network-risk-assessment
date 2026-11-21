# Home network risk assessment

A security assessment of a real home network, written the way a small-office assessment would be: scope and authorisation, findings with evidence, a scored risk register, treatment decisions, and a mapping to the NCA Essential Cybersecurity Controls (ECC-2:2024).

The network is an ISP fibre gateway with four mesh nodes, a workstation, a laptop and phones. The assessment ran in three passes: a configuration review of the gateway (2026-09-26), a full scan of the LAN (2026-10-06) and a re-test of the open items (2026-11-21).

**Full report:** [report/assessment.md](report/assessment.md) · **Evidence:** [evidence/](evidence/)

## Result in one table

Risk = likelihood (1 to 5) × impact (1 to 5). 12 and above is high, 6 to 11 medium, below 6 low.

| # | Finding | L | I | Risk | Treatment | Status |
|---|---|---|---|---|---|---|
| F1 | Gateway DMZ pointed at the workstation, exposing it to the internet | 4 | 4 | **16 High** | Mitigate | Closed 2026-09-26 |
| F2 | UPnP on the gateway lets any LAN device open inbound ports with no credentials | 3 | 4 | **12 High** | Mitigate | Open, proven by test |
| F3 | Android Debug Bridge listening on the Wi-Fi | 3 | 4 | **12 High** | Mitigate | Closed 2026-10-06 |
| F4 | Mesh nodes run a 2017 DNS server (dnsmasq 2.78) | 2 | 4 | 8 Medium | Transfer to the ISP | Open |
| F5 | Wi-Fi accepted the legacy WPA/TKIP mode | 2 | 3 | 6 Medium | Mitigate | Closed 2026-09-26 |
| F6 | Wi-Fi management frames are not protected (no 802.11w) | 3 | 2 | 6 Medium | Accept, monitor | Open |
| F7 | Mesh admin panels run a 2008 web server (lighttpd 1.4.20) | 2 | 3 | 6 Medium | Transfer to the ISP | Open |
| F8 | Two listening services could not be identified | 2 | 2 | 4 Low | Investigate | Open |
| F9 | The ISP holds the gateway's administrator account | 3 | 3 | 9 Medium | Accept | Open |

The worst finding (F1) came from reading configuration, not from scanning, and had been in place unnoticed: one checkbox in the gateway put a workstation directly on the internet.

```
 Impact →     1        2        3        4        5
 L 5
   4                                    F1*
   3                   F6      F9       F2 F3*
   2                   F8      F5* F7   F4
   1
                                        * closed
```

## What makes this more than a scan

- **A false negative was caught and corrected.** The first UPnP check used SSDP discovery, got no answer and recorded the gateway as clean. The port sweep had flagged an unknown service on a high port, which turned out to be the UPnP control service. Discovery said "nothing there", the port said otherwise. The report keeps the correction visible.
- **Version matches are labelled as leads.** The old `dnsmasq` matches the DNSpooq CVEs by banner only. Vendors backport fixes without changing version strings, so the finding is rated on what is proven (unsupported software), not on what the scanner printed.
- **Open risks have owners and reasons.** Six findings stay open. Two cannot be fixed by the network owner at all because the ISP controls the firmware, which is a third-party risk and is recorded as one.
- **Nothing destructive was run.** Denial-of-service scripts were excluded and the UPnP proof used read-only calls.

## Mapping to ECC-2:2024

ECC applies to organisations, not homes. It is used here as the reference a Saudi organisation would be assessed against.

| Finding | ECC control |
|---|---|
| F1 | 2-5-3-1 Isolation and segmentation of networks using firewalls and defence in depth |
| F2, F3, F8 | 2-5-3-5 Restricting and managing network services, protocols and ports |
| F4 | 2-5-3-7 Security of DNS · 2-3-3-3 Patch management |
| F5, F6 | 2-5-3-4 Wireless network security using secure authentication and encryption |
| F7 | 2-3-3-3 Patch management · 2-10-3-3 Vulnerability remediation based on classification |
| F9 | 4-1 Third-Party Cybersecurity |
| The assessment itself | 2-10-3-1 Periodic vulnerability assessment · 1-5 Cybersecurity Risk Management |

## Authorisation

Every device belongs to the household, and the network owner authorised the work in advance. No neighbouring network or third-party system was touched.
