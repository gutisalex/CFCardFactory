# VeryFlow - Pflichtenheft Softwareentwicklung

**Version:** MVP 1.0

---

## 1. Ziel des Systems

VeryFlow ist eine Zutrittsplattform, die physische Zutrittsentscheidungen lokal und offlinefähig trifft, während Verwaltung, Nachvollziehbarkeit und Abrechnung zentral erfolgen.

Zutritt ist Modul 1. Die Plattform ist bewusst offen für weitere Berechtigungsfälle.

### Kernprinzipien

- Lokale Entscheidung an der Tür
- Keine Cloud- oder Blockchain-Abhängigkeit pro Zutritt
- Echte kryptografische Kartenprüfung
- Einfaches Preismodell pro aktiver Karte
- Hardwareunabhängigkeit innerhalb definierter Profile

---

## 2. Systemabgrenzung

### Im Scope MVP

| Komponente | Beschreibung |
|------------|--------------|
| Webportal | Browserbasierte Administrationsoberfläche |
| Backend Services | Mandantenfähige Plattform |
| Edge Gateway Software | Lokale Instanz pro Standort |
| Karten- und Schlüssel-Lifecycle-Logik | Verwaltung des gesamten Kartenlebenszyklus |
| Proof Layer Integration | Web3-basierte Nachweisverankerung |
| Monitoring und Basisdiagnose | Überwachung und Fehleranalyse |

### Nicht im Scope MVP

- Eigenentwicklung von Türhardware
- Mechanische Türkomponenten
- Brandschutz- und Fluchtweglogik
- Smart Lock Integration
- Mobile Wallets
- On-Chain-Transaktionen pro Zutritt

---

## 3. Architekturübersicht

### Komponenten

#### 3.1 Webportal

Browserbasierte Administrationsoberfläche für Mandanten-Administratoren.

#### 3.2 Backend

- Mandantenfähige Plattform
- Regelverwaltung
- Kartenverwaltung
- Proof-Erzeugung
- Synchronisation mit Gateways

#### 3.3 Edge Gateway

- Lokale Instanz pro Standort
- Synchronisiert Regeln und Sperrlisten
- Trifft Zutrittsentscheidungen offline
- Kommuniziert mit Türcontrollern

#### 3.4 Türcontroller und Reader

- Referenzhardware
- OSDP Secure Channel
- Relaissteuerung für Türöffner

#### 3.5 Web3 Proof Layer

- Nur Nachweis
- Keine Echtzeitfunktion

---

## 4. Technologievorgaben

| Bereich | Technologie |
|---------|-------------|
| Frontend | Next.js |
| Backend | Go |
| Edge Gateway | Go, Linux, Containerfähig |
| Datenbank | PostgreSQL, Event Store (append-only) |
| Auth Portal | Auth0 oder Clerk |
| Cloud Frontend | Vercel |
| Cloud Backend | AWS Frankfurt oder GCP EU |
| Blockchain | EVM-kompatibel (Polygon oder Base), Abstraktion über Adapter |

---

## 5. Mandanten- und Datenmodell

### Zentrale Objekte

| Objekt | Beschreibung |
|--------|--------------|
| Mandant | Kundenfirma mit eigener Datenisolation |
| Standort | Physischer Ort mit Türen |
| Tür | Zugangspunkt |
| Gateway | Edge-Gerät pro Standort |
| Controller | Türcontroller (OSDP) |
| Karte | DESFire EV3 Zutrittskarte |
| Mitarbeiter | Person mit Zutrittsberechtigung |
| Gruppe | Berechtigungsgruppe |
| Zeitplan | Gültigkeitsfenster für Zutritt |
| Regelversion | Versionierter Regelsatz |
| Event | Zutrittsereignis |
| Proof Record | Blockchain-verankerter Nachweis |

### Kardinalitäten

- Mandant hat mehrere Standorte
- Standort hat mehrere Türen
- Tür ist einem Controller zugeordnet
- Karte gehört zu genau einem Mandanten
- Karte kann gesperrt oder aktiv sein
- Mitarbeiter kann mehrere Karten haben

---

## 6. Karten- und Sicherheitsmodell

### Kartenstandard

DESFire EV3 oder gleichwertig

### Authentifizierungsprinzip

- Kryptografische Mutual Authentication
- **Keine UID-Authentifizierung**

### Schlüsselmodell

| Aspekt | Vorgabe |
|--------|---------|
| Master Key | Pro Mandant |
| Key Diversification | Pro Karte |
| Key Versioning | Verpflichtend |

### Verantwortung

- Schlüssel liegen **nicht im Klartext** im Portal
- Gateway kennt Key Version und Ableitung
- Portal verwaltet Status und Versionen

---

## 7. Zutrittslogik

### Ablauf Zutritt

```
1. Karte an Leser
2. Controller oder Gateway erzeugt Challenge
3. Karte antwortet kryptografisch
4. Gateway prüft Echtheit
5. Gateway prüft lokal Berechtigung
6. Relais wird geschaltet
7. Event wird lokal gespeichert
```

### Offlinefähigkeit

- Zutritt **muss ohne Internet funktionieren**
- Synchronisation erfolgt nachträglich

---

## 8. Webportal Funktionen MVP

| Funktion | Beschreibung |
|----------|--------------|
| Mandant anlegen | Neue Kundenfirma registrieren |
| Standorte und Türen anlegen | Physische Struktur abbilden |
| Gateways registrieren | Per QR-Code |
| Mitarbeiter anlegen und importieren | Manuell oder CSV-Import |
| Karten zuweisen und sperren | Kartenlebenszyklus verwalten |
| Gruppen und Zeitpläne verwalten | Berechtigungslogik definieren |
| Statusübersicht | Türen und Gateways überwachen |
| Eventanzeige | Zutrittsereignisse einsehen |
| Proof-Exporte | Blockchain-Nachweise exportieren |

### Einschränkung

> **Keine freie Regel-Engine. Nur vordefinierte Templates.**

---

## 9. Edge Gateway Funktionen

| Funktion | Beschreibung |
|----------|--------------|
| Provisioning per QR | Ersteinrichtung via QR-Code |
| Regelsynchronisation | Regeln vom Backend holen |
| Sperrlistenverwaltung | Gesperrte Karten lokal cachen |
| Offline Decision Engine | Zutrittsentscheidung ohne Internet |
| Event Buffering | Events lokal speichern bei Offline |
| Monitoring-Daten liefern | Status an Backend melden |
| Signierte Updates | Nur verifizierte Software installieren |

---

## 10. Türintegration Standard

### MVP Standard: Wired Door Kit

| Komponente | Beschreibung |
|------------|--------------|
| Controller | Mit Relais |
| Reader | OSDP-fähig |
| Protokoll | OSDP Secure Channel (verpflichtend) |

### Wiegand

- Nur als Ausnahme
- **Nicht Standard**

### Integration erfolgt über

- Controller-API, oder
- Definiertes Gateway-Adapter-Modul

---

## 11. Web3 Proof Layer

### Zweck

**Nachweis, nicht Steuerung**

### On-Chain Events

| Event | Beschreibung |
|-------|--------------|
| Karte ausgegeben | CardIssued |
| Karte gesperrt | CardRevoked |
| Regelversion aktiviert | RulesActivated |
| Event Batch Hash | Optional, periodisch |

### Nicht erlaubt

- On-Chain-Transaktion pro Zutritt
- On-Chain-Echtzeitprüfung

---

## 12. Monitoring und Betrieb

| Metrik | Beschreibung |
|--------|--------------|
| Gateway Online Status | Verbindungsstatus |
| Letzte Synchronisation | Zeitstempel |
| Türstatus | Verfügbar / Nicht verfügbar |
| Event Queue Größe | Anzahl gepufferter Events |
| Basisdiagnose Export | JSON/CSV Export |

---

## 13. Abrechnung

### Preismodell

**Preis pro aktiver Karte pro Monat**

### Definition aktive Karte

Karte mit Status "aktiv" im Abrechnungszeitraum

### System muss liefern

- Anzahl aktiver Karten pro Zeitraum
- Export für Abrechnung

---

## 14. Abnahmekriterien MVP

| Nr. | Kriterium |
|-----|-----------|
| 1 | Zutritt funktioniert offline |
| 2 | Gesperrte Karte öffnet keine Tür |
| 3 | Kopierte UID öffnet keine Tür |
| 4 | Regeln werden synchronisiert |
| 5 | Proof-Einträge werden erzeugt |
| 6 | Keine On-Chain-Kosten bei Zutritten |

---

## 15. Offene Entscheidungen für Umsetzung

### Konkretes Referenzhardwareprofil

- Controller-Modell
- OSDP-Reader-Modell

### Integrationsansatz

- Controller-native-API, oder
- Gateway als Logik-Layer

---

## Klartext-Fazit für den Entwickler

> **Baue kein Cloud-gesteuertes Zutrittssystem.**
> 
> **Baue ein lokales, robustes Edge-System, das von einer Plattform verwaltet wird.**
> 
> **Web3 ist Buchhaltung und Notar, nicht Türöffner.**
