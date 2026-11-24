# 06 – DHCP Server (Legacy vs. Extended) og DHCP Relay

> **Brugt i:** `ass9` (lokal DHCP, central DHCP-server & DHCP relay), `ass18` (legacy DHCP, extended DHCP, MAC-reservation & BOOTP helper), `ass34`/`ass35` (legacy DHCP), `ass42`, `ass45`, `ass52` (extended DHCP).

I dine repos har du arbejdet med **to forskellige måder** at konfigurere DHCP-server på Juniper SRX (Legacy vs. Extended) og **to forskellige måder** at konfigurere DHCP-relay på (`dhcp-relay` vs. `helpers bootp`). Til en multiple-choice eksamen er det guld værd at kunne kende forskel på dem!

---

## 1. Metode A: Legacy DHCP (`system services dhcp`)

Brugt i `ass18` (R2) og `ass34`/`ass35`:

```junos
system {
    services {
        dhcp {
            pool 192.168.3.0/24 {
                address-range low 192.168.3.100 high 192.168.3.200;
                default-lease-time 86400;
                name-server {
                    8.8.8.8;
                }
                router {
                    192.168.3.1;
                }
            }
        }
    }
}
```

### Hvad gør hver linje?
- Alt ligger samlet under `system services dhcp` (ingen `access`-stanza).
- `pool 192.168.3.0/24`: Angiver hvilket subnet poolen dækker. Junos binder automatisk poolen til det interface, som har en IP i `192.168.3.0/24` (`ge-0/0/1.0` = `192.168.3.1/24`).
- `address-range low ... high ...`: Det interval af IP-adresser DHCP-serveren må uddele dynamisk (`192.168.3.100` til `.200`).
- `router 192.168.3.1`: **DHCP Option 3** – sender default gateway til klienten.
- `name-server 8.8.8.8`: **DHCP Option 6** – sender DNS-server til klienten.
- `default-lease-time 86400`: Hvor længe (i sekunder – 86400 s = 24 timer) klienten må beholde sin IP, før den skal fornyes.

---

## 2. Metode B: Extended DHCP (`dhcp-local-server` + `access`)

Brugt i `ass9`, `ass18` (R1), `ass42`, `ass45` og `ass52`. Det er Junos' moderne/anbefalede metode og består altid af **TO dele**:

### Del 1 – Aktivér DHCP-lytning på interfaces (`system services dhcp-local-server`)
```junos
system {
    services {
        dhcp-local-server {
            group USERLAN1 {
                interface ge-0/0/4.0;
            }
            group Relay {
                interface ge-0/0/3.0;
            }
        }
    }
}
```
- **Hvad gør den i praksis?** Fortæller SRX'en, **hvilke interfaces** DHCP-serveren skal lytte på efter DHCP Discover/Request-pakker.
- **MEGET VIGTIGT ved central DHCP-server (`ass9`, `ass18`):** Det interface hvor *relayed* DHCP-pakker ankommer fra en anden router (fx `ge-0/0/3.0` i `group Relay`) **SKAL** også være med under `dhcp-local-server`! Ellers ignorerer den centrale DHCP-server de relayerede pakker.

### Del 2 – Definer adresse-pools og parametre (`access address-assignment`)
```junos
access {
    address-assignment {
        pool USERLAN1 {
            family inet {
                network 192.168.13.0/24;
                range USERS {
                    low 192.168.13.10;
                    high 192.168.13.20;
                }
                dhcp-attributes {
                    maximum-lease-time 3600;
                    name-server {
                        8.8.8.8;
                    }
                    router {
                        192.168.13.1;
                    }
                }
                host PC1-Reserved {
                    hardware-address 00:0c:29:ca:cc:09;
                    ip-address 192.168.1.50;
                }
            }
        }
    }
}
```

| Linje | Hvad gør den i praksis? |
| :--- | :--- |
| `network 192.168.13.0/24;` | Subnet og maske for poolen. Bruges af serveren til at matche enten det lokale interfaces subnet eller **giaddr** (Gateway IP Address) fra en DHCP-relay! |
| `range USERS { low ...; high ...; }` | De dynamiske IP'er der kan udlejes til almindelige klienter. |
| `maximum-lease-time 3600;` | Maksimal lejetid i sekunder (3600 s = 1 time; 600 s = 10 min). |
| `router { 192.168.13.1; }` | Default gateway (DHCP Option 3) som klienten modtager. |
| `name-server { 8.8.8.8; }` | DNS-server (DHCP Option 6) som klienten modtager. |
| `host PC1-Reserved { hardware-address ...; ip-address ...; }` | **Statisk DHCP-reservation** (`ass18`): Binder en specifik **MAC-adresse** (`00:0c:29:ca:cc:09`) til altid at få den samme faste IP (`192.168.1.50`). |

> **Vigtigt:** Du kan **ikke** køre både Legacy DHCP (`system services dhcp`) og Extended DHCP (`system services dhcp-local-server`) samtidigt på samme SRX – de er gensidigt udelukkende.

---

## 3. DHCP Relay – Når DHCP-serveren ligger på et andet subnet!

**Hvorfor er DHCP Relay nødvendigt?**
Fordi en klient starter DHCP-processen (**DORA**: *Discover → Offer → Request → Acknowledge*) med en **Layer 2/3 broadcast (`255.255.255.255`)**. **Routere videresender IKKE broadcasts** på tværs af subnets!

DHCP Relay løser dette ved at:
1. Lytte efter DHCP-broadcasts på klient-interfacet (fx `ge-0/0/3.0`).
2. Indsætte klient-interfacets egen IP i feltet **`giaddr`** (Gateway IP Address) i DHCP-pakken.
3. Videresende pakken som **unicast** (UDP port 67) til den centrale DHCP-server (`10.10.10.11`).
4. Den centrale DHCP-server kigger på `giaddr` for at vælge den rigtige `access address-assignment pool` (fx `192.168.12.0/24`), og sender Offer/ACK tilbage til relay-routeren, som afleverer den til klienten.

### Moderne metode: `forwarding-options dhcp-relay` (`ass9`)
```junos
forwarding-options {
    dhcp-relay {
        server-group R1-DHCP-SERVER {
            10.10.10.11;
        }
        group RELAY-NET3-NET4 {
            active-server-group R1-DHCP-SERVER;
            interface ge-0/0/3.0;
            interface ge-0/0/4.0;
        }
    }
}
```
- `server-group R1-DHCP-SERVER`: IP-adressen på den centrale DHCP-server (`10.10.10.11`).
- `group RELAY-NET3-NET4`: Angiver `active-server-group` og hvilke **klient-vendte interfaces** (`ge-0/0/3.0`, `ge-0/0/4.0`) der skal relayeres DHCP-forespørgsler fra.

### Legacy metode: `forwarding-options helpers bootp` (`ass18`)
```junos
forwarding-options {
    helpers {
        bootp {
            server 10.10.10.10;
            interface {
                ge-0/0/1;
                ge-0/0/2;
            }
        }
    }
}
```
- Gør præcis det samme som `dhcp-relay` (DHCP bygger historisk på BOOTP, UDP 67/68), men bruger den ældre Junos-syntaks.

---

## 4. De 4 ting der SKAL være på plads for at DHCP / Relay virker på SRX

1. **`host-inbound-traffic system-services dhcp`** på klient-interfacet (og på serverens link-interface ved relay) – ellers dropper SRX firewallen DHCP-pakkerne før de når DHCP-processen!
2. **Routing mellem relay og central DHCP-server:** Begge routere skal have ruter (`static` eller `ospf`) til hinandens subnets, så den centrale server kan sende svar tilbage til `giaddr`!
3. **`dhcp-local-server`** på den centrale server skal inkludere det interface, hvor de relayerede pakker ankommer (fx `group Relay { interface ge-0/0/3.0; }`).
4. **Korrekt `network`-pool** på den centrale server, der matcher `giaddr`'s subnet.

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"Hvordan ved en central DHCP-server, hvilken IP-pool (fx Net3 vs. Net4) den skal tildele en IP fra, når begge netværk bruger samme DHCP-relay?"*
  → Relay-routeren udfylder feltet **`giaddr`** med IP-adressen på det interface, hvor DHCP Discover-broadcasten blev modtaget. Serveren matcher `giaddr` mod `network` i sine pools.
- **Fælde 2:** *"Hvad sker der med en DHCP Discover-pakke, når den rammer en router uden DHCP-server og uden DHCP-relay?"*
  → Den droppes ved routeren, fordi routere ikke videresender Layer 3 broadcasts (`255.255.255.255`).
- **Fælde 3:** *"Hvad gør `hardware-address` under `access address-assignment pool ... host`?"*
  → Laver en statisk DHCP-reservation baseret på klientens **MAC-adresse**, så enheden altid får den samme IP-adresse.
