# 07 – Site-to-Site IPsec VPN (Route-Based)

> **Brugt i:** `ass45network` (`r1-config.json`, `r2-config.json`, `r3-config.json`).

I `ass45network` har I opsat en **route-based Site-to-Site IPsec VPN** mellem to SRX-routere (SRX-A og SRX-B) hen over en mellemliggende "internet"-router (SRX-C2). IPsec forhandles altid i **to faser**:
- **Phase 1 (IKE):** Opretter en krypteret, autentificeret **kontrolkanal** mellem de to VPN-gateways (UDP port 500).
- **Phase 2 (IPsec):** Forhandler nøgler og kryptering til selve **datatrafikken** (ESP – IP protokol 50) inde i tunnelen.

---

## 1. Phase 1: `security ike`

```junos
security {
    ike {
        proposal ike-phase1-proposal {
            authentication-method pre-shared-keys;
            dh-group group14;
            authentication-algorithm sha-256;
            encryption-algorithm aes-128-cbc;
            lifetime-seconds 28800;
        }
        policy ike-phase1-policy {
            mode main;
            proposals ike-phase1-proposal;
            pre-shared-key ascii-text "Ass45-VPN-key";
        }
        gateway gw-to-peer {
            ike-policy ike-phase1-policy;
            address 10.4.4.1;
            external-interface ge-0/0/3.0;
        }
    }
}
```

### Hver enkelt linje i `security ike` forklaret:

| Stanza / Parameter | Hvad gør den i praksis? |
| :--- | :--- |
| `authentication-method pre-shared-keys;` | De to routere beviser deres identitet med en **fælles hemmelig nøgle (PSK)** (i stedet for certifikater). |
| `dh-group group14;` | **Diffie-Hellman gruppe 14** (2048-bit MODP). Bruges til sikkert at udveksle en fælles krypteringsnøgle over et usikkert netværk (internettet), uden at selve nøglen nogensinde sendes i klartekst. |
| `authentication-algorithm sha-256;` | Hash-algoritme til **integritet** i Phase 1 (sikrer at IKE-beskeder ikke er ændret undervejs). |
| `encryption-algorithm aes-128-cbc;` | Symmetrisk kryptering (AES med 128-bit nøgle i CBC-mode) til at beskytte Phase 1-forhandlingen (**fortrolighed**). |
| `lifetime-seconds 28800;` | Hvor længe (28.800 sek = **8 timer**) Phase 1 SA'en (Security Association) er gyldig, før der genforhandles nøgler (rekey). |
| `mode main;` | IKEv1 **Main Mode** (6 beskeder). Beskytter peer-identiteterne i krypterede beskeder og kræver faste IP-adresser (sikrere end `aggressive` mode, som kun bruger 3 beskeder). |
| `pre-shared-key ascii-text "Ass45-VPN-key";` | Selve den delte nøgle – **skal være 100 % identisk i begge ender!** |
| `address 10.4.4.1;` | Den fjerne VPN-peers offentlige/WAN IP-adresse. |
| `external-interface ge-0/0/3.0;` | Det lokale WAN-interface, som IKE- og ESP-pakkerne sendes og modtages på. |

---

## 2. Phase 2: `security ipsec`

```junos
security {
    ipsec {
        proposal ipsec-phase2-proposal {
            protocol esp;
            authentication-algorithm hmac-sha-256-128;
            encryption-algorithm aes-128-cbc;
            lifetime-seconds 3600;
        }
        policy ipsec-phase2-policy {
            perfect-forward-secrecy {
                keys group14;
            }
            proposals ipsec-phase2-proposal;
        }
        vpn vpn-to-peer {
            bind-interface st0.1;
            ike {
                gateway gw-to-peer;
                ipsec-policy ipsec-phase2-policy;
            }
            establish-tunnels immediately;
        }
    }
}
```

### Hver enkelt linje i `security ipsec` forklaret:

| Stanza / Parameter | Hvad gør den i praksis? |
| :--- | :--- |
| `protocol esp;` | **Encapsulating Security Payload** (IP protokol 50). Giver både **kryptering (fortrolighed)** og **integritet/autentificering** af brugerdata (i modsætning til AH, som kun giver integritet og ingen kryptering). |
| `authentication-algorithm hmac-sha-256-128;` | Integritetstjek (HMAC med SHA-256, trunkeret til 128 bit) på hver eneste krypterede datapakke. |
| `encryption-algorithm aes-128-cbc;` | Krypterer selve IP-pakkerne (brugertrafikken) inde i VPN-tunnelen. |
| `lifetime-seconds 3600;` | Phase 2 SA udløber efter 3600 sek (**1 time**), hvorefter nye sessionsnøgler genereres (kortere end Phase 1 for højere sikkerhed). |
| `perfect-forward-secrecy { keys group14; }` | **PFS:** Tvinger en helt ny Diffie-Hellman-udveksling (group 14) ved hver Phase 2 rekey, så Phase 2-nøglerne **ikke** afledes af Phase 1-nøglen. Hvis en nøgle senere kompromitteres, kan tidligere optaget trafik **ikke** dekrypteres! |
| `bind-interface st0.1;` | Gør VPN'en **route-based**: Al trafik der routes ud gennem `st0.1` krypteres automatisk og sendes gennem denne IPsec-tunnel. |
| `ike { gateway gw-to-peer; ipsec-policy ipsec-phase2-policy; }` | Kobler VPN-tunnelen sammen med Phase 1-gatewayen og Phase 2-politikken. |
| `establish-tunnels immediately;` | Får SRX'en til at **aktivt opbygge tunnelen med det samme** efter commit (uden at vente på den første brugerpakke – i modsætning til standarden `on-traffic`). |

---

## 3. De 4 ekstra ting der kræves for at Route-Based VPN virker!

Ud over `security ike` og `security ipsec` har `ass45network` disse 4 helt nødvendige brikker:

1. **Tunnel-interfacet `st0.1`:**
   ```junos
   set interfaces st0 unit 1 family inet
   ```
2. **Statisk rute ind i tunnelen:**
   ```junos
   set routing-options static route 10.2.2.0/24 next-hop st0.1
   ```
   Fortæller routeren, at pakker til det fjerne lokalnet (`10.2.2.0/24`) skal sendes ind i `st0.1` (og dermed krypteres i IPsec) i stedet for ud over default-ruten i klartekst.
3. **`host-inbound-traffic system-services ike` på `untrust`-interfacet (`ge-0/0/3.0`):**
   ```junos
   set security zones security-zone untrust interfaces ge-0/0/3.0 host-inbound-traffic system-services ike
   ```
   Uden `ike` blokeres UDP port 500 ind til SRX'en, og Phase 1 kommer **aldrig** op!
4. **Security zone + policies for `st0.1`:**
   - `st0.1` lægges i en security-zone (fx `vpn`).
   - Der oprettes policies både `from-zone trust to-zone vpn` og `from-zone vpn to-zone trust` med `then permit`, ellers dropper firewallen trafikken før/efter tunnelen.

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"Hvad er forskellen på IKE Phase 1 og IPsec Phase 2?"*
  → Phase 1 autentificerer peers og opbygger en sikker kontrolkanal (IKE SA). Phase 2 opbygger den krypterede datatunnel (IPsec SA med ESP) til selve brugertrafikken.
- **Fælde 2:** *"Hvad er formålet med `perfect-forward-secrecy (PFS)` i Phase 2?"*
  → At generere uafhængige sessionsnøgler med ny DH-udveksling ved hver rekey, så kompromittering af én nøgle ikke afslører tidligere optaget VPN-trafik.
- **Fælde 3:** *"Hvad sker der, hvis `host-inbound-traffic system-services ike` mangler på `ge-0/0/3.0` (`untrust`)?"*
  → SRX'en dropper indgående IKE-pakker (UDP 500), og VPN-tunnelen kommer ikke op.
- **Fælde 4:** *"Hvad er forskellen på `establish-tunnels immediately` og `on-traffic`?"*
  → `immediately` starter IKE/IPsec-forhandlingen med det samme. `on-traffic` venter med at opbygge tunnelen, indtil en bruger rent faktisk sender trafik mod `st0.1`.
- **Fælde 5:** *"Hvad skal matche 100 % mellem begge VPN-gateways for at tunnelen kommer op?"*
  → Phase 1 proposal (encryption, auth, DH-group, auth-method), `mode` (`main`), `pre-shared-key`, samt Phase 2 proposal (`esp`, encryption, auth) og `PFS`-gruppe.
