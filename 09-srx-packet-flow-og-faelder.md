# 09 – SRX Packet Flow og de 15 klassiske Multiple-Choice Fælder

Denne side er dit **cheat sheet** lige før eksamen. Hvis du kan packet-flowet og de 15 fælder nedenfor, kan du gennemskue størstedelen af alle konfigurationsspørgsmål til en Juniper SRX / netværks-eksamen.

---

## 1. Juniper SRX Packet Flow (første pakke i ny session)

```
 [Indgående pakke på ingress interface]
                │
                ▼
 1. Ingress stateless firewall filter (hvis konfigureret på interface `input`)
                │
                ▼
 2. Session Lookup: Tilhører pakken en eksisterende stateful session?
    ├── JA → Fast Path! (Omvendt NAT/encap udføres automatisk → direkte til egress)
    └── NEJ → First Path (fortsæt nedenfor)
                │
                ▼
 3. Er pakken adresseret TIL selve SRX'en (routerens egen IP / multicast som OSPF / DHCP)?
    ├── JA → Tjek `host-inbound-traffic` (`system-services` / `protocols`) på zonen/interfacet!
    └── NEJ → Transit-trafik (fortsæt nedenfor)
                │
                ▼
 4. Destination NAT (DNAT)   ← Oversætter destination IP/port FØR routing og policy!
                │
                ▼
 5. Route Lookup             ← Finder next-hop (IP eller `st0.1`) og udgående interface/zone
                │
                ▼
 6. Security Policy Lookup   ← `from-zone X to-zone Y` (bruger POST-DNAT dst-IP og PRE-SNAT src-IP!)
    └── Ingen match? → `default-policy deny-all` (drop!)
                │
                ▼
 7. Source NAT (SNAT)        ← Oversætter source IP/port (fx `source-nat interface`)
                │
                ▼
 8. IPsec-kryptering (hvis egress er `st0.1`) + Egress stateless firewall filter (`output`)
                │
                ▼
 [Udgående pakke på egress interface]
```

---

## 2. De 15 klassiske Multiple-Choice Fælder

| # | Fælde / Spørgsmålstype | Det rigtige svar i praksis |
| :---: | :--- | :--- |
| **1** | Hvilken IP skal stå som `destination-address` i `security policy` når man bruger **DNAT**? | Den **private/oversatte** IP (`192.168.4.10/32`), fordi DNAT sker **før** security policy. |
| **2** | Hvorfor har `security nat destination` kun `from zone` og ikke `to zone`? | Fordi DNAT udføres **før** route lookup, så udgående zone kendes ikke endnu. |
| **3** | Hvad er forskellen på `host-inbound-traffic` og `security policies`? | `host-inbound-traffic` = trafik **til selve SRX'en** (ping til gateway, SSH, DHCP, OSPF, IKE). `security policies` = trafik **gennem** SRX'en fra ét netværk til et andet. |
| **4** | Hvad sker der, hvis du sætter `host-inbound-traffic system-services dhcp` på et interface i en zone, der har `ping` på zone-niveau? | Interface-niveau **overskriver (erstatter)** zone-niveau for `system-services`! Interfacet mister `ping`, medmindre `ping` også nævnes på interfacet. |
| **5** | Kan to interfaces i **samme** security zone tale sammen uden en policy? | **Nej!** Intra-zone trafik (`from-zone X to-zone X`) er blokeret som standard på SRX og kræver en eksplicit `permit`-policy. |
| **6** | Skal man lave en omvendt `security policy` for svarpakker (fx `SALES -> OPERATIONS`)? | **Nej!** SRX er **stateful**. Når `OPERATIONS -> SALES` tillades, tillades returtrafikken automatisk via session-tabellen. |
| **7** | `system services ssh` er sat, men SSH til routeren virker ikke. Hvad mangler? | `host-inbound-traffic system-services ssh` på det indgående interface/zone. |
| **8** | OSPF er konfigureret i `protocols ospf area 0.0.0.0`, men ingen naboer dukker op på SRX. Hvad mangler? | `host-inbound-traffic protocols ospf` på zonen (eller mismatch i subnet/area/`interface-type`). |
| **9** | Hvad gør `interface-type p2p` i OSPF? | Slår **DR/BDR-valg fra** på linket (hurtigere konvergens, ingen Network LSA). Begge ender skal matche! |
| **10** | Hvad gør `priority 200` i OSPF? | Gør routeren mere tilbøjelig til at blive **DR** på et `broadcast`-netværk (standard = 128, højest vinder, 0 = aldrig DR/BDR). Virker **ikke** på `p2p`! |
| **11** | Hvad er forskellen på `policy-options` og `security policies`? | `policy-options` styrer **routing-annoncering/redistribution** (fx `ospf export`). `security policies` styrer **firewall-tilladelser** for brugerpakker. |
| **12** | Hvad er forskellen på en **access-port** og en **trunk-port** (`port-mode trunk`)? | **Access** = ét VLAN, **untagged** frames mod PC/server. **Trunk** = flere VLANs over samme kabel, **802.1Q tagged** frames mellem switches. |
| **13** | Hvordan finder en central DHCP-server den rigtige pool ved **DHCP Relay**? | Relay-routeren skriver sin egen klient-interface IP i **`giaddr`**-feltet, som serveren matcher mod `network` i sine `access address-assignment` pools. |
| **14** | Hvad gør `perfect-forward-secrecy (PFS)` i IPsec Phase 2? | Tvinger ny Diffie-Hellman-nøgleudveksling ved rekey, så kompromittering af én nøgle ikke kan dekryptere tidligere trafik. |
| **15** | Hvad sker der hvis `then accept;` mangler i et stateless `firewall filter`? | Pakkerne rammes af filterets **implicitte discard** og droppes! |
