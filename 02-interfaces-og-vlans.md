# 02 – Interfaces, L2 Switching og VLANs (802.1Q)

> **Brugt i:** `ass6`–`ass54` (L3 interfaces), `ass34`–`ass38` (L2 switching, VLANs, RVI & 802.1Q trunk), `ass45` (`st0`), `ass51`–`ass52` (`lo0`).

---

## 1. Opbygningen af et Junos-interface

```junos
interfaces {
    ge-0/0/1 {
        description "USERLAN routed gateway - patch to ge-0/0/10";
        unit 0 {
            family inet {
                address 192.168.3.1/24;
            }
        }
    }
}
```

| Linje | Hvad gør den i praksis? |
| :--- | :--- |
| `ge-0/0/1` | Fysisk Gigabit Ethernet-port (FPC 0 / PIC 0 / Port 1). |
| `description "..."` | Ren menneskelig dokumentation. Har **ingen** funktionel effekt på pakker. |
| `unit 0` | Logisk sub-interface (`ge-0/0/1.0`). I Junos bindes IP-adresser, zoner og protokoller altid til det **logiske** interface (`ge-0/0/1.0`), ikke kun det fysiske (`ge-0/0/1`). |
| `family inet` | Aktiverer **IPv4** (Layer 3 routing) på det logiske interface. |
| `address 192.168.3.1/24` | Tildeler routeren IP-adressen `192.168.3.1` med subnetmaske `/24` (`255.255.255.0`). Dette skaber automatisk en **direkte rute (`Direct`)** til `192.168.3.0/24` i routingtabellen. |

> **Vigtigt:** Et interface kan **ikke** have både `family inet` og `family ethernet-switching` på samme `unit 0` samtidigt. Enten er porten en **Layer 3 router-port** (`family inet`), eller også er den en **Layer 2 switch-port** (`family ethernet-switching`).

---

## 2. Specielle logiske interfaces i dine repos

### Loopback-interface (`lo0.0`) – brugt i `ass51`, `ass52`
```junos
interfaces {
    lo0 {
        unit 0 {
            family inet {
                address 192.168.100.1/32;
            }
        }
    }
}
```
- **Hvad gør den i praksis?** Et rent virtuelt L3-interface inde i routeren, som **aldrig går ned**, så længe selve routeren er tændt (uanset om fysiske kabler falder ud).
- **Hvorfor `/32`?** Det repræsenterer én enkelt host-IP (selve routeren), ikke et helt LAN med andre enheder.
- **Brug i OSPF:** Når `lo0.0` tilføjes under `protocols ospf area 0.0.0.0`, annonceres loopback-adressen som en stabil `/32`-rute i OSPF, så man altid kan nå/pinge routeren uanset hvilken fysisk vej pakkerne tager.

### Secure Tunnel interface (`st0.1`) – brugt i `ass45`
```junos
interfaces {
    st0 {
        description "Secure tunnel to SRX-B";
        unit 1 {
            family inet;
        }
    }
}
```
- **Hvad gør den i praksis?** Et virtuelt point-to-point tunnel-interface til **route-based IPsec VPN**.
- **Hvorfor står `family inet;` tom (uden IP-adresse)?** Det kaldes et **unnumbered** interface. Det behøver ingen egen IP-adresse for at route IPv4-pakker ind i VPN-tunnelen – det er nok at `family inet` er aktiveret, og at en statisk rute peger `next-hop st0.1`. (Se [07-site-to-site-ipsec-vpn.md](07-site-to-site-ipsec-vpn.md)).

### VLAN L3-interface / RVI (`vlan.10`, `vlan.20`) – brugt i `ass37`, `ass38`
```junos
interfaces {
    vlan {
        unit 10 {
            description "SALES gateway";
            family inet {
                address 192.168.10.1/24;
            }
        }
        unit 20 {
            description "OPERATIONS gateway";
            family inet {
                address 192.168.20.1/24;
            }
        }
    }
}
```
- **Hvad gør den i praksis?** Opretter et **Routed VLAN Interface (RVI)** (svarende til SVI på Cisco). Det fungerer som **default gateway** (Layer 3) for alle hosts i det pågældende VLAN, uden at man skal bruge et fysisk patch-kabel mellem en switch-port og en router-port.
- **Vigtig binding:** `vlan.10` virker **først**, når den kobles til VLAN'et under `vlans`-stanzaen med `l3-interface vlan.10;`.

---

## 3. Layer 2 Switching: Access-porte vs. 802.1Q Trunk-porte

### Access-port (standard i Junos) – brugt i `ass34`, `ass35`, `ass36`, `ass37`, `ass38`
```junos
interfaces {
    ge-0/0/6 {
        description "SALES - PC1";
        unit 0 {
            family ethernet-switching {
                vlan {
                    members SALES;
                }
            }
        }
    }
}
```
- **Hvad gør den i praksis?**
  - `family ethernet-switching`: Sætter porten til at fungere som en **Layer 2 switch-port** (lærer MAC-adresser og videresender Ethernet-frames inden for samme VLAN).
  - Når `port-mode` **ikke** er angivet, er porten automatisk i **`access`-mode**.
  - **Untagged trafik:** En access-port sender og modtager almindelige **untagged** Ethernet-frames (uden 802.1Q VLAN-header) mod slut-enheder som PC'er og servere. Når en frame kommer ind på `ge-0/0/6`, tilhører den internt VLAN `SALES`, og når den sendes ud af porten igen, fjernes eventuelle VLAN-tags.
  - En access-port kan kun være medlem af **ét** VLAN.

### Trunk-port (802.1Q) – brugt i `ass38`
```junos
interfaces {
    ge-0/0/5 {
        description "TRUNK 802.1Q to SRX-C2 ge-0/0/5";
        unit 0 {
            family ethernet-switching {
                port-mode trunk;
                vlan {
                    members [ SALES OPERATIONS ];
                }
            }
        }
    }
}
```
- **Hvad gør den i praksis?**
  - `port-mode trunk;`: Skifter porten fra access til **IEEE 802.1Q trunk**.
  - **Tagged trafik:** Porten kan bære trafik fra **flere VLANs samtidigt** over det samme fysiske kabel (typisk mellem to switches/SRX'er). Hver Ethernet-frame får indsat et **4-byte 802.1Q-tag** med `vlan-id` (fx 10 for SALES, 20 for OPERATIONS), så den modtagende switch ved, hvilket VLAN framen hører til.
  - `vlan members [ SALES OPERATIONS ];`: Fungerer som en tilladelsesliste – **kun** frames fra VLAN `SALES` (10) og `OPERATIONS` (20) må krydse trunk-linket.

> **Wireshark-observation fra `ass38network`:**
> - På en **access-port** (`ge-0/0/7`) ser Wireshark **ingen** 802.1Q-header (untagged).
> - På en **trunk-port** (`ge-0/0/2`) ser Wireshark et `802.1Q Virtual LAN` felt med `ID: 10` eller `ID: 20`.

---

## 4. `vlans`-stanzaen

### Ren L2-VLAN (`ass34`, `ass35`, `ass36`)
```junos
vlans {
    SALES {
        vlan-id 10;
    }
    OPERATIONS {
        vlan-id 20;
    }
}
```
- **Hvad gør den i praksis?** Opretter to adskilte **Layer 2 broadcast-domæner** (VLAN ID 10 og 20).
- Hosts i `SALES` kan kommunikere direkte på L2 (ARP, ping osv.), men **kan ikke nå** hosts i `OPERATIONS`, fordi der ikke er nogen L3-router imellem dem. Broadcasts (som ARP og DHCP Discover) fra `SALES` stopper ved VLAN-grænsen og når aldrig `OPERATIONS`.

### L2-VLAN med L3 Inter-VLAN Routing (`ass37`, `ass38`)
```junos
vlans {
    SALES {
        vlan-id 10;
        l3-interface vlan.10;
    }
    OPERATIONS {
        vlan-id 20;
        l3-interface vlan.20;
    }
}
```
- **Hvad gør `l3-interface vlan.10;` i praksis?** Forbinder det logiske L2-VLAN (`SALES`) internt med L3-interfacet `vlan.10` (`192.168.10.1/24`).
- Nu kan PC'er i `SALES` sende pakker til deres gateway `192.168.10.1`, hvorefter SRX'en kan route trafikken til `OPERATIONS` (`vlan.20`) – **forudsat** at `vlan.10` og `vlan.20` er lagt i `security zones` og der findes en tilladende `security policy`!

### Sammenligning: Fysisk patch (`ass34`/`ass35`) vs. RVI (`ass37`/`ass38`)
- I `ass34`/`ass35` havde `mySwitch1Vlan` og `mySwitch2Vlan` **ingen** `l3-interface`. I stedet forbandt man et fysisk patch-kabel fra en L2 switch-port (`ge-0/0/6`) til en L3 router-port (`ge-0/0/2`), så L3-porten kunne agere gateway.
- I `ass37`/`ass38` fjernes det fysiske patch-kabel ved at bruge `l3-interface vlan.10` (RVI), som kobler L2-switchen og L3-routeren internt inde i SRX'en.

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"To PC'er sidder på samme fysiske SRX-switch, én i VLAN 10 og én i VLAN 20, uden `l3-interface`. Kan de pinge hinanden, hvis de har IP'er i samme subnet?"*
  → **Nej!** VLANs adskiller Layer 2 broadcast-domæner fuldstændigt. ARP når aldrig frem mellem VLAN 10 og VLAN 20.
- **Fælde 2:** *"Hvad sker der, hvis `port-mode trunk` udelades på en port mellem to switches med `vlan members [ SALES OPERATIONS ]`?"*
  → Porten er som standard en **access-port**, som kun må have **ét** VLAN (commit fejler, eller trafikken tagges ikke med 802.1Q).
- **Fælde 3:** *"Hvorfor har `st0.1` kun `family inet;` og ingen IP-adresse i `ass45`?"*
  → Fordi det er et unnumbered tunnel-interface til route-based IPsec VPN – ruter peger direkte på `next-hop st0.1`.
- **Fælde 4:** *"Skal L2 switch-porte (`family ethernet-switching`) tilføjes til en `security-zone`?"*
  → **Nej!** Security zones på SRX gælder for **Layer 3-interfaces** (`ge-0/0/x.0` med `family inet`, `vlan.10`, `st0.1`). L2-trafik inden for samme VLAN krydser ikke L3-firewallen.
