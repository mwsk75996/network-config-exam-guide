# 04 – Security Zones, Host-Inbound-Traffic, Address Books, Applications og Policies

> **Brugt i:** Alle SRX/vSRX-repos (`ass6`–`ass54`). Særligt avancerede eksempler i `ass32`, `ass34`, `ass35`, `ass37`, `ass38`, `ass45`, `ass52`, `ass54`.

Juniper SRX er en **zone-baseret, stateful firewall**. Det betyder to fundamentale regler:
1. Et L3-interface kan **ikke** sende eller modtage trafik, før det er medlem af en **security zone**.
2. Som standard er **al trafik mellem zoner (og selv inden for samme zone!) blokeret**, indtil en `security policy` eksplicit tillader den.

---

## 1. `security zones` og `host-inbound-traffic`

```junos
security {
    zones {
        security-zone lab {
            host-inbound-traffic {
                system-services {
                    ping;
                }
                protocols {
                    ospf;
                }
            }
            interfaces {
                ge-0/0/1.0;
                ge-0/0/2.0;
                ge-0/0/4.0 {
                    host-inbound-traffic {
                        system-services {
                            dhcp;
                            ping;
                        }
                    }
                }
            }
        }
    }
}
```

### Hvad er `host-inbound-traffic`?
- **KRITISK SKELNEN TIL EKSAMEN:**
  - `host-inbound-traffic` styrer **kun trafik, der er adresseret TIL selve SRX-routeren** (fx ping til routerens egen gateway-IP, SSH til routeren, DHCP-forespørgsler til routerens DHCP-server, OSPF Hello til routeren, IKE-forhandling til routeren).
  - Trafik der bare **passerer igennem** routeren fra én PC til en anden styres **ikke** af `host-inbound-traffic` – den styres af `security policies`!

### `system-services` vs. `protocols` under `host-inbound-traffic`

| Indstilling | Kategori | Hvad tillader den i praksis? | Brugt i |
| :--- | :--- | :--- | :--- |
| `ping` | `system-services` | ICMP Echo Request til routerens egen IP. | Næsten alle repos |
| `ssh` | `system-services` | SSH-login (TCP 22) til selve routeren (kræver også `system services ssh`). | `ass6`, `ass18`, `ass42`, `ass45`, `ass52` |
| `dhcp` | `system-services` | Indgående DHCP-pakker (UDP 67/68) til routerens DHCP-server eller DHCP-relay. | `ass18`, `ass42`, `ass52` |
| `dns` | `system-services` | DNS-forespørgsler til selve routeren. | `ass42` |
| `traceroute` | `system-services` | Traceroute-probes (UDP/ICMP) mod routeren. | `ass42`, `ass54` |
| `ike` | `system-services` | IKE Phase 1/2 VPN-forhandling (UDP 500 / 4500) på WAN-interfacet. | `ass45` |
| `all` | `system-services` / `protocols` | Tillader alle system-services / alle routingprotokoller til routeren (nemt i lab, usikkert i produktion). | `ass6`, `ass9`, `ass34`, `ass35` |
| `ospf` | `protocols` | OSPF-kontrolpakker (IP protokol 89) til routeren, så naboer kan dannes. | `ass49`, `ass50`, `ass51`, `ass52` |

### MEGET VIGTIG FÆLDE: Zone-niveau vs. Interface-niveau!
Kig på eksemplet ovenfor fra `ass52network`:
- På **zone-niveau** (`security-zone lab`) er `ping` og `ospf` tilladt. Det gælder automatisk for `ge-0/0/1.0` og `ge-0/0/2.0`.
- På **interface-niveau** (`ge-0/0/4.0`) er der angivet sin egen `host-inbound-traffic`-blok med `dhcp` og `ping`.
- **I Junos OVERRIDER (erstatter) interface-niveau zone-niveau for den angivne kategori (`system-services`)!**
  - Det betyder, at `ge-0/0/4.0` får `dhcp` og `ping` (og stadig `ospf` fra zone-niveau, fordi `protocols` ikke blev overstyret på interfacet).
  - Hvis du kun skrev `dhcp` på `ge-0/0/4.0` uden `ping`, ville `ge-0/0/4.0` **miste** `ping` fra zone-niveau!

---

## 2. `address-book` (Zone-lokal vs. Global)

### Zone-lokal address-book (`ass34`, `ass35`, `ass52`)
```junos
security {
    zones {
        security-zone DMZ {
            address-book {
                address WebServer 192.168.12.55/32;
            }
        }
    }
}
```
- Definerer navnet `WebServer` = `192.168.12.55/32`, som **kun** kan bruges i policies hvor `DMZ` er den relevante zone (typisk `to-zone DMZ` som `destination-address WebServer`).

### Global address-book (`ass54`)
```junos
security {
    address-book {
        global {
            address SENSOR_LAN 192.168.10.0/24;
            address WEB_SERVER 192.168.11.2/32;
            address PC4 192.168.13.2/32;
        }
    }
}
```
- Definerer adresser **globalt**, så de kan genbruges på tværs af **alle** zoner og policies på routeren.
- **Vigtigt:** I Junos må man som hovedregel ikke blande zone-baserede og globale address-books på en måde der skaber konflikt – enten bruger man zone-lokale eller `global`.

---

## 3. `applications` (Indbyggede vs. brugerdefinerede)

### Indbyggede Junos-applikationer (`junos-*`)
I dine configs bruges bl.a.:
- `any` – matcher alle protokoller og porte.
- `junos-http` – TCP destination port 80.
- `junos-icmp-ping` – ICMP Echo Request/Reply (ping).
- `junos-udp-any` – al UDP-trafik (uanset port).

### Brugerdefineret applikation (`ass6`, `ass54`)
```junos
applications {
    application my-http-8000 {
        application-protocol http;
        protocol tcp;
        destination-port 8000;
    }
}
```
- **Hvad gør den i praksis?**
  - `protocol tcp;` + `destination-port 8000;`: Matcher TCP-trafik til port 8000 (fx en lokal Python/Node webserver).
  - `application-protocol http;`: Fortæller SRX'en (og evt. ALG/logging), at applikationsprotokollen inde i strømmen er HTTP.
  - Bruges i en policy som `application [ junos-http my-http-8000 ];`.

---

## 4. `security policies` og `default-policy`

```junos
security {
    policies {
        from-zone untrust to-zone DMZ {
            policy WebServerPolicy {
                match {
                    source-address any;
                    destination-address WebServer;
                    application junos-http;
                }
                then {
                    permit;
                }
            }
        }
        from-zone DMZ to-zone lab {
            policy deny-DMZ-to-lab {
                match {
                    source-address any;
                    destination-address any;
                    application any;
                }
                then {
                    deny;
                }
            }
        }
        default-policy {
            deny-all;
        }
    }
}
```

### Sådan fungerer `security policies` i praksis:
1. **Retningsbestemt (`from-zone X to-zone Y`):**
   - En policy fra `OPERATIONS` til `SALES` (`ass37`, `ass38`) tillader **kun** nye forbindelser initieret fra `OPERATIONS` mod `SALES`.
   - **Stateful returtrafik:** Fordi SRX er en *stateful* firewall, oprettes der automatisk en **session** i session-tabellen, når en pakke rammer `permit`. Svarpakker (returtrafik) fra `SALES` tilbage til `OPERATIONS` på den samme forbindelse **tillades automatisk**, selvom der findes en `deny_all_traffic` fra `SALES` til `OPERATIONS`!
   - Derimod kan en PC i `SALES` **ikke selv starte** en ny forbindelse til `OPERATIONS`.
2. **Intra-zone trafik (`from-zone lab to-zone lab`):**
   - Selv når to interfaces ligger i **samme** security zone (`ge-0/0/1.0` og `ge-0/0/2.0` i `lab`), kræver SRX en policy (`from-zone lab to-zone lab`), hvis trafik skal routes mellem de to interfaces! (Se `ass45`, `ass50`, `ass52`).
3. **Top-down evaluering (First Match Wins):**
   - Policies inden for samme `from-zone ... to-zone ...` læses **oppefra og ned**. Den **første** policy der matcher (`source-address`, `destination-address`, `application`) afgør pakkens skæbne – resten evalueres ikke.
   - Eksempel fra `ass54`: `allow-pc4-http-to-webserver` står før `allow-inter-lan-to-server-lan-icmp`.
4. **Implicit Deny / `default-policy deny-all`:**
   - Hvis ingen policy matcher, falder pakken igennem til `default-policy`, som på SRX som standard er `deny-all` (pakken droppes lydløst uden ICMP-fejl).

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"OPERATIONS må pinge SALES (`Allow_HTTP_and_ping`), og SALES har `deny_all_traffic` til OPERATIONS (`ass37`/`ass38`). Virker ping fra OPERATIONS til SALES?"*
  → **Ja!** Ping-svaret (ICMP Echo Reply) er returtrafik på en tilladt stateful session og passerer derfor tilbage uden at blive stoppet af `SALES -> OPERATIONS deny`.
- **Fælde 2:** *"To interfaces er begge i zonen `trust`. Kan hosts på de to interfaces tale sammen uden en policy?"*
  → **Nej!** På SRX er intra-zone trafik (`from-zone trust to-zone trust`) blokeret som standard, medmindre der findes en policy der tillader den.
- **Fælde 3:** *"En PC i `USERLAN` kan surfe på internettet, men kan ikke pinge sin egen default gateway `192.168.3.1` på SRX'en. Hvad mangler?"*
  → `host-inbound-traffic system-services ping` mangler på `USERLAN`-zonen/interfacet. Security policies gælder kun trafik *gennem* routeren, ikke *til* routeren.
