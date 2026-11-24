# 08 – Port Mirroring, Stateless Firewall Filter og Snort IDS

> **Brugt i:** `ass35network` (`r1-config.json`, `r1-config.set`, `dokumentation/snort/local.rules`).

I `ass35network` overvåger man al trafik til/fra DMZ WebServeren ved at **spejle (kopiere)** pakker fra DMZ-interfacet (`ge-0/0/2.0`) ud på et dedikeret monitor-interface (`ge-0/0/4.0`), hvor en Linux-maskine (PC4) kører **Snort IDS** og Wireshark.

---

## 1. Stateless `firewall filter` (IKKE det samme som `security policies`!)

```junos
firewall {
    filter my-port-filter {
        term myMirrorTerm {
            from {
                source-address {
                    0.0.0.0/0;
                }
            }
            then {
                count mirror-count;
                port-mirror;
                accept;
            }
        }
    }
}
```

### Hvad gør hver linje i praksis?
- **`firewall filter my-port-filter`**: Et **stateless** pakkefilter (svarer til en router ACL). Det tjekker hver enkelt pakke individuelt uden at huske sessioner.
- **`term myMirrorTerm`**: En regel i filteret.
- **`from { source-address 0.0.0.0/0; }`**: Matcher **alle** IPv4-pakker uanset kilde-IP.
- **`then { count mirror-count; }`**: Tæller hver matchet pakke og byte i en tæller (`show firewall`), så man kan verificere, at filteret faktisk rammes.
- **`then { port-mirror; }`**: **Non-terminating action** – sender en **kopi** af pakken til `forwarding-options port-mirroring`-destinationen, uden at forstyrre den originale pakke.
- **`then { accept; }`**: **KRITISK!** Tillader den originale pakke at fortsætte sin normale vej.
  - **HVAD SKER DER HVIS `accept;` MANGLER?** I Junos har et `firewall filter` en **implicit `discard` (drop)** til sidst! Hvis man glemmer `then accept;`, bliver pakkerne spejlet – eller droppet – så WebServeren stopper med at virke!

---

## 2. Påsætning af filteret på interfacet (`interfaces`)

```junos
interfaces {
    ge-0/0/2 {
        unit 0 {
            family inet {
                filter {
                    input my-port-filter;
                    output my-port-filter;
                }
                address 192.168.4.1/24;
            }
        }
    }
}
```
- **`filter input my-port-filter;`**: Spejler pakker der kommer **ind** i SRX'en fra DMZ (fx HTTP-svar *fra* WebServeren).
- **`filter output my-port-filter;`**: Spejler pakker der sendes **ud** af SRX'en mod DMZ (fx HTTP-requests og SYN-flood *til* WebServeren).
- Ved at sætte filteret på **både `input` og `output`** får Snort/Wireshark en komplet kopi af **begge retninger** af kommunikationen!

---

## 3. `forwarding-options port-mirroring`

```junos
forwarding-options {
    port-mirroring {
        input {
            rate 1;
        }
        family inet {
            output {
                interface ge-0/0/4.0 {
                    next-hop 2.2.2.1;
                }
            }
        }
    }
}
```

### Hvad gør hver linje i praksis?
- **`input { rate 1; }`**: **Sampling rate = 1 ud af 1 pakke (100 %)** af de pakker, som `then port-mirror` rammer, bliver kopieret. (Hvis `rate 10` var sat, ville kun hver 10. pakke blive kopieret).
- **`family inet { output { interface ge-0/0/4.0 { next-hop 2.2.2.1; } } }`**:
  - Sender de spejlede IPv4-pakker ud af `ge-0/0/4.0` (MonitorLan, som har IP `2.2.2.2/24`) med næste-hop destination `2.2.2.1` (IDS/Snort-maskinen PC4).
  - SRX'en bruger ARP på `2.2.2.1` til at finde PC4's MAC-adresse og aflevere kopierne direkte til PC4's netkort.

---

## 4. Snort IDS-reglen (`ass35network/dokumentation/snort/local.rules`)

```snort
alert tcp any any -> 192.168.4.10 80 ( msg:"ASS35 TCP SYN flood mod WebServer"; flags:S; detection_filter:track by_dst, count 100, seconds 1; classtype:attempted-dos; sid:1000035; rev:1; )
```

### Hver enkelt del af Snort-reglen forklaret:

#### Regel-header (`alert tcp any any -> 192.168.4.10 80`)
| Del | Betydning i praksis |
| :--- | :--- |
| `alert` | **Action:** Generér en alarm og log pakken (IDS-mode – blokerer **ikke** trafikken, modsat `drop` i IPS-mode). |
| `tcp` | **Protokol:** Matcher kun TCP-pakker. |
| `any any` | **Source IP & Source Port:** Fra enhver IP og enhver port. |
| `->` | **Retning:** Kun pakker i retning *mod* destinationen (ikke omvendt). |
| `192.168.4.10 80` | **Destination IP & Port:** WebServerens **private DMZ-IP** på port 80. *(Hvorfor `192.168.4.10` og ikke `10.56.16.85`? Fordi port mirroring sidder på `ge-0/0/2.0`, hvor Destination NAT allerede har oversat IP'en!)* |

#### Regel-options (inde i `(...)`)
| Option | Betydning i praksis |
| :--- | :--- |
| `msg:"..."` | Tekstbeskeden der skrives i Snorts alarm-log, når reglen udløses. |
| `flags:S;` | Matcher **kun** TCP-pakker hvor **udelukkende SYN-flaget** er sat (kendetegnet ved en ny TCP-forbindelse eller et TCP SYN-flood angreb). Matcher fx *ikke* SYN-ACK (`SA`) eller ACK (`A`). |
| `detection_filter:track by_dst, count 100, seconds 1;` | **Tærskelværdi (threshold):** Udløser først en alarm, når der ses **mere end 100** matchende SYN-pakker inden for **1 sekund** mod samme destination (`track by_dst` = `192.168.4.10`). Almindelig HTTP-trafik udløser derfor ingen alarm – kun et SYN-flood! |
| `classtype:attempted-dos;` | Kategoriserer alarmen som et forsøg på Denial-of-Service (DoS). |
| `sid:1000035;` | **Snort Rule ID** – unikt ID for reglen (lokale/egne regler bruger typisk `sid >= 1000000`). |
| `rev:1;` | **Revision** – versionsnummer på reglen (øges når man redigerer reglen). |

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"Hvorfor matcher Snort-reglen `192.168.4.10 80` i stedet for den offentlige IP `10.56.16.85 80`?"*
  → Fordi port-mirroring filteret sidder på DMZ-interfacet (`ge-0/0/2.0`), og på det tidspunkt har SRX'ens **Destination NAT** allerede oversat `10.56.16.85` til `192.168.4.10`.
- **Fælde 2:** *"Hvad er forskellen på `firewall filter` og `security policies` på en Juniper SRX?"*
  → `firewall filter` er **stateless** (per-pakke, bundet direkte til et interface `input`/`output`, fx til port-mirroring og tællere). `security policies` er **stateful** (session-baseret, mellem `from-zone` og `to-zone`).
- **Fælde 3:** *"Hvad sker der hvis `then accept;` fjernes fra `myMirrorTerm`?"*
  → Pakkerne rammes af filterets **implicitte discard**, og al trafik ind/ud af DMZ-interfacet droppes!
- **Fælde 4:** *"Hvorfor bruger Snort-reglen `track by_dst` i stedet for `track by_src` ved et SYN-flood?"*
  → Fordi et SYN-flood (især med spoofede source-IP'er / DDoS) kan komme fra mange forskellige source-IP'er samtidigt, men alle rammer samme offer (`by_dst` = WebServeren).
- **Fælde 5:** *"Stopper Snort i `ass35network` selve SYN-flood angrebet mod WebServeren?"*
  → **Nej!** Snort modtager kun en *kopi* af trafikken via port mirroring (passiv **IDS** med `alert`), så den detekterer og alarmerer kun – den sidder ikke inline og kan ikke droppe pakkerne.
