# Funkprüfung und Garagen-Pilot – 15. Oktober 2026

## Ergebnis und Geltungsbereich

Der Zähler-Hub bleibt an seiner bisherigen Aufgabe und Adresse. Die Änderung
entfernt nur die alten 1100-ms-Scanvorgaben zugunsten der versionsabhängigen
ESPHome-Standards, erhält den passiven Betrieb und ergänzt RSSI/BSSID.
Der manuelle Scanbutton erhält den Dauerscan. Kein WLAN-AP/NAT auf dem Zähler-Hub.
Im verschlüsselten Migrationseinstieg wird zusätzlich das funktionslose Captive
Portal entfernt und HTTP-OTA deaktiviert: Das Portal lädt sonst Web-OTA nach,
obwohl dessen expliziter Eintrag entfernt wurde. Die Passwortbrücke bleibt intakt.

Grundlage: vollständig gelesene Repository-Konfigurationen auf main
`5d75e81d0c8e58f80a13bf8205feced67e8b55a7`, README und OTA-Anleitung.
Es gibt genau zwei Firmware-Einstiege: `espsmarthomehub,yaml` sowie das
inkludierende `espsmarthomehub-encrypted-ota.yaml`. Keine Konfiguration der
anderen beiden Proxies ist in diesem Repository enthalten.

Die frühere private Netzplanung (Version 8, geändert 13.10.) wurde gelesen.
Ihre Livebefunde sind vom **09.10.**, keine Messungen vom 15.10.
Ein Desktop-Aufruf lieferte am 15.10. nur einen MCP-Timeout; daraus folgt weder
ein heutiger LAN-Zustand noch eine Geräte-/Firmwareänderung. HA-Livewerte,
lokal installierte Proxy-YAMLs und aktuelle FRITZ!-Assoziationen wurden hier
nicht abgerufen. Alle nachfolgenden Hardwarewirkungen sind **unbestätigt**.

## BLE-Scan und Ressourcen

- 1100/1100 ms ergibt rechnerisch 100 % angefordertes Scanfenster.
  Das sind weder 100 % CPU noch ein Beleg für durchgehend belegte Funkzeit:
  WLAN und BLE teilen sich unter ESP-IDF ein Funkteil und dessen Arbiter.
- Der ESPHome-2026.9.1-Quellcode warnt ausdrücklich vor Fenstern über 600 ms
  bei WLAN. Diese Konfiguration trifft die Warnbedingung. Das belegt ein
  Konfigurationsrisiko am Hub, keine Ursache der Garagenabbrüche.
- Standardintervall ist 320 ms. Bei WLAN, aktivierter Software-Koexistenz
  und ESP-IDF >=5.5.5 wird ein nicht explizit gesetztes Fenster auf das
  Intervall angehoben. Sonst ist es 30 ms. Deshalb beide alten Zahlen
  entfernen und den tatsächlich aufgelösten Wert aus dem Buildprotokoll lesen.
  Nicht pauschal 30/320 ms erzwingen: der IDF-Koexistenzfix änderte die Wirkung.
- `scan_parameters.active: false` verhindert aktive Scan Requests;
  `bluetooth_proxy.active: false` verhindert aktive GATT-Verbindungen.
  Die beiden Optionen sind unabhängig. Passives Weiterreichen bleibt erhalten;
  der Hub ist damit kein aktiver SolarVault-GATT-Proxy.
- Der bisherige Button setzt `continuous: false`. Upstream
  `ESP32BLEStartScanAction::play()` überschreibt damit den Dauerscanstatus;
  nach Scanende startet die Schleife nicht erneut. `continuous: true`
  erhält die vorgesehene passive Dauererfassung.
- BLE-Stack und Webserver benötigen RAM. Ohne Heap-/Loop-/Reconnectmessung
  ist keine konkrete Überlastung belegt. Webserver/Recovery und Logger werden
  nicht vorsorglich entfernt. WLAN-Sendeleistung und Power-Save bleiben
  unverändert, um nicht mehrere Funkvariablen gleichzeitig zu verändern.

## Bestand: dokumentiert versus heute gemessen

| Knoten | Dokumentierter Stand | Entscheidung |
|---|---|---|
| Hub .40, Flur EG | Repository: esp32dev/ESP-IDF, SML GPIO16/9600 8N1; Impulse GPIO18; passive BLE-Weitergabe. Früher -63 dBm | Zählerstandort, Identität und Energiepfade erhalten; kein S3-Tausch für diesen Pilot |
| Proxy .46, Wohnzimmer EG | Netzplan: WROOM/BLE-Proxy, früher -63 dBm | Bestehende Funktion erhalten; erster Standort-/Board-Vergleichskandidat mit separatem Reserveboard |
| Proxy .47, Schlafzimmer DG | Netzplan: WROOM/BLE-Proxy, früher -39 dBm | Behalten; kein belegter Austauschbedarf |
| Shelly Plus 1 .72, Garage 2 | Wiederholte WLAN-Abbrüche vom Nutzer berichtet; 09.10. -85 dBm, drei laufende Scripts dokumentiert | Eigentlichen WLAN-Pfad verbessern; Torimpuls/Auto-off/Taster lokal erhalten |
| Zwei Tuya-Kontakte | Netzplan identifiziert Verpackungen C30-ZY_D02_1PCS als WiFi Version; physische HA-ID-Zuordnung nicht bestätigt | Kontakt A Garage 1, B Garage 2 als Ziel; einzeln zuordnen, WLAN/App/HA-Ereignisweg testen |
| Vorhandene Gen2/Gen3-Shellys | Mehrere native Extender-Kandidaten; blauer Mini ggf. bereits LichtWZ | Kein bereits belegtes Gerät als freie Reserve umwidmen; nur passendes freies/versorgtes Gerät verwenden |
| Repeater 6000 | Nutzerbestätigung vom 15.10.: Auch am Fenster getestet, ohne besseren WLAN-Empfang zur Garage. Damaliger Diagnoseabruf HTTP 500; Uplinkmodus ungeklärt | Fensterversuch als erfolglos abgeschlossen übernehmen; nicht erneut als Lösung planen. HTTP 500 beweist keinen Funk-/Hardwaredefekt |

Die aktuelle Repository-YAML hat DG-Impulszählung, während ältere Nutzerentscheidungen
auch einen späteren DG-SML-Zähler nennen. Repository, installierte Firmware und
physische Hardware sind deshalb nicht gleichgesetzt. Keine Zählerumstellung aus
dieser Funkprüfung ableiten. DG-Richtungsheuristik ist eine Schätzung, kein
gemessenes Vorzeichen; ihre fachliche Abnahme gehört zur Zählerprüfung.

## WROOM, S3 und ESP8266

| Fähigkeit | ESP32-WROOM-32D | ESP32-S3 | ESP8266 |
|---|---|---|---|
| WLAN-Client | 2,4 GHz 802.11 b/g/n | 2,4 GHz 802.11 b/g/n | WLAN-Client, kein BLE |
| ESPHome BLE-Proxy | Advertisements; aktive GATT-Verbindungen mit entsprechender Proxy-Konfiguration | Ebenfalls unterstützt; exaktes Board/Flash/PSRAM prüfen | Nicht möglich |
| WLAN-AP | Mit geeigneter Firmware; ESPHome-Fallback-AP allein ist keine Weiterleitung | Gleiches Prinzip | AP-Fähigkeit allein ist kein Repeaternachweis |
| WLAN-Weiterleitung | Separate NAT-/Bridge-Firmware erforderlich | esp32_nat_router veröffentlicht S3-Binaries; separates Subnetz/NAT | Für diesen Auftrag kein geprüfter Firmwarepfad |
| Reichweite | PCB-Antenne am 32D | Abhängig vom konkreten Modul, Antenne, Gehäuse, Versorgung und Ort | Kein sinnvoller BLE-Ersatz |

S3-Rechenleistung beweist keine bessere Reichweite. BLE 5 bedeutet nicht, dass
vorhandene Sensoren automatisch Coded PHY/Long Range verwenden. Ein S3 ist weder
ein Thread-/Zigbee-Ersatz noch automatisch WLAN-Mesh. Ein fairer Vergleich:
WROOM am bisherigen Ort, Reserve-S3 am gleichen Ort, danach derselbe Pilot am
besseren Ort; Geräte-/Anwendungserfolg und nicht nur ESP-RSSI vergleichen.

## Konkreter WLAN-Pilot Richtung Garage

**Der Fensterversuch mit dem vorhandenen Repeater ist bereits erfolglos getestet.**
Quelle: direkte Nutzerkorrektur vom 15.10.2026. Auch am Fenster verbessert sich der
WLAN-Empfang zur Garage nicht. Dieser Versuch wird nicht erneut eingeplant.
Konkrete RSSI-, Verlust- oder Latenzwerte wurden dabei nicht übermittelt;
der bestätigte fehlende Nutzen wird trotzdem als Befund übernommen.

Der nächste Pilot muss sich in Funkpfad, Standort oder Uplink tatsächlich ändern:
ein AP näher an den Garagen mit nachgewiesen brauchbarem Uplink. Ein LAN-Uplink
ist eine mögliche Ausführung, sofern ein Kabelweg verfügbar ist; dieser ist
noch nicht belegt. Eine weitere WLAN-Strecke braucht eine gemessene brauchbare
Verbindung und darf nicht allein wegen der kurzen Luftlinie als geeignet gelten.
Den bestehenden Repeater-Uplink nur zur Einordnung des bisherigen Pfads auslesen;
daraus wird keine Wiederholung des bereits erfolglosen Fenstertests abgeleitet.

Ein S3 am gleichen Fenster ist ebenfalls keine bereits belegte Lösung.
Sein Vergleich wäre ein gesonderter Hardware-/Antennenversuch mit offenem Ausgang.
Bei unzureichendem Uplink wird der NAT-Pilot an diesem Ort verworfen.

Mit dem vorhandenen Bestand ist ein freies USB-S3-Board der bewegliche
**WLAN-only-NAT-Pilot**. Keine BLE-, Voice- oder CSI-Aufgabe gleichzeitig.
SSID des Pilot-AP eindeutig und WPA2-geschützt; eigenes, nicht überlappendes
Subnetz; DHCP/DNS/Internetzugang und Rückweg zu HA ausdrücklich prüfen.
Boardaufschrift, Flashgröße und Antenne müssen vor Auswahl/Flash des S3-Targets
feststehen. Es wird deshalb keine geraten board-spezifische Flashanweisung ausgeführt.

Ein geeigneter freier Gen2/Gen3-Shelly ist eine Alternative mit nativem Range
Extender. HA dokumentiert den nachgelagerten Gen2+-Shelly über feste/reservierte
Hauptgeräte-IP plus tatsächlich angezeigten Client-TCP-Port. Die Shelly-Dokumentation
belegt den Shelly-Pfad, aber keine Zusage speziell für diese Tuya-Kontakte.
Deren Verbindung und Ereigniszustellung sind auch dort ein eigener Test.

| Datenweg | Abnahme |
|---|---|
| Tuya A/B → Pilot-AP → DNS/Internet → bestehende Tuya-Anbindung → HA | Beide Kontakte einzeln; App und HA müssen dieselben Magnetwechsel mit korrekter Polarität liefern |
| Shelly .72 → Pilot-AP → NAT/Weiterleitung → native HA-Integration | HTTP-RPC und WebSocket-Ereignisse aus dem HA-Netz prüfen; keine ausgehende Cloudverbindung als Beweis für lokalen Rückweg verwenden |
| BLE-Sensor → .46/.47 oder später separater S3-BLE-Proxy → ESPHome-API → HA | Datenfrische, Empfangsquelle und ggf. GATT-Verbindung separat nachweisen |
| Hub .40 → bestehendes WLAN → HA | SML-/Impuls-/Solarpfad bleibt vollständig erhalten |

Bei S3-NAT nicht blind Shelly-Extender-Portschemata übernehmen. mDNS und
automatische Discovery sind über Subnetzgrenzen nicht vorausgesetzt.
S3-SoftAP mit Clients plus BLE ist laut Espressif eine unterstützte, aber
leistungsinstabile Koexistenzkombination; daher getrennte Geräte/Rollen.
Kein Tor wird für Funkmessungen automatisch betätigt.

## Messung und Abnahme

1. Ausgangsstand mit Uhrzeit, Firmware, MAC/IP, AP-BSSID/Kanal, Standort und
   Torstellung sichern. Shelly `WiFi.GetStatus` und `Sys.GetStatus`
   ausschließlich lesend erfassen; AP-Clientliste am Router/Extender gegenprüfen.
2. Pro Variante 30 Minuten gleiche Bedingungen: ICMP-Verlust/RTT für dauerhaft
   aktive Knoten und lesende RPC-Erfolgsquote/Antwortzeit am Shelly, RSSI-Verlauf,
   Uptime und Reconnects. Nicht-ICMP-Antwort als ungeklärt behandeln, nicht
   automatisch als WLAN-Paketverlust. Batteriekontakte nicht per Dauerping bewerten.
3. Je Tuya-Kontakt 50 kontrollierte Magnetzustandswechsel ohne Torfahrt, im
   guten Referenz-WLAN und am vorgesehenen Tor bei geschlossenem Metalltor.
   Sendeabstand gemäß tatsächlichem Wake-/Entprellverhalten wählen; Zeitpunkte
   und Zustände lokal/App/HA zählen, fehlende oder verzögerte Meldungen erfassen.
4. Planungsziel: auf beiden WLAN-Hops möglichst etwa -67 dBm oder besser;
   kein Herstellergrenzwert und keine Garantie. Abnahme verlangt zusätzlich
   vollständige Zustellung der Testereignisse, erreichbare HA-RPC-/Eventroute
   und keine unerklärten Abbrüche/Neustarts im Messfenster.
5. Danach sieben Tage vergleichbare Langzeitbeobachtung ohne neue unbekannte
   HA-Verfügbarkeitslücken oder Energieunterbrechungen. Diese Beobachtung wurde
   hier **nicht gestartet**. Bestehende Haus-/Kellerclients mitbewerten.

## Sichere Auslieferung und Wiederherstellung

Vor Upload: tatsächliche installierte YAML und Firmwareversion sichern; sie sind
nicht durch main bewiesen. Aktuelle Geräteidentität/MAC/IP, API-Key/OTA-Verfahren,
vollständiges Flash-/NVS-Backup und bekannte funktionierende Firmware erhalten.
Kein erase-flash, keine Umbenennung, keine neue MAC, keine Partitionsmigration.
USB-Zugang sicherstellen. Ein altes komplettes Flashbackup kann bei späterem
Restore auch Energieakkumulatoren zurücksetzen: vor/nach Rückkehr Werte abgleichen.

Beide Repository-Einstiege zuerst mit synthetischen Secrets validieren und
kompilieren; CI verwendet aktuelles stabiles ESPHome ohne Paketpin.
Die dort erzeugte Firmware hat Testzugangsdaten und darf **nicht geflasht** werden.
Windows: `py -3.14 -m esphome config "espsmarthomehub,yaml"`;
Ubuntu: `python3.14 -m esphome config "espsmarthomehub,yaml"`.
Danach mit `compile` und der gleichen Datei; für den verschlüsselten Einstieg
entsprechend `espsmarthomehub-encrypted-ota.yaml`. Reale Secrets bleiben lokal.

Das Captive Portal ist ohne Fallback-AP bereits im Ausgangsstand unerreichbar;
es ist kein nachgewiesener Recovery-Weg. Nur im verschlüsselten Einstieg wird es
entfernt, damit seine Auto-Load-Abhängigkeit HTTP-OTA nicht erneut aktiviert.
Die Web-Statusoberfläche bleibt dort erhalten, mit explizitem `ota: false`.
Im aufgelösten verschlüsselten Build darf kein `platform: web_server` unter
`ota` und kein `captive_portal` mehr enthalten sein; der `hub_ota`-Eintrag
muss weiterhin Verschlüsselung mit dem unveränderten API-Key verwenden.

Die OTA-Migrationsanleitung bleibt gültig: Passwortbrücke zuerst, verschlüsselter
Einstieg erst nach bestätigtem erfolgreichen Übergang. Kein Firmwareupload in
dieser Untersuchung. Firmwaretests belegen weder Funkwirkung noch Recovery.

Vor/nach späterem Update: SML-Werte, Impulsfortschritt, HA-Solarquellen,
kumulative Energie (kein unerklärter Reset), Entitätsidentitäten und BLE-Frische
prüfen. Beim manuellen BT-Start muss der Scanner über mindestens zwei komplette
Scanperioden weiterlaufen. Bei Regression auf gesicherte Firmware zurückkehren,
ohne NVS zu löschen. Bei minutlicher Flashsicherung ist Verlust noch nicht
gesicherter Akkumulation bei Stromausfall möglich; kein Nullverlustversprechen.

## Primärquellen

- [ESPHome Tracker](https://esphome.io/components/esp32_ble_tracker/)
- [Tracker-Validierung 2026.9.1](https://github.com/esphome/esphome/blob/2026.9.1/esphome/components/esp32_ble_tracker/__init__.py): Koexistenz, 600-ms-Warnung und bedingte Standards
- [Scan-Action 2026.9.1](https://github.com/esphome/esphome/blob/2026.9.1/esphome/components/esp32_ble_tracker/automation.h): set_scan_continuous
- [Captive-Portal Auto-Load](https://github.com/esphome/esphome/blob/2026.9.1/esphome/components/captive_portal/__init__.py)
- [Webserver OTA-Schalter](https://github.com/esphome/esphome/blob/2026.9.1/esphome/components/web_server/__init__.py)
- [Bluetooth-Proxy](https://esphome.io/components/bluetooth_proxy/)
- [S3-Proxy-Beispiel](https://github.com/esphome/bluetooth-proxies/blob/main/m5stack/m5stack-atom-s3.yaml)
- [WLAN](https://esphome.io/components/wifi/)
- [RSSI](https://esphome.io/components/sensor/wifi_signal/)
- [BSSID](https://esphome.io/components/text_sensor/wifi_info/)
- [S3-Koexistenz](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/coexist.html)
- [WROOM-32D Datenblatt](https://www.espressif.com/sites/default/files/documentation/esp32-wroom-32d_esp32-wroom-32u_datasheet_en.pdf)
- [S3-Eigenschaften](https://www.espressif.com/en/products/socs/esp32-s3)
- [S3-NAT-Firmware](https://github.com/martin-ger/esp32_nat_router)
- [Shelly WLAN/Extender](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/WiFi/)
- [HA Shelly: Extender und BLE](https://www.home-assistant.io/integrations/shelly/)
