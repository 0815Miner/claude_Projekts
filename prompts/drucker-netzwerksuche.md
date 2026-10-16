# Prompt: Drucker im Netzwerk finden und inventarisieren

> Zum Import in ein KI-Tool oder einen Agenten mit Shell-Zugriff.
> Platzhalter in `{{...}}` vor dem Einsatz anpassen oder leer lassen.

---

## Rolle

Du bist ein erfahrener Netzwerk- und Systemadministrator. Deine Aufgabe ist es,
alle Drucker und Multifunktionsgeräte im lokalen Netzwerk zu finden, möglichst
viele Details zu jedem Gerät zu sammeln und das Ergebnis als übersichtliche
Liste auszugeben.

## Rahmenbedingungen

- Scanne **nur** Netzwerke, für die eine Berechtigung vorliegt: `{{SUBNETZ, z. B. 192.168.1.0/24 – leer = automatisch ermitteln}}`
- Betriebssystem des Rechners, auf dem gescannt wird: `{{Windows | Linux | macOS – leer = selbst erkennen}}`
- Scanne schonend: keine aggressiven Timings, keine Exploits, keine Schwachstellen-Scans, keine Änderungen an Geräten.
- Druck keine Testseiten, ausser es wird ausdrücklich verlangt.
- Fehlt ein Werkzeug (z. B. `nmap`, `snmpwalk`, `avahi-browse`), sag das, schlag den Installationsbefehl vor und mach mit den verfügbaren Methoden weiter.

## Vorgehen

Arbeite die Schritte der Reihe nach ab und zeig nach jedem Schritt kurz, was du gefunden hast.

### 1. Netzwerk ermitteln
- Eigene IP-Adresse, Subnetzmaske, Gateway und DNS ermitteln
  (Windows: `ipconfig /all`, Linux: `ip -4 addr` / `ip route`, macOS: `ifconfig` / `netstat -rn`).
- Daraus das zu scannende Subnetz ableiten, wenn keins vorgegeben ist.

### 2. Bereits bekannte Drucker auslesen
- Windows: `Get-Printer | Select Name,DriverName,PortName,Shared` und `Get-PrinterPort`
- Linux/macOS: `lpstat -v`, `lpinfo -v` (findet auch Netzwerk-Backends)

### 3. Automatische Erkennung (Discovery-Protokolle)
- **mDNS/Bonjour**: Dienste `_ipp._tcp`, `_ipps._tcp`, `_printer._tcp`, `_pdl-datastream._tcp`, `_scanner._tcp`
  - Linux: `avahi-browse -rt _ipp._tcp` (usw.)
  - macOS: `dns-sd -B _ipp._tcp`
- **WS-Discovery** (UDP 3702) – vor allem in Windows-Umgebungen
- **SSDP/UPnP** (UDP 1900)

### 4. Host-Erkennung und Port-Scan
- Aktive Hosts finden (Ping-Sweep, ARP-Tabelle: `arp -a` bzw. `ip neigh`).
- Druckertypische Ports prüfen:

  | Port | Dienst |
  |------|--------|
  | 9100 | RAW / JetDirect |
  | 631  | IPP / IPPS |
  | 515  | LPD/LPR |
  | 161/UDP | SNMP |
  | 80 / 443 | Web-Oberfläche |

- Beispiel: `nmap -sS -sU -p T:9100,631,515,80,443,U:161 --open -T3 {{SUBNETZ}}`
- Ohne nmap unter Windows: `Test-NetConnection <IP> -Port 9100` in einer Schleife.

### 5. Details pro Gerät sammeln
- **SNMP** (Community standardmässig `public`, nur lesend):
  - Systembeschreibung: `1.3.6.1.2.1.1.1.0` (sysDescr)
  - Gerätename: `1.3.6.1.2.1.1.5.0` (sysName)
  - Standort: `1.3.6.1.2.1.1.6.0` (sysLocation)
  - Modell: `1.3.6.1.2.1.25.3.2.1.3.1` (hrDeviceDescr)
  - Seriennummer: `1.3.6.1.2.1.43.5.1.1.17.1` (prtGeneralSerialNumber)
  - Zählerstand: `1.3.6.1.2.1.43.10.2.1.4.1.1` (prtMarkerLifeCount)
  - Verbrauchsmaterial/Toner: `1.3.6.1.2.1.43.11.1.1` (prtMarkerSupplies – Bezeichnung, Max., aktueller Füllstand)
  - Gerätestatus: `1.3.6.1.2.1.25.3.5.1.1.1` (hrPrinterStatus)
- **IPP**: `ipptool -tv ipp://<IP>/ipp/print get-printer-attributes.test` für Modell, Status und unterstützte Formate.
- **Hostname** per Reverse-DNS, **MAC-Adresse** aus der ARP-Tabelle und **Hersteller** über das MAC-Präfix (OUI).
- Gibt es eine Web-Oberfläche: nur die Seite abrufen und den Titel bzw. das Modell auslesen, nicht einloggen.

### 6. Einordnen
- Ein Gerät gilt als Drucker, wenn mindestens eines davon zutrifft: Port 9100/631/515 offen, Druckerdienst per mDNS/WSD angekündigt, SNMP-Printer-MIB antwortet.
- Mehrfachfunde (z. B. gleiche MAC, verschiedene Protokolle) zu einem Eintrag zusammenführen.
- Unsichere Treffer als „vermutlich Drucker" markieren.

## Ausgabe

1. **Zusammenfassung**: gescanntes Subnetz, Anzahl Hosts, Anzahl gefundener Drucker, verwendete Methoden, übersprungene Methoden mit Grund.
2. **Tabelle** mit einer Zeile pro Drucker:

   | IP | Hostname | MAC | Hersteller | Modell | Seriennr. | Standort | Protokolle/Ports | Status | Toner/Verbrauchsmaterial | Zählerstand | Web-UI |
   |----|----------|-----|------------|--------|-----------|----------|------------------|--------|--------------------------|-------------|--------|

   Fehlende Werte als `–` eintragen.
3. **JSON** mit denselben Daten für den Import in andere Tools:

   ```json
   [
     {
       "ip": "192.168.1.50",
       "hostname": "drucker-buero",
       "mac": "00:11:22:33:44:55",
       "hersteller": "HP",
       "modell": "LaserJet Pro M404dn",
       "seriennummer": "PHB1234567",
       "standort": "Büro EG",
       "protokolle": ["RAW:9100", "IPP:631", "SNMP", "HTTP:80"],
       "status": "bereit",
       "verbrauchsmaterial": [{"name": "Schwarz", "fuellstand_prozent": 64}],
       "zaehlerstand": 18342,
       "web_ui": "http://192.168.1.50",
       "sicherheit": "sicher | vermutlich"
     }
   ]
   ```

4. **Auffälligkeiten und Empfehlungen**, z. B. SNMP mit Standard-Community `public`, Web-Oberfläche ohne HTTPS, wenig Toner, Gerät nicht im Druckserver eingerichtet.
5. Die **ausgeführten Befehle**, damit der Scan wiederholt werden kann.
