# 01 – System, grundlæggende struktur og config-formater

> **Brugt i:** Alle Junos-repos (`ass6`–`ass54`).

---

## 1. De tre Junos-formater du har set i dine repos

Junos lagrer den samme konfiguration, men kan vise den i tre forskellige formater. Til eksamen kan spørgsmål vise et hvilket som helst af dem:

1. **Hierarkisk curly-brace format** (`show configuration` – ofte i `.conf`, `.txt` og nogle af dine `.json`-filer):
   ```junos
   system {
       host-name SRX-C1;
       services {
           ssh;
       }
   }
   ```
2. **`set`-kommandoer** (`show configuration | display set` – fx `ass35network/dokumentation/r1-config.set` og `ass38network/r1-config.set`):
   ```junos
   set system host-name SRX-C1
   set system services ssh
   ```
3. **JSON-format** (`show configuration | display json` – fx `ass34`, `ass35`, `ass37`, `ass45`, `ass52`):
   ```json
   {
     "system": {
       "host-name": "SRX-C1",
       "services": { "ssh": true }
     }
   }
   ```

> **Bemærk til eksamen:** Indholdet og betydningen er 100 % identisk i alle tre formater – det er kun syntaksen der er forskellig.

---

## 2. `version`

```junos
version 12.1X47-D15.4;
```

- **Hvad gør den i praksis?** Angiver hvilken Junos OS-softwareversion konfigurationen blev gemt under (fx `12.1X46-D55.3` på fysisk SRX240 eller `12.1X47-D15.4` på vSRX).
- **Effekt på trafik:** Ingen direkte effekt på routing eller firewall. Den bruges af Junos til at tjekke syntaks-kompatibilitet, når en konfiguration indlæses (`load override`).

---

## 3. `system`-stanzaen

```junos
system {
    host-name SRX-C1;
    root-authentication {
        encrypted-password "$1$4TkbZDtp$6E8C6Bg7K6gnHR31XnJjl0";
    }
    services {
        ssh;
    }
}
```

### `host-name <navn>;`
- **Hvad gør den i praksis?** Sætter routerens navn i CLI-prompten (`root@SRX-C1>`) og i logs/SNMP.
- **Vigtigt:** `host-name` påvirker **ikke** IP-adresser, DNS eller OSPF router-id.

### `root-authentication { encrypted-password "..."; }`
- **Hvad gør den i praksis?** Gemmer hash'et (krypteret med MD5-crypt, kendetegnet ved `$1$...`) af `root`-brugerens adgangskode.
- **Krav i Junos:** Du **kan ikke committe** en konfiguration på en Juniper SRX uden at `root-authentication` er sat. Hvis den slettes, fejler `commit check`.

### `services { ssh; }`
- **Hvad gør den i praksis?** Starter SSH-daemonen (TCP port 22) på selve SRX-enheden, så man kan logge ind via netværket.
- **Klassisk eksamensfælde:** `set system services ssh` er **ikke nok alene** for at kunne SSH'e til routeren! Det indgående interface's security-zone skal **også** tillade `ssh` under `host-inbound-traffic system-services` (se [04-security-zones-policies-apps.md](04-security-zones-policies-apps.md)).
  - `system services ssh` = *tænder serveren* på routeren.
  - `security zones ... host-inbound-traffic system-services ssh` = *åbner firewallen* ind til routerens egen kontrolplan på det pågældende interface/zone.

---

## 4. Commit-modellen i Junos (kort opsamling fra `ass9` / `ass35` / `ass38`)

- **Candidate configuration** vs. **Active configuration**: Når du skriver `set ...` eller `load override terminal`, ændrer du kun *candidate*-konfigurationen. Intet træder i kraft i netværket, før du kører `commit`.
- `commit check`: Validerer syntaks og afhængigheder (fx at en zone eller pool der refereres til faktisk eksisterer) uden at aktivere config'en.
- `load override terminal`: Erstatter **hele** candidate-konfigurationen med det du indsætter (i modsætning til `load merge`, som fletter ind).

---

## Typiske multiple-choice fælder

- **Fælde 1:** *"SSH er aktiveret under `system services ssh`, men administrator kan stadig ikke oprette SSH-forbindelse til `ge-0/0/5.0`. Hvorfor?"*
  → Fordi `ssh` mangler i `host-inbound-traffic system-services` på zonen/interfacet for `ge-0/0/5.0`.
- **Fælde 2:** *"Hvorfor står adgangskoden som `$1$...` i config-filen?"*
  → Fordi Junos gemmer adgangskoder som en envejs-hash (`encrypted-password`), ikke i klartekst.
