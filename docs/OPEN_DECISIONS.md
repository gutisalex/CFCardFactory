# VeryFlow - Offene Entscheidungen vor Umsetzung

**Status:** Klärung erforderlich vor Entwicklungsstart
**Stand:** Januar 2026

---

## Zusammenfassung

Die Architektur steht. Die offenen Punkte sind keine Richtungsfragen mehr, sondern **Konkretisierung und Auswahl** - vor allem bei Hardware und Sicherheitsprozessen.

---

## 1. Referenzhardware final festlegen

### Offene Auswahl

| Komponente | Status | Optionen |
|------------|--------|----------|
| Controller-Modell | **Offen** | z.B. HID Mercury, ASSA ABLOY, Axis |
| OSDP-Reader-Modell | **Offen** | z.B. HID Signo 20, STid Architect |

### Davon abhängig

- Welche Schnittstellen wir integrieren (OSDP-Befehle, Firmware-Versionen)
- Wie Offline-Logik technisch umgesetzt wird
- Wie Tests und Abnahme aussehen
- Beschaffung für Pilot-Hardware

### Empfehlung: HID Signo 20 + Mercury Controller

**Reader: HID Signo 20**

| Kriterium | Bewertung |
|-----------|-----------|
| OSDP Secure Channel | Ja, vollständig unterstützt |
| DESFire EV3 Support | Ja, native Unterstützung |
| Marktverbreitung | Sehr hoch, gute Ersatzteilversorgung |
| Dokumentation | Umfangreich, guter Developer Support |
| Preis | Mittelfeld (~150-200€) |

**Begründung:**
- HID ist Marktführer im Enterprise-Bereich - Kunden kennen und vertrauen der Marke
- Signo-Serie ist speziell für OSDP Secure Channel optimiert
- Große Installationsbasis bedeutet viele Erfahrungsberichte und Community-Support
- Zukunftssicher: HID investiert stark in OSDP-Weiterentwicklung

**Controller: HID Mercury LP1502 oder EP1502**

| Kriterium | Bewertung |
|-----------|-----------|
| OSDP Host | Ja |
| Offline-Speicher | Ja, bis zu 100.000 Credentials |
| API | REST + proprietär |
| Relais-Steuerung | Integriert |

**Begründung:**
- Mercury-Controller sind Industriestandard für mittlere bis große Installationen
- Gute Dokumentation für Integration
- Falls wir später Controller-Logik nutzen wollen, ist die Option offen
- Alternativ: Günstigerer "dumm-Controller" nur für Relais, wenn Gateway alle Logik übernimmt

**Alternative für Budget-Pilot: STid Architect Blue**
- Günstiger (~100€), gute OSDP-Unterstützung
- Weniger Enterprise-Features, aber für Pilot ausreichend

### Referenz-Kit für Pilot

| Komponente | Menge | Geschätzter Preis |
|------------|-------|-------------------|
| HID Signo 20 Reader | 2 | ~400€ |
| Mercury LP1502 Controller | 1 | ~300€ |
| DESFire EV3 Karten | 20 | ~100€ |
| Raspberry Pi 4 (4GB) | 1 | ~80€ |
| Netzteil, Verkabelung | 1 Set | ~50€ |
| **Gesamt** | | **~930€** |

---

## 2. Integrationsansatz: Controller vs. Gateway

### Zwei Optionen

| Ansatz | Beschreibung | Vor-/Nachteile |
|--------|--------------|----------------|
| **A: Controller-API** | Controller trifft Entscheidung, Gateway synchronisiert nur Regeln | + Nutzung vorhandener Logik<br>- Abhängigkeit von Controller-Firmware |
| **B: Gateway als Logik-Layer** | Controller ist nur I/O (Reader + Relais), Gateway entscheidet alles | + Volle Kontrolle, Hardware-unabhängig<br>- Mehr Entwicklungsaufwand |

### Entscheidungskriterien

- Welche Controller-APIs sind verfügbar?
- Wie schnell ist die Challenge-Response über Controller vs. direkt?
- Wie wird Offline-Capability beeinflusst?
- Wie einfach ist spätere Hardware-Erweiterung?

### Empfehlung: Option B - Gateway als Logik-Layer

**Klare Empfehlung für Option B** aus folgenden Gründen:

| Argument | Begründung |
|----------|------------|
| **Produktdifferenzierung** | VeryFlow's USP ist die Blockchain-Verankerung und eigene Sicherheitslogik. Wenn der Controller entscheidet, verlieren wir Kontrolle über das Kernprodukt. |
| **Hardware-Unabhängigkeit** | Mit Gateway als Logik-Layer können wir später jeden OSDP-fähigen Reader/Controller unterstützen, ohne Code-Änderungen. Das Pflichtenheft fordert explizit "Hardwareunabhängigkeit innerhalb definierter Profile". |
| **Offline-Garantie** | Wenn das Gateway entscheidet, kontrollieren wir exakt, wie Offline funktioniert. Bei Controller-Logik sind wir von Firmware-Verhalten abhängig. |
| **Key Management** | Die DESFire-Challenge-Response mit Key Diversification muss vom Gateway gesteuert werden - der Controller kennt unsere Schlüsselableitung nicht. |
| **Audit-Trail** | Wir brauchen jeden Zutrittsversuch im Event-Log. Controller loggen oft nur erfolgreiche Zutritte. |

**Technische Architektur:**

```
┌─────────────┐     OSDP-SC      ┌─────────────┐
│   Reader    │◄────────────────►│   Gateway   │
│ (HID Signo) │                  │    (Go)     │
└─────────────┘                  └──────┬──────┘
                                        │
                                        │ GPIO/Relais
                                        ▼
                                 ┌─────────────┐
                                 │ Controller  │
                                 │ (nur I/O)   │
                                 └─────────────┘
```

**Entwicklungsaufwand:**
- OSDP Secure Channel Client: ~2-3 Wochen
- DESFire Auth-Logik: ~1-2 Wochen
- Decision Engine: ~1 Woche

Der Mehraufwand ist überschaubar und zahlt sich langfristig aus.

**Fallback-Option:**
Falls Performance-Probleme auftreten (unwahrscheinlich bei <300ms Anforderung), können wir später Controller-Logik als Cache nutzen, aber Gateway bleibt Master.

---

## 3. Karten-Personalisierung und Key Handling

### Offene Prozessfragen

| Frage | Status |
|-------|--------|
| Wo werden Karten personalisiert? | **Offen** - Intern, beim Kunden, Drittanbieter? |
| Wie werden Schlüssel auf Karte gesetzt? | **Offen** - Personalisierungsstation, welche Software? |
| Wie wird Key Versioning technisch umgesetzt? | **Konzept steht**, Implementierung offen |
| Wie kommt cardId sauber ins System? | **Offen** - Scan, Import, manuelle Eingabe? |

### Empfehlung: Zentrale Personalisierung bei CF Card Factory

**Empfohlenes Modell: "Personalization-as-a-Service"**

| Aspekt | Empfehlung | Begründung |
|--------|------------|------------|
| **Ort** | CF Card Factory (zentral) | Schlüssel verlassen nie das kontrollierte Umfeld. Beim Kunden wäre Key-Schutz schwer zu garantieren. |
| **Equipment** | Desktop-Reader + Personalisierungs-Tool | Einmalinvestition ~500€, CF Card Factory hat vermutlich bereits NFC-Equipment. |
| **Prozess** | Batch-Personalisierung vor Versand | Karten werden vorpersonalisiert, Kunde aktiviert nur noch im Portal. |

**Personalisierungsflow (empfohlen):**

```
┌─────────────────────────────────────────────────────────────┐
│                    CF Card Factory                          │
│                                                             │
│  1. Mandant bestellt Karten über Portal                     │
│  2. Backend generiert Card-UIDs + Key-Slots                 │
│  3. Personalisierungs-Tool holt Aufträge vom Backend        │
│  4. Für jede Karte:                                         │
│     a) Card Key = HKDF(MasterKey, CardUID || Version)       │
│     b) Card Key auf DESFire EV3 schreiben                   │
│     c) Public Key Hash berechnen                            │
│     d) Backend: cardId + pubKeyHash speichern               │
│  5. Karten an Kunden versenden                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Beim Kunden                              │
│                                                             │
│  6. Admin öffnet Portal → Karten → "Aktivieren"             │
│  7. Karte an beliebigen Reader halten (oder UID eingeben)   │
│  8. Karte wird Mitarbeiter zugewiesen                       │
│  9. Backend: On-Chain Event "CardIssued"                    │
│  10. Gateway synchronisiert neue Berechtigung               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Begründung für zentrale Personalisierung:**

1. **Sicherheit:** Master Keys verlassen nie CF Card Factory. Beim dezentralen Modell müssten wir Key-Material an Kunden oder Dritte geben.

2. **Qualitätskontrolle:** Fehler bei Personalisierung sind teuer (Karte unbrauchbar). Zentrales Setup mit geschultem Personal minimiert Fehlerrate.

3. **Geschäftsmodell:** CF Card Factory produziert bereits Karten - Personalisierung ist natürliche Erweiterung. Zusätzliche Marge möglich.

4. **Einfachheit für Kunden:** Kunden bekommen fertig personalisierte Karten, müssen nur noch aktivieren. Keine Hardware, keine Schulung.

**Key Versioning - technische Umsetzung:**

```go
// Card Key Derivation
func DeriveCardKey(masterKey []byte, cardUID []byte, version uint32) []byte {
    info := append(cardUID, byte(version))
    return hkdf.Expand(sha256.New, masterKey, info, 16) // AES-128
}

// Gateway kennt alle Versionen und probiert bei Auth:
// 1. Aktuelle Version
// 2. Falls Fehler: vorherige Version (für Übergangszeit)
```

**Personalisierungs-Tool (zu entwickeln):**

```
tools/card-personalizer/
├── cmd/personalizer/     # CLI-Tool
├── internal/
│   ├── backend/         # API-Client für Aufträge
│   ├── desfire/         # DESFire EV3 Kommunikation
│   └── crypto/          # Key Derivation
└── README.md            # Anleitung für Operator
```

Geschätzter Aufwand: ~1-2 Wochen

**Alternative: Dezentrale Personalisierung (nicht empfohlen)**

Falls Kunde selbst personalisieren muss:
- HSM beim Kunden erforderlich
- Schulung notwendig
- Höheres Fehlerrisiko
- Komplexere Key-Distribution

---

## 4. Schlüsselmanagement Governance

### Organisatorische Fragen (nicht Code)

| Frage | Status |
|-------|--------|
| Wer erzeugt Mandanten-Keys? | **Offen** - VeryFlow, Kunde, HSM-Provider? |
| Wo liegen Master Keys? | **Konzept:** AWS KMS / GCP KMS |
| Wie läuft Key Rotation? | **Offen** - Automatisch, manuell, Zeitintervall? |
| Sonderfälle? | **Offen** - Key Compromise, Mandant-Exit, etc. |

### Empfehlung: VeryFlow-managed Keys in Cloud KMS

**Governance-Modell (empfohlen):**

| Aspekt | Empfehlung | Begründung |
|--------|------------|------------|
| **Key-Erzeugung** | VeryFlow bei Mandant-Onboarding | Einheitlicher Prozess, keine Abhängigkeit vom Kunden |
| **Key-Speicherung** | AWS KMS (Frankfurt) | DSGVO-konform, Audit-Logs, keine eigene HSM-Infrastruktur nötig |
| **Key-Zugriff** | Nur Backend-Service | Kein Portal-Zugriff auf Keys, nur auf verschlüsselte Ableitungen |
| **Rotation** | Jährlich, manuell ausgelöst | Automatische Rotation ist für MVP Overkill |

**Warum AWS KMS statt eigenem HSM:**

1. **Kosten:** Eigenes HSM ~10.000€+ Setup + Wartung. AWS KMS: ~1€/Key/Monat + Nutzung.
2. **Compliance:** AWS KMS ist SOC2, ISO 27001, DSGVO-zertifiziert.
3. **Verfügbarkeit:** AWS garantiert 99.999% Uptime.
4. **Audit:** Jeder Key-Zugriff wird in CloudTrail geloggt.

**Key-Hierarchie (empfohlen):**

```
┌─────────────────────────────────────────┐
│           AWS KMS (Frankfurt)           │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │     VeryFlow Root Key           │    │
│  │     (CMK, nie exportiert)       │    │
│  └──────────────┬──────────────────┘    │
│                 │                       │
│                 │ KDF                   │
│                 ▼                       │
│  ┌─────────────────────────────────┐    │
│  │   Tenant Master Key (Mandant A) │    │
│  │   (Data Key, verschlüsselt)     │    │
│  └──────────────┬──────────────────┘    │
│                 │                       │
└─────────────────┼───────────────────────┘
                  │
                  │ HKDF (im Backend-RAM)
                  ▼
         ┌─────────────────┐
         │   Card Key      │
         │   (pro Karte)   │
         └─────────────────┘
```

**Rotation-Prozess (empfohlen):**

```
1. Admin löst Rotation aus (Portal → Backend)
2. Backend erzeugt neuen Tenant Master Key (Version N+1)
3. Alter Key bleibt aktiv (Übergangszeit: 90 Tage)
4. Neue Karten bekommen Version N+1
5. Gateways kennen beide Versionen
6. Nach 90 Tagen: Key Version N deaktivieren
7. Dokumentation im Audit-Log
```

**Sonderfälle - Runbook (MVP):**

| Szenario | Maßnahme |
|----------|----------|
| **Key Compromise vermutet** | 1. Alle Karten des Mandanten sperren<br>2. Neuen Master Key erzeugen<br>3. Alle Karten neu personalisieren<br>4. Vorfall dokumentieren |
| **Mandant kündigt** | 1. Alle Karten deaktivieren<br>2. Master Key archivieren (7 Jahre Aufbewahrung)<br>3. Mandant-Daten nach Frist löschen |
| **Gateway gestohlen** | Gateway hat nie Master Keys, nur abgeleitete Card Keys im RAM. Risiko: minimal. Gateway in Portal deaktivieren. |

**Warum keine automatische Rotation im MVP:**

- Rotation erfordert Karten-Repersonalisierung oder Dual-Key-Support
- Komplexität für MVP zu hoch
- Jährliche manuelle Rotation ist für Pilot ausreichend
- Kann in Phase 3 automatisiert werden

---

## 5. Pilot-Rahmen

### Noch nicht festgezurrt

| Parameter | Vorschlag | Bestätigung |
|-----------|-----------|-------------|
| Anzahl Türen | 1-3 | **Offen** |
| Anzahl Karten | 10-20 | **Offen** |
| Anzahl Gateways | 1 | **Offen** |
| Offline-Anforderung | 24h | **Offen** |
| Pilot-Dauer | 4-8 Wochen | **Offen** |

### Empfehlung: Minimaler Pilot mit klarem Fokus

**Empfohlener Pilot-Scope:**

| Parameter | Empfehlung | Begründung |
|-----------|------------|------------|
| **Türen** | 2 | Eine Tür = kein echter Test. Zwei Türen zeigen Multi-Door-Logik, ohne Komplexität zu explodieren. |
| **Karten** | 15 | 10 aktive Nutzer + 5 Reserve für Tests (Sperrung, Ersatz). |
| **Gateways** | 1 | Ein Gateway pro Standort. Mehr Gateways erst nach Validierung. |
| **Offline-Test** | 48h | 24h ist Minimum, 48h gibt Sicherheitspuffer und testet Event-Queue besser. |
| **Pilot-Dauer** | 6 Wochen | 4 Wochen zu kurz für echte Nutzung. 8 Wochen verzögert Feedback unnötig. |

**Empfohlener Pilot-Zeitplan:**

```
Woche 1:    Setup + Schulung Admin
Woche 2-3:  Normalbetrieb, Feedback sammeln
Woche 4:    Offline-Test (48h), Sperrungen testen
Woche 5:    Edge Cases, Probleme beheben
Woche 6:    Abnahme, Dokumentation, Go/No-Go
```

### Abnahmekriterien - Empfehlung

| Nr. | Kriterium | Testfall | Akzeptanz |
|-----|-----------|----------|-----------|
| 1 | **Offline-Zutritt** | Gateway 48h vom Netz trennen, 50 Zutritte durchführen | 100% erfolgreich |
| 2 | **Sperrung wirkt** | Karte im Portal sperren, nach Sync (<60s) Zutritt versuchen | Abgelehnt |
| 3 | **Klon-Schutz** | UID mit Chameleon-Karte klonen, Zutritt versuchen | Abgelehnt + Security-Event |
| 4 | **Sync nach Offline** | Nach 48h Offline wieder online, Events prüfen | Alle Events synchronisiert |
| 5 | **Proof on-chain** | Karte ausgeben + sperren | TX auf Polygon/Base Testnet |
| 6 | **Latenz** | 100 Zutritte messen | Median <200ms, P99 <500ms |
| 7 | **Monitoring** | Dashboard prüfen | Gateway-Status, letzte Sync, Queue sichtbar |

**Begründung für strenge Kriterien:**

- Pilot-Erfolg entscheidet über Produktreife
- Lieber früh Probleme finden als beim echten Kunden
- Klare Metriken vermeiden Diskussionen bei Abnahme

### Monitoring-Metriken - Empfehlung

| Metrik | Pflicht | Begründung |
|--------|---------|------------|
| Gateway Online/Offline | **Ja** | Grundlage für Support |
| Letzte Sync-Zeit | **Ja** | Zeigt Probleme früh |
| Event-Queue-Größe | **Ja** | Warnung bei Offline-Überlauf |
| Türstatus | **Ja** | Schnelle Fehlerdiagnose |
| Zugriffslatenz (ms) | **Ja** | Performance-Baseline für Produkt |
| Fehlerrate | **Ja** | Qualitätsmetrik |

**Empfehlung: Alle Metriken Pflicht**

Im Pilot wollen wir maximale Sichtbarkeit. Kosten für Logging sind minimal, Wert für Debugging hoch.

---

## 6. Abrechnungstechnische Details

### Preismodell

**Preis pro aktiver Karte pro Monat** (bestätigt)

### Technisch offene Fragen

| Frage | Status |
|-------|--------|
| Was zählt als "aktiv"? | **Offen** - Status=aktiv zu irgendeinem Zeitpunkt im Monat? |
| Stichtag-Zählung oder Tagesdurchschnitt? | **Offen** |
| Export-Format? | **Offen** - CSV, JSON, API-Abruf? |
| Selbst-Abrechnung oder Daten für Buchhaltung? | **Offen** |

### Empfehlung: High-Water-Mark Zählung

**Definition "aktive Karte" (empfohlen):**

> Eine Karte gilt als aktiv im Abrechnungsmonat, wenn sie **an mindestens einem Tag** des Monats den Status "aktiv" hatte.

| Zählmethode | Beschreibung | Empfehlung |
|-------------|--------------|------------|
| **High-Water-Mark** | Karte war irgendwann im Monat aktiv → zählt | **Empfohlen** |
| Stichtag (z.B. 1. des Monats) | Nur aktive Karten am Stichtag | Unfair bei Monatsmitte-Aktivierung |
| Tagesdurchschnitt | Summe(aktive Tage) / 30 | Komplex, schwer erklärbar |

**Begründung High-Water-Mark:**

1. **Einfach zu erklären:** "Wenn die Karte im Monat benutzt werden konnte, zahlen Sie."
2. **Fair für beide Seiten:** Kunde zahlt nicht für komplett inaktive Karten, aber auch nicht für Tages-Nutzung.
3. **Einfach zu implementieren:** Ein SQL-Query, keine Tagesaggregation.
4. **Branchenüblich:** Die meisten SaaS-Abrechnungen funktionieren so.

**SQL-Logik (empfohlen):**

```sql
SELECT 
    tenant_id,
    DATE_TRUNC('month', :billing_date) AS billing_month,
    COUNT(DISTINCT card_id) AS active_cards
FROM cards c
LEFT JOIN card_assignments ca ON c.id = ca.card_id
WHERE 
    c.tenant_id = :tenant_id
    AND ca.assigned_at <= DATE_TRUNC('month', :billing_date) + INTERVAL '1 month - 1 day'
    AND (ca.revoked_at IS NULL OR ca.revoked_at >= DATE_TRUNC('month', :billing_date))
GROUP BY tenant_id;
```

### Export-Format - Empfehlung

**Beide Formate anbieten:**

| Format | Zielgruppe | Inhalt |
|--------|------------|--------|
| **CSV** | Buchhaltung, Excel-Import | Mandant, Monat, Anzahl, Einzelpreis, Summe |
| **JSON** | API-Integration, Automatisierung | Strukturierte Daten + Metadaten |

**CSV-Beispiel:**

```csv
tenant_id,tenant_name,billing_month,active_cards,unit_price,total
550e8400-e29b,Musterfirma GmbH,2026-01,47,2.50,117.50
```

**JSON-Beispiel:**

```json
{
  "tenant_id": "550e8400-e29b-...",
  "tenant_name": "Musterfirma GmbH",
  "billing_month": "2026-01",
  "active_cards": 47,
  "unit_price_eur": 2.50,
  "total_eur": 117.50,
  "card_ids": ["card-1", "card-2", ...],  // Optional, auf Anfrage
  "generated_at": "2026-02-01T00:00:00Z",
  "proof_hash": "0x..."  // Optional: Hash on-chain verankert
}
```

### Selbst-Abrechnung vs. Daten-Export

**Empfehlung: Daten-Export (keine Rechnungsstellung im MVP)**

| Option | Aufwand | Empfehlung |
|--------|---------|------------|
| Volle Rechnungsstellung | Hoch (Umsatzsteuer, Zahlungsabwicklung, Mahnwesen) | **Nicht MVP** |
| Daten-Export für Buchhaltung | Niedrig | **MVP** |
| Stripe/Paddle Integration | Mittel | **Phase 3** |

**Begründung:**
- CF Card Factory hat vermutlich bereits Buchhaltung/ERP
- Export-Daten können dort eingelesen werden
- Rechnungsstellung ist regulatorisch komplex (Umsatzsteuer, Reverse Charge, etc.)
- Im MVP: Fokus auf Kernprodukt, nicht auf Billing-Software

**Spätere Erweiterung (Phase 3):**
- Stripe-Integration für automatische Abrechnung
- Selbstbedienungs-Billing-Portal für Mandanten
- Nutzungsbasierte Abrechnung mit On-Chain-Verankerung

---

## 7. Bewusst nicht im MVP (Scope-Schutz)

### Explizit ausgeschlossen

| Feature | Grund | Wann sinnvoll? |
|---------|-------|----------------|
| **Smart Locks** | Bluetooth/WiFi-Protokolle, Batterie-Management, völlig andere Integration | Phase 3+, separates Modul |
| **Mobile Wallets** | NFC-Emulation, Apple/Google Wallet-Integration, App-Entwicklung | Phase 3+, nach Marktvalidierung |
| **HR-System-Integration** | Jeder Kunde hat anderes System, hoher Customizing-Aufwand | Phase 3, als kostenpflichtiges Add-on |
| **Freie Regel-Engine** | Komplexe Regeln → Audit-Probleme, Offline-Logik explodiert | Vermutlich nie, by Design |
| **Multi-Site Management** | UI-Komplexität, Sync-Logik zwischen Standorten | Phase 2, nach Single-Site-Validierung |
| **Besuchermanagement** | Temporäre Credentials, Self-Service-Kiosk | Phase 3 |
| **Video-Integration** | Türkamera-Anbindung, Event-Korrelation | Phase 3+ |

### Begründung für harten Scope-Schnitt

**Warum ist Scope-Disziplin hier besonders wichtig?**

1. **Hardware-Abhängigkeit:** Jedes Feature muss offline funktionieren. Mehr Features = mehr Edge Cases = mehr Bugs.

2. **Sicherheitskritisch:** Zutrittssysteme sind Angriffsziele. Jede Codezeile ist potenzielle Angriffsfläche.

3. **Pilotfokus:** Der Pilot muss zeigen, dass das Kernprodukt funktioniert. Nicht, dass wir viel bauen können.

4. **Time-to-Market:** Je länger MVP dauert, desto höher das Risiko, dass Wettbewerber überholen.

### Scope-Creep-Warnung

**Bei Anfragen für ausgeschlossene Features:**

```
1. "Das ist eine gute Idee für Phase 3."
2. "Für den Pilot fokussieren wir uns auf [Kernfeature]."
3. "Ich notiere das in der Roadmap."
4. NICHT in Sprint aufnehmen.
5. NICHT "mal schnell" einbauen.
```

**Ausnahme:** Wenn Pilot-Kunde Feature als Blocker nennt → Eskalation an Produktverantwortlichen, bewusste Entscheidung.

---

## Nächste Schritte

### Empfohlener Ablauf vor Entwicklungsstart

```
┌─────────────────────────────────────────────────────────────┐
│  WOCHE 1: Hardware-Entscheidung                             │
│                                                             │
│  □ Referenz-Kit bestellen (HID Signo + Mercury)             │
│  □ DESFire EV3 Karten bestellen                             │
│  □ Raspberry Pi 4 + Zubehör bestellen                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  WOCHE 2-3: Hardware-Test + Integration entscheiden         │
│                                                             │
│  □ OSDP-Kommunikation testen                                │
│  □ DESFire Challenge-Response testen                        │
│  □ Latenz messen                                            │
│  □ Entscheidung: Gateway als Logik-Layer bestätigen         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  WOCHE 3: Personalisierungs-Workshop                        │
│                                                             │
│  □ Mit CF Card Factory: Personalisierungs-Equipment klären  │
│  □ Key-Flow durchsprechen                                   │
│  □ Verantwortlichkeiten festlegen                           │
│  □ Governance-Dokument erstellen                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  WOCHE 4: Pilot-Planung                                     │
│                                                             │
│  □ Pilot-Kunde identifizieren                               │
│  □ Scope bestätigen (2 Türen, 15 Karten)                    │
│  □ Abnahmekriterien abstimmen                               │
│  □ Zeitplan festlegen                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  WOCHE 5+: Entwicklungsstart                                │
│                                                             │
│  → Alle Entscheidungen dokumentiert                         │
│  → Hardware verfügbar                                       │
│  → Pilot-Rahmen klar                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Entscheidungs-Checkliste

| # | Entscheidung | Empfehlung | Status |
|---|--------------|------------|--------|
| 1 | Controller-Modell | HID Mercury LP1502 | ☐ Offen |
| 2 | Reader-Modell | HID Signo 20 | ☐ Offen |
| 3 | Integrationsansatz | Gateway als Logik-Layer | ☐ Offen |
| 4 | Personalisierung | Zentral bei CF Card Factory | ☐ Offen |
| 5 | Key-Speicherung | AWS KMS Frankfurt | ☐ Offen |
| 6 | Pilot-Scope | 2 Türen, 15 Karten, 6 Wochen | ☐ Offen |
| 7 | Billing-Methode | High-Water-Mark + CSV-Export | ☐ Offen |

### Entscheidungs-Template

Für jede Entscheidung bitte dokumentieren:

```markdown
## Entscheidung: [Thema]

**Datum:** YYYY-MM-DD
**Teilnehmer:** [Namen]
**Entscheidung:** [Gewählte Option]
**Begründung:** [Warum diese Option?]
**Verworfene Alternativen:** [Was wurde nicht gewählt und warum?]
**Nächste Aktion:** [Konkrete nächste Schritte]
**Verantwortlich:** [Wer setzt um?]
```

---

## Zusammenfassung der Empfehlungen

| Bereich | Empfehlung | Konfidenz |
|---------|------------|-----------|
| Hardware | HID Signo 20 + Mercury Controller | Hoch |
| Integration | Gateway als Logik-Layer (Option B) | Hoch |
| Personalisierung | Zentral bei CF Card Factory | Hoch |
| Key Management | AWS KMS, VeryFlow-managed | Hoch |
| Pilot | 2 Türen, 15 Karten, 6 Wochen | Mittel |
| Billing | High-Water-Mark, CSV-Export | Hoch |

**Konfidenz-Legende:**
- **Hoch:** Klare technische/wirtschaftliche Gründe, geringe Risiken
- **Mittel:** Abhängig von Kundenfeedback, kann angepasst werden
- **Niedrig:** Mehrere gleichwertige Optionen, braucht mehr Input

---

## Referenzen

- [Architekturplan (EN)](./architecture/ARCHITECTURE_PLAN.md)
- [Architekturplan (DE)](./architecture/ARCHITECTURE_PLAN_DE.md)
- [Pflichtenheft MVP](./requirements/PFLICHTENHEFT_MVP.md)
