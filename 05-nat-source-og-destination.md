# 05 – NAT: Source NAT (SNAT / PAT) og Destination NAT (DNAT)

> **Brugt i:** Source NAT i `ass6`, `ass9`, `ass32`, `ass34`, `ass35`, `ass42`, `ass52`. Destination NAT i `ass34`, `ass35`, `ass52`.

---

## 1. Source NAT (`security nat source`) – "Ud på internettet"

Eksempel fra `ass34`, `ass35` og `ass52`:

```junos
security {
    nat {
        source {
            rule-set soho {
                from zone [ USERLAN DMZ ];
                to zone untrust;
                rule hide-inside {
                    match {
                        source-address 0.0.0.0/0;
                        destination-address 0.0.0.0/0;
                    }
                    then {
                        source-nat {
                            interface;
                        }
                    }
                }
            }
        }
    }
}
```

### Hvad gør hver linje i praksis?
- **`rule-set soho`**: Grupperer en eller flere SNAT-regler efter trafikretning.
- **`from zone [ USERLAN DMZ ];` + `to zone untrust;`**: Regelsættet udløses **kun** for pakker der går fra zonen `USERLAN` eller `DMZ` ud gennem et interface i zonen `untrust`.
- **`match { source-address 0.0.0.0/0; destination-address 0.0.0.0/0; }`**: Matcher al IPv4-trafik uanset kilde- og destinations-IP. (`destination-address` kan udelades som i `ass34`/`ass35`, hvilket svarer til `0.0.0.0/0`).
- **`then { source-nat { interface; } }`**:
  - Oversætter pakkens **source IP** (fx `192.168.3.100`) til **egress-interfacets egen IP-adresse** i `untrust`-zonen (fx `10.56.16.85`), og oversætter samtidig **source port** (**PAT / Port Address Translation / NAT overload**).
  - På den måde kan mange private hosts dele én enkelt offentlig/LabLan IP-adresse.
  - Når svarpakken kommer retur til `10.56.16.85:<oversat port>`, husker SRX'en sessionen og oversætter automatisk destinations-IP/port tilbage til `192.168.3.100:<original port>`.

> **Hvorfor skal man bruge SNAT i `ass34`/`ass52`?**
> Fordi private RFC1918-adresser (`192.168.x.x`) ikke er routbare på internettet / LabLan (eller upstream-routeren kender ikke ruten tilbage til `192.168.3.0/24`). Med `source-nat interface` ser upstream-netværket kun SRX'ens egen WAN-IP (`10.56.16.85`).

---

## 2. Destination NAT (`security nat destination`) – "Port Forwarding til DMZ WebServer"

Eksempel fra `ass34`, `ass35` og `ass52`:

```junos
security {
    nat {
        destination {
            pool webserver-pool {
                address 192.168.4.10/32;
            }
            rule-set from-lablan {
                from zone untrust;
                rule to-web {
                    match {
                        destination-address 10.56.16.85/32;
                        destination-port 80;
                    }
                    then {
                        destination-nat {
                            pool webserver-pool;
                        }
                    }
                }
            }
        }
    }
}
```

### Hvad gør hver linje i praksis?
- **`pool webserver-pool { address 192.168.4.10/32; }`**: Definerer den **interne/private IP-adresse** (`/32` = én specifik server), som trafikken skal oversættes til.
- **`from zone untrust;`**: Bemærk at Destination NAT **kun** har `from zone` (aldrig `to zone`)!
  - **HVORFOR?** Fordi DNAT udføres **FØR route lookup** i SRX! På det tidspunkt ved SRX endnu ikke, hvilken udgående zone pakken ender i – den kender kun den indgående zone (`untrust`).
- **`match { destination-address 10.56.16.85/32; destination-port 80; }`**: Matcher indgående pakker fra `untrust`, der er sendt til routerens offentlige/LabLan IP `10.56.16.85` på TCP/UDP port `80` (HTTP).
- **`then { destination-nat { pool webserver-pool; } }`**: Omskriver pakkens **destination IP** fra `10.56.16.85` til `192.168.4.10` (port 80 bevares, da poolen ikke ændrer port).

---

## 3. DEN VIGTIGSTE EKSAMENSREGEL OM NAT PÅ SRX: Rækkefølgen!

På en Juniper SRX sker tingene i denne **faste rækkefølge** for den første pakke i en session:

```
Indgående pakke (untrust)
   │
   ▼
1. Static NAT / Destination NAT (DNAT)   ← Dst IP ændres fra 10.56.16.85 til 192.168.4.10!
   │
   ▼
2. Route Lookup                          ← SRX slår 192.168.4.10 op → egress interface ge-0/0/2.0 (zone DMZ)
   │
   ▼
3. Security Policy Lookup                ← Evaluerer "from-zone untrust to-zone DMZ" med Dst IP = 192.168.4.10!
   │
   ▼
4. Source NAT (SNAT)                     ← Src IP oversættes (hvis matchende SNAT-regel findes)
   │
   ▼
Udgående pakke (DMZ / untrust)
```

### To kæmpe konsekvenser af denne rækkefølge (som næsten ALTID spørges om til eksamen!):

1. **Hvilken IP skal stå i `security policy` ved Destination NAT?**
   - **Den PRIVATE (oversatte / post-NAT) IP-adresse** (`192.168.4.10/32`, dvs. `webserver` i `DMZ address-book`), **IKKE** den offentlige WAN-IP (`10.56.16.85`)!
   - Fordi DNAT allerede har omskrevet destination-IP'en, *før* security policy tjekkes.
2. **Hvilken IP skal stå i `security policy` ved Source NAT?**
   - **Den PRIVATE (oprindelige / pre-NAT) source-IP**, fordi SNAT først sker *efter* security policy er tjekket!
3. **Bonus fra `ass35` (Snort IDS):**
   - Fordi port mirroring sidder på `ge-0/0/2.0` (DMZ-siden, efter DNAT), ser Snort destination-IP'en `192.168.4.10` – **ikke** `10.56.16.85`!

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"En administrator har konfigureret Destination NAT fra `10.56.16.85:80` til `192.168.4.10`, og har oprettet en security policy `from-zone untrust to-zone DMZ` med `destination-address 10.56.16.85/32`. Trafikken blokeres. Hvorfor?"*
  → Fordi DNAT sker **før** security policy! Policy'en skal matche den oversatte adresse `192.168.4.10/32` (`webserver`).
- **Fælde 2:** *"Hvorfor har `security nat destination rule-set` kun `from zone` og ikke `to zone`?"*
  → Fordi DNAT udføres **før** route lookup, så udgående zone kendes ikke endnu.
- **Fælde 3:** *"Hvad gør `source-nat interface` i modsætning til en SNAT-pool?"*
  → Den bruger direkte udgangsinterfacets egen IP-adresse med PAT (port overload), uden behov for en separat NAT-pool af ekstra IP-adresser.
- **Fælde 4:** *"Skal man også lave en `security policy` fra `untrust` til `USERLAN` for at svarpakker fra internettet kan nå tilbage til PC'erne efter SNAT?"*
  → **Nej!** SRX er stateful – returtrafik for en etableret session matches automatisk i session-tabellen og får omvendt NAT automatisk.
