# 03 – Routing: Statiske ruter, OSPF og Route Redistribution

> **Brugt i:** Statiske ruter i `ass6`, `ass9`, `ass18`, `ass29`, `ass32`, `ass34`, `ass35`, `ass42`, `ass45`, `ass52`, `ass54`. OSPF & redistribution i `ass49`, `ass50`, `ass51`, `ass52`.

---

## 1. `routing-options`: Statiske ruter og Router-ID

```junos
routing-options {
    router-id 2.2.2.2;
    static {
        route 0.0.0.0/0 next-hop 10.56.16.1;
        route 192.168.10.0/24 next-hop 10.10.10.11;
        route 10.2.2.0/24 next-hop st0.1;
    }
}
```

| Linje | Hvad gør den i praksis? |
| :--- | :--- |
| `router-id 2.2.2.2;` | Giver routeren en **unik 32-bit identifikator** (skrevet som en IPv4-adresse, men den behøver ikke være tildelt noget interface). Bruges af OSPF til at identificere routere i LSAs og som tie-breaker ved DR/BDR-valg. |
| `route 0.0.0.0/0 next-hop 10.56.16.1;` | **Default route** (gateway of last resort). Matcher alle destinationer, som routeren ikke har en mere specifik rute til (longest prefix match), og sender dem til `10.56.16.1` (typisk internet-gateway). |
| `route 192.168.10.0/24 next-hop 10.10.10.11;` | Sender al trafik til subnet `192.168.10.0/24` videre til naboreouteren `10.10.10.11`. |
| `route 10.2.2.0/24 next-hop st0.1;` | Sender trafik til det fjerne VPN-netværk `10.2.2.0/24` direkte ind i IPsec-tunnelen `st0.1` (`ass45`). |

> **Huskeregel – Longest Prefix Match:** Hvis routingtabellen har både `0.0.0.0/0` og `192.168.10.0/24`, vælger routeren altid den **mest specifikke rute** (højeste prefix-længde `/24` slår `/0`).

---

## 2. `protocols ospf`: Dynamisk Link-State Routing

Eksempel fra `ass49`, `ass50`, `ass51` og `ass52`:

```junos
protocols {
    ospf {
        export [ my-route-to-USERLAN1 my-route-to-DMZ my-default-static-route-to-internet ];
        traceoptions {
            file ospf-trace-selective.log size 1m files 5 world-readable;
            flag hello;
            flag lsa-update;
            flag error;
        }
        area 0.0.0.0 {
            interface lo0.0;
            interface ge-0/0/2.0 {
                metric 1;
            }
            interface ge-0/0/1.0 {
                interface-type p2p;
                metric 10;
                priority 200;
            }
        }
    }
}
```

### Hver enkelt linje i `protocols ospf` forklaret:

- **`area 0.0.0.0`** (Backbone Area / Area 0):
  - Alle routere i samme OSPF-område deler den samme **LSDB** (Link-State Database) og beregner korteste vej med **Dijkstra's SPF-algoritme**.
  - Alle OSPF-netværk skal som udgangspunkt have et Area `0.0.0.0`.
- **`interface ge-0/0/2.0`** (uden `interface-type`):
  - På Ethernet-interfaces er OSPF's standard netværkstype **`broadcast`**.
  - På et broadcast-netværk vælger OSPF en **DR** (Designated Router) og **BDR** (Backup Designated Router) for at begrænse mængden af LSA-opdateringer.
  - Routeren sender OSPF Hello-pakker (multicast `224.0.0.5`) ud af interfacet og inkluderer interfacet's subnet i sine Router LSAs.
- **`interface-type p2p;`**:
  - Tvinger OSPF til at behandle Ethernet-linket som **Point-to-Point** (kun 2 routere på linket).
  - **Konsekvens i praksis:** Der foretages **intet DR/BDR-valg** på linket! Naboskabet går hurtigere i `Full`-state, og der genereres ingen Network LSA (Type 2).
- **`metric 10;`**:
  - Sætter OSPF-**cost** (omkostningen) for at sende trafik ud af dette interface.
  - OSPF vælger altid ruten med **laveste samlede metric (cost)** fra kilde til destination.
  - Eksempel fra `ass49`/`ass52`: Hvis R2→R4 har `metric 1` og R4→R5 har `metric 5` (samlet cost = 6), mens R2→R5 direkte har `metric 10`, vælger OSPF den længere vej via R4 (6 < 10)!
- **`priority 200;`** (`ass50`):
  - Sætter routerens **DR-prioritet** (0–255, standard er **128**) på et broadcast-interface.
  - **Højeste priority vinder DR-valget.** Hvis to routere har samme priority, vinder den med **højeste `router-id`**.
  - Hvis `priority 0` sættes, kan routeren **aldrig** blive DR eller BDR.
  - **Vigtigt:** `priority` har **ingen effekt** på et `interface-type p2p`-link, fordi der ikke er noget DR/BDR-valg på p2p!
- **`interface lo0.0;`** (`ass51`, `ass52`):
  - Tilføjer loopback-interfacet til OSPF Area 0, så dets `/32`-adresse annonceres til alle andre OSPF-routere. (Loopback-interfaces er automatisk passive i Junos OSPF – de danner ikke naboskaber, de annoncerer bare adressen).
- **`export [ ... ];`** (`ass51`, `ass52`):
  - Knytter en eller flere `policy-statement`s til OSPF for at **redistribuere eksterne ruter** (fx statiske ruter eller direkte tilsluttede LAN/DMZ-subnets, der ikke selv kører OSPF) ind i OSPF.
  - Ruter der eksporteres med `export` annonceres som **OSPF External (Type 5 LSA)**, og routeren fungerer som **ASBR** (Autonomous System Boundary Router).
- **`traceoptions`** (`ass50`):
  - Logger OSPF-hændelser til en fil (`/var/log/ospf-trace-selective.log`) til fejlfinding.
  - `size 1m files 5`: Maks 1 MB pr. fil, roterer op til 5 filer.
  - `world-readable`: Tillader også ikke-root brugere at læse logfilen.
  - `flag hello;`, `flag lsa-update;`, `flag error;`: Logger kun Hello-pakker, LSA-opdateringer og fejl (i stedet for alt).

> **KRITISK SRX-KRAV FOR OSPF:** På en Juniper SRX blokeres OSPF-pakker som standard af firewallen! Du **skal** tillade `ospf` i security-zonen for de interfaces, der kører OSPF:
> ```junos
> set security zones security-zone lab host-inbound-traffic protocols ospf
> ```
> Uden denne linje kommer OSPF-naboskabet **aldrig** op (forbliver tomt/Init).

---

## 3. `policy-options`: Route Redistribution til OSPF

Eksempel fra `ass51` og `ass52`:

```junos
policy-options {
    policy-statement my-default-static-route-to-internet {
        term accept-my-default-route {
            from {
                protocol static;
                route-filter 0.0.0.0/0 exact;
            }
            then accept;
        }
    }
    policy-statement my-route-to-USERLAN1 {
        term accept-USERLAN1 {
            from {
                route-filter 192.168.13.0/24 exact;
            }
            then accept;
        }
    }
}
```

### Hvad gør hver del i praksis?
- **`policy-statement <navn>`**: En routing-politik (ikke en firewall-regel!). Den gør **ingenting** før den anvendes, fx under `protocols ospf export <navn>;`.
- **`term <navn>`**: En navngiven regel med en `from`-betingelse (match) og en `then`-handling (`accept` eller `reject`).
- **`from protocol static;`**: Matcher kun ruter i routingtabellen, der stammer fra `routing-options static`. (Udelades `protocol`, matches enhver aktiv rute, fx `direct`-ruter som `192.168.13.0/24`).
- **`route-filter 192.168.13.0/24 exact;`**:
  - Matcher **kun** præcis præfikset `192.168.13.0/24` – ikke `/25`-undernet eller større `/16`-supernet.
  - For default route matcher `0.0.0.0/0 exact` **kun** selve default-ruten og ikke alle andre ruter.
- **`then accept;`**: Tillader at den matchede rute eksporteres ind i OSPF.
- **Hvorfor bruger man `export` i stedet for bare at tilføje `USERLAN1` som OSPF-interface?**
  - Fordi man ikke ønsker at sende OSPF Hello-pakker ud på et bruger-LAN eller DMZ (hvor en angriber potentielt kunne injicere falske OSPF-ruter), og fordi statiske ruter (som `0.0.0.0/0`) slet ikke kan annonceres blot ved at tilføje et interface til OSPF.

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"OSPF er konfigureret på `ge-0/0/1.0` på både R2 og R5 med matchende subnet og `area 0.0.0.0`, men `show ospf neighbor` er tom. Hvad mangler typisk på en SRX?"*
  → `host-inbound-traffic protocols ospf` på security-zonen for `ge-0/0/1.0` (eller `interface-type` mismatch, hvor den ene side er `p2p` og den anden `broadcast`!).
- **Fælde 2:** *"Hvad sker der hvis R2 har `interface-type p2p` og R5 har standard (`broadcast`) på samme link?"*
  → De kan godt komme i Full på nogle systemer eller hænge, men SPF-beregningen fejler fordi den ene forventer en Network LSA (DR) og den anden en P2P-link LSA → **ruter installeres ikke korrekt**. Begge ender skal have samme `interface-type`!
- **Fælde 3:** *"Hvorfor bruger R2 en højere cost (`metric 10`) på det direkte link til R5, men lavere cost (`1` + `5` = `6`) via R4?"*
  → For at styre trafikken via R4 som primær vej og bruge det direkte link som backup. OSPF vælger altid **laveste samlede metric**.
- **Fælde 4:** *"Hvad er forskellen på `policy-options` og `security policies`?"*
  → `policy-options` styrer **routing-opdateringer** (kontrolplan – hvilke ruter der udveksles/redistribueres). `security policies` styrer **datatrafik** (dataplan – hvilke pakker der må passere gennem firewallen mellem zoner).
