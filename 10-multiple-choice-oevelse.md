# 10 – Multiple-Choice Øvelse (30 spørgsmål fra dine egne configs)

Test dig selv før eksamen! Alle spørgsmål tager direkte udgangspunkt i konfigurationerne fra dine `assXXnetwork`-repos. Klik på **Vis facit og forklaring** efter du har valgt dit svar.

---

### Spørgsmål 1 (`ass34` / `ass35` / `ass52` – DNAT & Policy)
En SRX har følgende Destination NAT:
```junos
security nat destination pool webserver-pool address 192.168.4.10/32;
security nat destination rule-set from-lablan from zone untrust;
security nat destination rule-set from-lablan rule to-web match destination-address 10.56.16.85/32;
security nat destination rule-set from-lablan rule to-web match destination-port 80;
security nat destination rule-set from-lablan rule to-web then destination-nat pool webserver-pool;
```
Hvilken `destination-address` skal bruges i `security policy` fra `untrust` til `DMZ` for at tillade HTTP-trafikken?
- A) `10.56.16.85/32`
- B) `192.168.4.10/32`
- C) `192.168.4.1/24`
- D) `0.0.0.0/0` (kun dette virker ved DNAT)

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B (`192.168.4.10/32`)**
På Juniper SRX udføres Destination NAT **før** route lookup og security policy lookup. Når pakken rammer security policy'en, er destination-IP'en allerede omskrevet til den private DMZ-IP `192.168.4.10`.
</details>

---

### Spørgsmål 2 (`ass52` – Host-inbound-traffic override)
Se denne konfiguration fra `ass52network`:
```junos
security-zone lab {
    host-inbound-traffic {
        system-services { ping; }
        protocols { ospf; }
    }
    interfaces {
        ge-0/0/1.0;
        ge-0/0/4.0 {
            host-inbound-traffic {
                system-services { dhcp; }
            }
        }
    }
}
```
Kan en PC på `ge-0/0/4.0` pinge routerens interface-IP på `ge-0/0/4.0`?
- A) Ja, fordi `ping` er tilladt på zone-niveau i `lab`
- B) Nej, fordi `host-inbound-traffic system-services` på interface-niveau erstatter zone-niveauets `system-services`
- C) Ja, fordi `ping` altid er tilladt som standard på SRX
- D) Nej, fordi `ospf` blokerer ICMP

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
I Junos overskriver (erstatter) interface-specifik `host-inbound-traffic` zone-niveauet for den angivne kategori (`system-services`). Derfor skal man skrive både `dhcp;` og `ping;` på `ge-0/0/4.0` (som du også gjorde i `ass52network/r2-config.json`).
</details>

---

### Spørgsmål 3 (`ass37` / `ass38` – Stateful Firewall)
En SRX har to policies mellem `OPERATIONS` og `SALES`:
- `from-zone OPERATIONS to-zone SALES`: `Allow_HTTP_and_ping` (`then permit`)
- `from-zone SALES to-zone OPERATIONS`: `deny_all_traffic` (`then deny`)

Hvad sker der, når en PC i `OPERATIONS` pinger en PC i `SALES`?
- A) Ping fejler, fordi svaret blokeres af `deny_all_traffic`
- B) Ping lykkes, fordi ICMP Echo Reply er returtrafik på en etableret stateful session
- C) Ping lykkes kun, hvis `host-inbound-traffic system-services ping` er slået til
- D) Ping fejler, fordi `junos-http` og `junos-icmp-ping` ikke kan kombineres

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
SRX er en stateful firewall. Når den udgående ICMP Echo Request tillades af `Allow_HTTP_and_ping`, oprettes en session, så returtrafikken (Echo Reply) automatisk tillades tilbage.
</details>

---

### Spørgsmål 4 (`ass38` – 802.1Q Trunk vs. Access)
Hvad gør linjen `port-mode trunk;` under `interfaces ge-0/0/5 unit 0 family ethernet-switching`?
- A) Giver interfacet dobbelt båndbredde (Link Aggregation)
- B) Gør porten til en L3 router-port med IPv4
- C) Tillader porten at bære flere VLANs samtidigt med 802.1Q VLAN-tags på frames
- D) Tillader porten at sende untagged trafik til flere VLANs samtidigt

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: C**
En trunk-port bruger IEEE 802.1Q til at indsætte et 4-byte VLAN-tag i Ethernet-framen, så trafik fra flere VLANs (`SALES` og `OPERATIONS`) kan transporteres over samme kabel mellem to switches.
</details>

---

### Spørgsmål 5 (`ass36` vs. `ass37` – `l3-interface`)
Hvad er effekten af at tilføje `l3-interface vlan.10;` under `vlans SALES`?
- A) Det aktiverer 802.1Q tagging på alle porte i `SALES`
- B) Det forbinder L2-VLAN'et `SALES` internt til det logiske L3-gateway-interface `vlan.10` (RVI), så inter-VLAN routing er mulig
- C) Det tildeler VLAN 10 til `ge-0/0/10`
- D) Det deaktiverer L2 switching i `SALES`

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`l3-interface vlan.10` kobler L2 broadcast-domænet `SALES` til L3-interfacet `interfaces vlan unit 10` (`192.168.10.1/24`), som fungerer som default gateway for hosts i VLAN 10.
</details>

---

### Spørgsmål 6 (`ass49` / `ass50` / `ass52` – OSPF `interface-type p2p`)
Hvad gør `interface-type p2p;` under `protocols ospf area 0.0.0.0 interface ge-0/0/1.0`?
- A) Krypterer OSPF-pakkerne mellem de to routere
- B) Behandler linket som point-to-point, så OSPF springer DR/BDR-valget over
- C) Gør interfacet passivt, så der ikke sendes OSPF Hello-pakker
- D) Sætter OSPF-metric automatisk til 1

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
På et Ethernet-link vælger OSPF som standard DR/BDR (`broadcast`). Med `interface-type p2p` undgås DR/BDR-valg og Network LSAs, hvilket giver hurtigere konvergens på et 2-router link.
</details>

---

### Spørgsmål 7 (`ass50` – OSPF `priority`)
På et OSPF `broadcast`-netværk har R2 `priority 200` (router-id `2.2.2.2`) og R4 har standard priority (router-id `4.4.4.4`). Hvem bliver valgt som DR (Designated Router)?
- A) R4, fordi den har højest `router-id`
- B) R2, fordi `priority 200` er højere end standardværdien `128`
- C) Ingen, fordi begge er i `area 0.0.0.0`
- D) R2, men kun hvis `interface-type p2p` er sat

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Ved DR-valg vinder **højeste `priority`** først (standard er 128). `router-id` bruges kun som tie-breaker, hvis prioriteten er ens.
</details>

---

### Spørgsmål 8 (`ass49` / `ass52` – OSPF Metric)
R2 har direkte link til R5 med `metric 10`. R2 har også en vej via R4, hvor R2→R4 har `metric 1` og R4→R5 har `metric 5`. Hvilken vej vælger OSPF fra R2 til R5?
- A) Direkte R2→R5, fordi det er færrest hops (1 hop)
- B) Via R4 (R2→R4→R5), fordi samlet cost (1 + 5 = 6) er lavere end 10
- C) Load-balancing 50/50 mellem begge veje
- D) Den vej der har højest `router-id`

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
OSPF bruger ikke hop count (det er RIP), men **laveste samlede metric (cost)**. Da `6 < 10`, vælges vejen via R4.
</details>

---

### Spørgsmål 9 (`ass51` / `ass52` – `policy-options` & `ospf export`)
Hvad gør følgende konfiguration i `ass52network`?
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
}
protocols ospf export my-default-static-route-to-internet;
```
- A) Tillader brugere på internettet at nå routeren via OSPF
- B) Redistribuerer kun den statiske default route (`0.0.0.0/0`) ind i OSPF, så andre OSPF-routere lærer vejen til internettet
- C) Opretter en firewall-regel der tillader al trafik til `0.0.0.0/0`
- D) Eksporterer alle statiske ruter på routeren ind i OSPF

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`protocol static` + `route-filter 0.0.0.0/0 exact` matcher udelukkende den statiske default route, og `protocols ospf export` redistribuerer den ind i OSPF til de øvrige routere.
</details>

---

### Spørgsmål 10 (`ass49` – OSPF på SRX firewall)
To SRX-routere er forbundet direkte på `ge-0/0/1.0` med IP'er i samme `/28`-subnet og begge i `protocols ospf area 0.0.0.0`. Alligevel viser `show ospf neighbor` intet. Hvad er den mest sandsynlige manglende linje i config?
- A) `set system services ospf`
- B) `set security zones security-zone lab host-inbound-traffic protocols ospf`
- C) `set security policies from-zone lab to-zone lab policy ospf then permit`
- D) `set interfaces ge-0/0/1 unit 0 family ospf`

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
OSPF Hello-pakker er adresseret til selve routeren (kontrolplanet), så de styres af `host-inbound-traffic protocols ospf` – ikke af `security policies`.
</details>

---

### Spørgsmål 11 (`ass9` / `ass18` – DHCP Relay)
Hvorfor er `forwarding-options dhcp-relay` nødvendig, når DHCP-serveren (R1) ligger på et andet subnet end klienterne (PC3 på R2)?
- A) Fordi DHCP bruger TCP, som ikke kan routes
- B) Fordi klientens DHCP Discover er en broadcast (`255.255.255.255`), og routere videresender ikke broadcasts mellem subnets
- C) Fordi SRX ikke understøtter mere end 2 DHCP-pools
- D) Fordi DHCP kun virker med Source NAT

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Routere stopper broadcasts ved subnet-grænsen. DHCP-relay opfanger broadcasten, udfylder `giaddr` og sender den som unicast til den centrale DHCP-server.
</details>

---

### Spørgsmål 12 (`ass9` / `ass18` – Central DHCP-server)
I `ass18network` har den centrale DHCP-server (R1) følgende under `system services dhcp-local-server`:
```junos
group Net1 { interface ge-0/0/1.0; }
group Relay { interface ge-0/0/3.0; }
```
Hvorfor er `ge-0/0/3.0` (transit-linket til R2) tilføjet under `dhcp-local-server`, selvom der ikke sidder nogen PC'er direkte på `ge-0/0/3.0`?
- A) For at give R2 en dynamisk IP-adresse
- B) Fordi de relayerede DHCP-forespørgsler fra R2 ankommer på `ge-0/0/3.0`, og DHCP-serveren kun behandler pakker på interfaces listet under `dhcp-local-server`
- C) For at aktivere DHCP-relay på R1
- D) Det er en fejl og gør ingen forskel

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Med Extended DHCP lytter `dhcp-local-server` kun på de interfaces, der eksplicit er tilføjet i en `group`. Uden `ge-0/0/3.0` ignorerer R1 de relayerede DHCP-pakker fra R2.
</details>

---

### Spørgsmål 13 (`ass18` – Statisk DHCP-reservation)
Hvad gør følgende blok i `access address-assignment pool Net1`?
```junos
host PC1-Reserved {
    hardware-address 00:0c:29:ca:cc:09;
    ip-address 192.168.1.50;
}
```
- A) Blokerer MAC-adressen `00:0c:29:ca:cc:09` fra at få en IP-adresse
- B) Reserverer fast IP `192.168.1.50` til enheden med MAC-adressen `00:0c:29:ca:cc:09`
- C) Opretter en statisk ARP-post i routingtabellen
- D) Ændrer routerens egen MAC-adresse

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Det er en statisk DHCP-reservation: Når enheden med den givne MAC-adresse sender DHCP Discover, får den altid `192.168.1.50`.
</details>

---

### Spørgsmål 14 (`ass45` – Route-based VPN `st0.1`)
Hvad er funktionen af denne statiske rute i `ass45network`?
```junos
routing-options static route 10.2.2.0/24 next-hop st0.1;
```
- A) Sender trafik til `10.2.2.0/24` ud på det offentlige internet uden kryptering
- B) Sender trafik til `10.2.2.0/24` ind i IPsec-tunnelen, der er bundet til `st0.1`
- C) Tildeler IP-adressen `10.2.2.0/24` til `st0.1`
- D) Opretter en IKE Phase 1 gateway

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
I en route-based VPN afgør routingtabellen, hvilken trafik der skal gennem VPN-tunnelen, ved at pege `next-hop` direkte på tunnel-interfacet `st0.1`.
</details>

---

### Spørgsmål 15 (`ass45` – IPsec `protocol esp`)
Hvad giver `protocol esp;` i en IPsec Phase 2 proposal?
- A) Kun integritetstjek (ingen kryptering)
- B) Både kryptering (fortrolighed) og integritet/autentificering af brugerdata
- C) Kun nøgleudveksling (Diffie-Hellman)
- D) Komprimering af Ethernet-frames

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
ESP (*Encapsulating Security Payload*, IP protokol 50) krypterer selve payloaden og beskytter integriteten med fx `hmac-sha-256-128`.
</details>

---

### Spørgsmål 16 (`ass45` – Perfect Forward Secrecy)
Hvad opnår man med `perfect-forward-secrecy { keys group14; }` i IPsec Phase 2?
- A) Hurtigere VPN-hastighed ved at genbruge Phase 1-nøglen
- B) At hver ny Phase 2-nøgle genereres med en ny uafhængig Diffie-Hellman-udveksling, så tidligere trafik ikke kan dekrypteres hvis en nøgle lækkes
- C) At tunnelen etableres automatisk uden brugertrafik
- D) At pre-shared-key krypteres i config-filen

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
PFS sikrer uafhængige sessionsnøgler ved hver Phase 2 rekey.
</details>

---

### Spørgsmål 17 (`ass45` – `establish-tunnels immediately`)
Hvad gør `establish-tunnels immediately;` under `security ipsec vpn vpn-to-peer`?
- A) Tvinger tunnelen til at forhandle og komme op med det samme, uden at vente på datatrafik
- B) Genstarter routeren efter commit
- C) Springer IKE Phase 1 over
- D) Tillader trafik i klartekst indtil tunnelen er oppe

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: A**
Standardværdien er `on-traffic` (venter på første pakke mod `st0.1`). `immediately` starter forhandlingen straks.
</details>

---

### Spørgsmål 18 (`ass45` – `host-inbound-traffic ike`)
Hvorfor skal `ike` være tilladt under `host-inbound-traffic system-services` på WAN-interfacet (`ge-0/0/3.0`) i `ass45network`?
- A) For at kryptere brugerpakkerne på LAN-siden
- B) For at SRX'ens firewall ikke dropper indgående IKE-forhandlingspakker (UDP 500) til routeren selv
- C) For at tillade `st0.1` at få en IP-adresse via DHCP
- D) For at tillade DNS-opslag til VPN-peer

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
IKE-pakker er adresseret direkte til WAN-interfacets IP på SRX'en og skal derfor tillades i `host-inbound-traffic`.
</details>

---

### Spørgsmål 19 (`ass35` – Firewall Filter `then accept`)
Se dette stateless firewall filter fra `ass35network`:
```junos
firewall {
    filter my-port-filter {
        term myMirrorTerm {
            from { source-address 0.0.0.0/0; }
            then {
                count mirror-count;
                port-mirror;
                accept;
            }
        }
    }
}
```
Hvad sker der, hvis linjen `accept;` slettes?
- A) Ingenting – `port-mirror` accepterer automatisk pakkerne
- B) Al trafik ind/ud af DMZ-interfacet droppes på grund af filterets implicitte `discard`
- C) Kun pakker fra `0.0.0.0` droppes
- D) Tælleren `mirror-count` nulstilles

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`port-mirror` og `count` er *non-terminating actions*. Uden `accept;` falder pakken videre til filterets implicitte `discard` og bliver droppet!
</details>

---

### Spørgsmål 20 (`ass35` – Port Mirroring `input rate 1`)
Hvad betyder `forwarding-options port-mirroring input rate 1;`?
- A) Maksimalt 1 Mbps spejles
- B) Kun 1 pakke i sekundet spejles
- C) Hver 1. pakke (100 % af matchende pakker) kopieres til monitor-interfacet
- D) Kun trafik fra `unit 1` spejles

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: C**
`rate 1` er sampling-ratioen 1:1, dvs. 100 % af de matchende pakker spejles.
</details>

---

### Spørgsmål 21 (`ass35` – Snort `flags:S` og `detection_filter`)
Se Snort-reglen fra `ass35network`:
```snort
alert tcp any any -> 192.168.4.10 80 ( msg:"ASS35 TCP SYN flood mod WebServer"; flags:S; detection_filter:track by_dst, count 100, seconds 1; classtype:attempted-dos; sid:1000035; rev:1; )
```
Hvornår udløser denne regel en alarm?
- A) Ved hver eneste HTTP-forbindelse til `192.168.4.10:80`
- B) Når der ses mere end 100 TCP-pakker med kun SYN-flaget sat mod `192.168.4.10:80` inden for 1 sekund
- C) Når `192.168.4.10` sender mere end 100 pakker ud på port 80
- D) Kun når en enkelt source-IP sender mere end 100 SYN-pakker på 1 sekund

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`flags:S` matcher kun rene SYN-pakker, og `detection_filter:track by_dst, count 100, seconds 1` tæller samlet pr. destination (`192.168.4.10`) uanset hvor mange forskellige source-IP'er der angriber.
</details>

---

### Spørgsmål 22 (`ass35` – Snort IP efter DNAT)
Hvorfor står der `192.168.4.10` (og ikke `10.56.16.85`) i Snort-reglen ovenfor?
- A) Fordi Snort ikke understøtter `10.x.x.x`-adresser
- B) Fordi port mirroring sidder på DMZ-interfacet (`ge-0/0/2.0`), hvor pakkerne allerede er oversat af Destination NAT til `192.168.4.10`
- C) Fordi Source NAT har ændret destination-IP'en
- D) Fordi `10.56.16.85` er en broadcast-adresse

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
DNAT omskriver destination-IP'en fra `10.56.16.85` til `192.168.4.10` før pakken sendes ud på `ge-0/0/2.0`, hvor mirror-filteret kopierer den.
</details>

---

### Spørgsmål 23 (`ass34` / `ass52` – Source NAT `interface`)
Hvad gør `then { source-nat { interface; } }` i en SNAT-regel fra `USERLAN` til `untrust`?
- A) Blokerer trafikken på udgangsinterfacet
- B) Oversætter pakkens source-IP til udgangsinterfacets egen IP-adresse og bruger PAT (port-oversættelse), så mange hosts kan dele én IP
- C) Oversætter destination-IP til udgangsinterfacets IP
- D) Tildeler en ny DHCP-adresse til klienten

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`source-nat interface` laver PAT/NAT overload ved hjælp af egress-interfacets IP-adresse.
</details>

---

### Spørgsmål 24 (`ass54` – `applications` og `default-policy`)
Hvad gør denne konfiguration i `ass54network`?
```junos
applications {
    application my-http-8000 {
        application-protocol http;
        protocol tcp;
        destination-port 8000;
    }
}
```
- A) Starter en lokal webserver på SRX-routeren på port 8000
- B) Definerer en brugerdefineret applikation for TCP port 8000, så den kan matches i `security policies`
- C) Omdirigerer port 80 til port 8000
- D) Blokerer port 8000 globalt

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`applications` definerer blot protokol/port-mønstre, som kan refereres i `security policies` (`match application my-http-8000`).
</details>

---

### Spørgsmål 25 (`ass51` / `ass52` – Loopback `lo0.0`)
Hvorfor bruger man et loopback-interface (`lo0.0` med `192.168.100.1/32`) i OSPF i `ass51`/`ass52`?
- A) Fordi OSPF ikke kan køre uden `lo0.0`
- B) Fordi `lo0.0` er et virtuelt interface, der aldrig går ned selvom et fysisk link fejler, og giver en stabil `/32` management/test-adresse i OSPF
- C) For at spejle trafik tilbage til afsenderen
- D) For at forbinde to VLANs på Layer 2

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Loopback-interfacet forbliver `up`, så længe routeren kører, og annonceres som en `/32`-rute i OSPF.
</details>

---

### Spørgsmål 26 (`ass50` – OSPF `traceoptions`)
Hvad gør `traceoptions` under `protocols ospf` i `ass50network`?
```junos
traceoptions {
    file ospf-trace-selective.log size 1m files 5 world-readable;
    flag hello;
    flag lsa-update;
    flag error;
}
```
- A) Filtrerer og dropper OSPF Hello-pakker med fejl
- B) Logger OSPF Hello-, LSA-update- og fejl-hændelser til en roterende logfil til fejlfinding
- C) Sender OSPF-ruter til en syslog-server
- D) Sætter maks 5 OSPF-naboer

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`traceoptions` påvirker ikke routing-trafikken, men gemmer detaljerede debug-logs i `/var/log/ospf-trace-selective.log` (maks 5 filer á 1 MB).
</details>

---

### Spørgsmål 27 (`ass36` – Ren L2 VLAN-isolation)
På en SRX i `ass36network` er `ge-0/0/6` i VLAN `SALES` (id 10) og `ge-0/0/3` i VLAN `OPERATIONS` (id 20). Der er ingen `l3-interface` konfigureret. Kan PC1 (`ge-0/0/6`) nå PC2 (`ge-0/0/3`)?
- A) Ja, hvis de får IP-adresser i samme subnet
- B) Nej, fordi de er i to forskellige Layer 2 broadcast-domæner (VLANs) uden nogen Layer 3-router imellem sig
- C) Ja, fordi de sidder i samme fysiske SRX
- D) Kun via ARP

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
VLANs adskiller Layer 2 fuldstændigt. Uden et L3-interface (`l3-interface vlan.X`) eller ekstern router kan trafik aldrig krydse mellem VLAN 10 og VLAN 20.
</details>

---

### Spørgsmål 28 (`ass29` / `ass52` – Intra-zone policy)
I `ass52network` ligger både `ge-0/0/1.0`, `ge-0/0/2.0` og `ge-0/0/4.0` i `security-zone lab`. Hvorfor er der oprettet en policy `from-zone lab to-zone lab` med `then permit`?
- A) Det er unødvendigt, for trafik inden for samme zone er altid tilladt
- B) Fordi SRX som standard blokerer intra-zone trafik mellem interfaces i samme zone, indtil en policy tillader den
- C) For at tillade OSPF Hello-pakker
- D) For at aktivere Source NAT mellem interfaces i `lab`

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
På Juniper SRX gælder implicit deny også for **intra-zone** trafik (`from-zone X to-zone X`).
</details>

---

### Spørgsmål 29 (`ass6` / `ass18` / `ass42` – SSH adgang)
En administrator kører `set system services ssh` og `commit`, men kan stadig ikke SSH'e til `ge-0/0/5.0` (som ligger i zonen `untrust`). Hvad mangler?
- A) `set system root-authentication ssh`
- B) `set security zones security-zone untrust interfaces ge-0/0/5.0 host-inbound-traffic system-services ssh`
- C) `set security policies from-zone untrust to-zone untrust policy ssh then permit`
- D) `set interfaces ge-0/0/5 unit 0 family ssh`

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
`system services ssh` starter SSH-tjenesten på routeren, men `host-inbound-traffic system-services ssh` skal også åbne for indgående SSH til routerens kontrolplan på det pågældende interface/zone.
</details>

---

### Spørgsmål 30 (`ass6` / `ass9` / `ass34` – Default Route & Longest Prefix Match)
En router har to ruter i routingtabellen:
- `0.0.0.0/0 next-hop 10.56.16.1`
- `192.168.10.0/24 next-hop 10.10.10.11`

Hvor sendes en pakke med destination `192.168.10.42`?
- A) Til `10.56.16.1`, fordi `0.0.0.0/0` matcher alle adresser
- B) Til `10.10.10.11`, på grund af **Longest Prefix Match** (`/24` er mere specifik end `/0`)
- C) Den droppes på grund af rute-konflikt
- D) Den sendes til begge next-hops samtidigt

<details>
<summary>Vis facit og forklaring</summary>

**Korrekt svar: B**
Routere vælger altid den rute med længste matchende prefix (flest bit i subnetmasken). `/24` vinder altid over `/0`.
</details>
