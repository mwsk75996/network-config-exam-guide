# Netværks-eksamensguide: Junos SRX & Config-reference

Dette repository samler og forklarer **alle** konfigurationer, stanzas og parametre, som du har arbejdet med på tværs af alle dine 19 `assXXnetwork`-repositories. Guiden er bygget specifikt til en **multiple-choice eksamen**, hvor du skal kunne læse en konfiguration og forklare, hvad de enkelte linjer gør i praksis – og hvad der går galt, hvis noget mangler eller ændres.

---

## Indholdsfortegnelse

| Fil | Emne | Primære `assXXnetwork` repos |
| :--- | :--- | :--- |
| [01-system-og-grundlaeggende.md](01-system-og-grundlaeggende.md) | `version`, `system` (`host-name`, `root-authentication`, `ssh`), formater (JSON / `{}` / `set`) | Alle Junos-repos |
| [02-interfaces-og-vlans.md](02-interfaces-og-vlans.md) | `interfaces` (`ge-0/0/x`, `lo0`, `st0`, `vlan.X`), L2 switching, 802.1Q trunk vs. access, `vlans` & RVI | `ass34`, `ass35`, `ass36`, `ass37`, `ass38`, `ass45`, `ass51`, `ass52` |
| [03-routing-static-og-ospf.md](03-routing-static-og-ospf.md) | `routing-options` (statiske ruter, default route, `router-id`), `protocols ospf`, `policy-options` (redistribution) | `ass6`–`ass54`, især `ass49`, `ass50`, `ass51`, `ass52` |
| [04-security-zones-policies-apps.md](04-security-zones-policies-apps.md) | `security zones`, `host-inbound-traffic`, `address-book`, `applications`, `security policies`, `default-policy` | Alle SRX-repos, især `ass32`, `ass37`, `ass38`, `ass52`, `ass54` |
| [05-nat-source-og-destination.md](05-nat-source-og-destination.md) | Source NAT (`interface`/PAT) og Destination NAT (`pool`), samt rækkefølge i SRX packet flow | `ass6`, `ass9`, `ass32`, `ass34`, `ass35`, `ass42`, `ass52` |
| [06-dhcp-og-dhcp-relay.md](06-dhcp-og-dhcp-relay.md) | Legacy DHCP vs. Extended DHCP (`dhcp-local-server` + `access`), reservationer, DHCP Relay & BOOTP helper | `ass9`, `ass18`, `ass34`, `ass35`, `ass42`, `ass45`, `ass52` |
| [07-site-to-site-ipsec-vpn.md](07-site-to-site-ipsec-vpn.md) | Route-based IPsec VPN: IKE Phase 1 (`proposal`, `policy`, `gateway`), IPsec Phase 2 (`esp`, `PFS`, `st0.1`) | `ass45` |
| [08-port-mirroring-firewall-og-snort.md](08-port-mirroring-firewall-og-snort.md) | Stateless `firewall filter`, `forwarding-options port-mirroring`, og Snort IDS-regler (`local.rules`) | `ass35` |
| [09-srx-packet-flow-og-faelder.md](09-srx-packet-flow-og-faelder.md) | Den komplette SRX-pakkerækkefølge + de 15 klassiske multiple-choice fælder | Tværgående |
| [10-multiple-choice-oevelse.md](10-multiple-choice-oevelse.md) | **30 eksamensspørgsmål** med skjult facit og forklaring baseret på dine egne configs | Alle emner |

---

## Hurtigt opslag: Top-level stanzas i Junos

| Top-level stanza | Hvad styrer den i praksis? |
| :--- | :--- |
| `version` | Hvilken Junos OS-version konfigurationen er gemt/eksporteret fra (ren metadata). |
| `system` | Enhedens eget styresystem: hostname, root-password, og lokale services på selve routeren (`ssh`, `dhcp`, `dhcp-local-server`). |
| `interfaces` | Fysiske og logiske porte: IP-adresser (`family inet`), L2 switch-porte (`family ethernet-switching`), VLAN-gateways (`vlan.10`), loopback (`lo0.0`) og VPN-tunneler (`st0.1`). |
| `vlans` | Opretter L2 broadcast-domæner med et `vlan-id` (802.1Q tag) og binder evt. en L3 gateway (`l3-interface vlan.X`) til VLAN'et. |
| `routing-options` | Statiske ruter (`static route ... next-hop ...`) og routerens unikke OSPF/routing ID (`router-id`). |
| `protocols` | Dynamiske routingprotokoller – i dine opgaver `ospf` (area, interfaces, `p2p`, `metric`, `priority`, `export`, `traceoptions`). |
| `policy-options` | Routing-politikker (`policy-statement`) der styrer hvilke ruter der må eksporteres/redistribueres ind i fx OSPF. **Ikke** firewall-regler! |
| `security` | Alt stateful firewall- og VPN-logik på SRX: `zones`, `policies`, `nat`, `address-book`, `ike`, `ipsec`. |
| `applications` | Egne brugerdefinerede protokol/port-definitioner (fx `my-http-8000` på TCP port 8000), som kan bruges i `security policies`. |
| `access` | Adresse-pools (`address-assignment pool`) til Extended DHCP-serveren (netværk, IP-range, gateway, DNS, lease time, MAC-reservationer). |
| `forwarding-options` | Speciel pakkehåndtering i forwarding-planet: `dhcp-relay`, `helpers bootp`, og `port-mirroring`. |
| `firewall` | **Stateless** pakkefiltre (ACL'er) bundet direkte til et interface (`input`/`output`) – i dine opgaver brugt til at tælle og spejle trafik (`port-mirror`). |

---

## Oversigt over alle 19 gennemgåede `assXXnetwork` repositories

| Repo | Emne / Opgave | Vigtigste config-elementer |
| :--- | :--- | :--- |
| `ass6network` | vSRX basis-setup med IoT-net | `interfaces`, `static route 0.0.0.0/0`, `applications` (`my-http-8000`), `security zones`, `policies`, `source-nat interface` |
| `ass9network` | Lokal DHCP, central DHCP-server & DHCP-relay | `dhcp-local-server`, `access address-assignment`, `forwarding-options dhcp-relay`, `source-nat`, `static route` |
| `ass18network` | DHCP (legacy + extended), MAC-reservation & BOOTP relay | `system services dhcp`, `dhcp-local-server`, `access ... host hardware-address`, `forwarding-options helpers bootp` |
| `ass29network` | Multi-router statisk routing (4 SRX/vSRX) | `interfaces`, flere `static routes`, `security zones`, `policies` |
| `ass32network` | SRX SOHO / LabLan uplink med SNAT | `interfaces`, `static route 0.0.0.0/0`, `source-nat interface`, `policies` (`permit`/`deny`) |
| `ass34network` | SRX L2 Switch, SOHO & Destination NAT | `family ethernet-switching`, `vlans`, fysisk patch mellem L3 og L2, `source-nat`, `destination-nat pool`, `address-book` |
| `ass35network` | Port Mirroring & Snort IDS (bygger på ass34) | `firewall filter`, `then port-mirror`, `then count`, `forwarding-options port-mirroring`, Snort `local.rules` (SYN flood) |
| `ass36network` | SRX Simple VLANs (kun L2) | `vlans` (`SALES` 10, `OPERATIONS` 20), `family ethernet-switching`, ren L2-isolation uden routing |
| `ass37network` | Inter-VLAN Routing (RVI / `l3-interface`) | `interfaces vlan unit 10/20`, `vlans ... l3-interface vlan.10`, `security zones` på `vlan.10`/`vlan.20`, policies mellem VLANs |
| `ass38network` | 802.1Q Tagged Trunk mellem 2 SRX-enheder | `port-mode trunk`, `vlan members [ SALES OPERATIONS ]`, tagged trunk vs. untagged access-porte |
| `ass42network` | Extended DHCP + DNS + SNAT + statisk routing | `dhcp-local-server`, `access address-assignment`, `host-inbound-traffic` (`dns`, `dhcp`, `ssh`, `ping`, `traceroute`) |
| `ass45network` | Site-to-Site IPsec VPN (Route-based) | `st0.1`, `security ike` (Phase 1), `security ipsec` (Phase 2, ESP, PFS), `static route ... next-hop st0.1`, `host-inbound-traffic ike` |
| `ass49network` | OSPF på 3 fysiske SRX-routere | `router-id`, `protocols ospf area 0.0.0.0`, `interface-type p2p`, `metric`, `host-inbound-traffic protocols ospf` |
| `ass50network` | OSPF DR/BDR valg & debugging (`traceoptions`) | `priority 200`, `traceoptions` (`file`, `flag hello`, `lsa-update`, `error`) |
| `ass51network` | OSPF med flere subnets & Loopback | `interfaces lo0.0 /32`, `policy-options policy-statement`, `route-filter ... exact`, `ospf export` |
| `ass52network` | OSPF Route Redistribution + DMZ + DNAT/SNAT + DHCP | Redistribution af `static` default route (`0.0.0.0/0 exact`) og direkte LAN/DMZ-ruter ind i OSPF + fuld SOHO/DMZ-opsætning |
| `ass54network` | vSRX med og uden Multizone + Global Address Book | `security address-book global`, `default-policy deny-all`, custom `applications`, granulære host-til-host policies |
| `ass57network` | MQTT Publisher & Subscriber (C++) | Ikke Junos-config – C++ MQTT-klienter (broker IP, topics, connection order) |
| `ass62network` | ESP32 WiFi + MQTT temperatursensor (C++) | Ikke Junos-config – ESP32 WiFi (`WiFi.mode(WIFI_STA)`, DHCP vs statisk IP) + MQTT port 1883 |
