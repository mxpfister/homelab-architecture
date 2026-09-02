# ⚙️ Homelab (Standort Würzburg)

## 🏗️ 1. System- & Container-Architektur

```mermaid
graph TB
    %% Styling
    classDef pveNode fill:#34495e,stroke:#2c3e50,stroke-width:2px,color:#fff;
    classDef vmStyle fill:#2980b9,stroke:#1c5980,stroke-width:2px,color:#fff;
    classDef lxcStyle fill:#27ae60,stroke:#1e7e45,stroke-width:2px,color:#fff;
    classDef dockerStyle fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
    classDef storageStyle fill:#7f8c8d,stroke:#34495e,stroke-width:2px,color:#fff;
    classDef extStyle fill:#8e44ad,stroke:#2c3e50,stroke-dasharray: 5 5,color:#fff;

    subgraph PVE ["💻 Proxmox Node (pve) [Bluechip BusinessLine NUC8BEK CPU i5-8259U RAM 16GB SSD 256GB]"]
        
        subgraph Storage ["💾 Speicher-Infrastruktur"]
            SSD["🟪 Externe SSD (256GB)<br>Pfad: /mnt/pve/ext-storage/dump"]:::storageStyle
        end

        subgraph VMs ["🖥️ Virtuelle Maschinen (VMs)"]
            HA["🔹 Home Assistant OS"]:::vmStyle
        end

        subgraph LXCs ["📦 LXC Container"]
            Plex["🟢 Plex Media Server"]:::lxcStyle
            Pihole["🟢 Pi-hole (DNS)"]:::lxcStyle
            Paperless["🟢 Paperless NGX"]:::lxcStyle
            
            subgraph DockerLXC ["🐋 Docker-Host (LXC)"]
                Portainer["🔸 Portainer"]:::dockerStyle
                Influx["🔸 InfluxDB"]:::dockerStyle
                Uptime["🔸 Uptime Kuma"]:::dockerStyle
                NPM["🔸 Nginx Proxy Manager"]:::dockerStyle
                Sponsor["🔸 iSponsorBlockTV"]:::dockerStyle
            end
            class DockerLXC lxcStyle;
        end
    end

    %% Externe Verbindungen
    GDrivePVE["☁️ Google Drive (Proxmox_Wue_Backups)"]:::extStyle
    GDrivePL["☁️ Google Drive (Paperless_Backup)"]:::extStyle
    
    %% Datenpfade
    VMs -->|03:00 vzdump| SSD
    LXCs -->|03:00 vzdump| SSD
    SSD -->|04:00 rclone sync| GDrivePVE
    Paperless -->|02:00 rclone sync| GDrivePL
    
    class PVE pveNode;
```

## 🕓 2. Cronjobs

1. 02:00 Uhr: Paperless Document Export & Cloud-Sync (Paperless-LXC) $\rightarrow$ Erstellt einen konsistenten Dokumenten-/Datenbankexport und synchronisiert ihn via Rclone zu Google Drive
2. 03:00 Uhr: Proxmox VZDump Backup (pve-Node) $\rightarrow$ Sichert alle VMs und LXC-Container als Backup-Dump auf die externe SSD
3. 04:00 Uhr: Proxmox Rclone Backup (pve-Node) $\rightarrow$ Synchronisiert lokale Dumps zu Google Drive.
4. 04:00 Uhr (Sonntags): yt-dlp Auto-Update (Plex-LXC) $\rightarrow$ Aktualisiert den yt-dlp auf die neueste Version.
5. 05:15 Uhr: Plex Leere Ordner löschen (Plex-LXC, yt-worker) $\rightarrow$ Entfernt verwaiste Verzeichnisse.
6. 05:30 Uhr: Plex Smart Cleanup (Plex-LXC, yt-worker) $\rightarrow$ Löscht gesehene YouTube-Videos nach einer Frist von 5 Tagen.

```mermaid
sequenceDiagram
    autonumber
    participant HA as 🔹 Home Assistant (VM)
    participant PVE as 💻 Proxmox Host (pve)
    participant PaperlessLXC as 🟢 Paperless LXC
    participant PlexLXC as 🟢 Plex LXC
    participant Influx as 🔸 InfluxDB (Docker)
    participant SSD as 🟪 Externe SSD
    participant GDrive as ☁️ Google Drive

    %% --- ABLAUF 1: PAPERLESS EXPORT & SYNC (02:00 UHR) ---
    rect rgb(39, 174, 96)
        Note over PaperlessLXC, GDrive: ABLAUF 1: Paperless Backup & Cloud Sync (02:00 Uhr)
        Note over PaperlessLXC: Crontab startet cron-monitor.sh
        activate PaperlessLXC
        PaperlessLXC->>PaperlessLXC: 1a. document_exporter schreibt Export nach /opt/paperless/export
        PaperlessLXC->>GDrive: 1b. rclone sync nach gdrive:Paperless_Backup
        PaperlessLXC->>Influx: 1c. cron-monitor.sh meldet duration & exit_code
        deactivate PaperlessLXC
    end

    %% --- ABLAUF 2: PROXMOX VZDUMP BACKUP (03:00 UHR) ---
    rect rgb(52, 73, 94)
        Note over PVE, SSD: ABLAUF 2: Proxmox Node Backup (03:00 Uhr)
        Note over PVE: Integrierter PVE-Backup-Job (vzdump)
        activate PVE
        PVE->>SSD: 2a. Sichert alle VMs & LXCs als Dump-Dateien (/mnt/pve/ext-storage/dump)
        deactivate PVE
    end

    %% --- ABLAUF 3: PROXMOX RCLONE CLOUD SYNC (04:00 UHR) ---
    rect rgb(46, 204, 113)
        Note over PVE, GDrive: ABLAUF 3: Proxmox Rclone Backup (04:00 Uhr)
        Note over PVE: Lokale System-Crontab löst Backup aus
        activate PVE
        PVE->>GDrive: 3a. rclone sync von externer SSD nach GDrive
        GDrive-->>PVE: 3b. Sync erfolgreich beendet
        PVE->>Influx: 3c. cron-monitor.sh sendet duration & exit_code
        deactivate PVE
    end

    %% --- ABLAUF 4: YT-DLP UPDATE (04:00 UHR, SONNTAGS) ---
    rect rgb(230, 126, 34)
        Note over PlexLXC: ABLAUF 4: yt-dlp Auto-Update (04:00 Uhr, Sonntags)
        Note over PlexLXC: Lokale Crontab (root/yt-worker)
        activate PlexLXC
        PlexLXC->>PlexLXC: 4a. yt-dlp -U
        deactivate PlexLXC
    end

    %% --- ABLAUF 5: PLEX LEERE ORDNER LÖSCHEN (05:15 UHR) ---
    rect rgb(44, 62, 80)
        Note over PlexLXC, Influx: ABLAUF 5: Plex Leere Ordner löschen (05:15 Uhr)
        Note over PlexLXC: Lokale Crontab (Benutzer: yt-worker)
        activate PlexLXC
        PlexLXC->>PlexLXC: 5a. find-Skript räumt leere Verzeichnisse auf
        PlexLXC->>Influx: 5b. cron-monitor.sh meldet Werte an InfluxDB
        deactivate PlexLXC
    end

    %% --- ABLAUF 6: PLEX SMART CLEANUP (05:30 UHR) ---
    rect rgb(142, 68, 173)
        Note over PlexLXC, HA: ABLAUF 6: Plex Smart Cleanup (05:30 Uhr)
        Note over PlexLXC: Lokale Crontab (Benutzer: yt-worker)
        activate PlexLXC
        PlexLXC->>PlexLXC: 6a. plex_cleaner.py löscht geschaute Videos (>5 Tage)
        PlexLXC->>Influx: 6b. cron-monitor.sh meldet Werte an InfluxDB
        PlexLXC->>HA: 6c. Python sendet Webhook an HA (Titel-Liste für iPhone)
        deactivate PlexLXC
    end

    %% --- ABFRAGE-SCHLEIFE (DEAD-MAN-SWITCH) ---
    rect rgb(41, 128, 185)
        Note over HA, Influx: KONTINUIERLICHE ÜBERWACHUNG (Dead-Man-Switch)
        loop Alle paar Minuten
            HA->>Influx: 7a. Flux Query (Prüfe letzten Eintrag im Fenster -26h)
            Influx-->>HA: 7b. Liefert Daten für ALLE Jobs (Dauer & Status)
        end
    end
```

⚠️ Wichtiger Hinweis zum Linux-Pipe-Buffer: Bei Aufrufen über die Home Assistant SSH-Schnittstelle müssen textintensive Skripte zwingend mittels > /dev/null 2>&1 umgeleitet werden. Andernfalls läuft der 64 KB große OS-Pipe-Buffer voll, was zu einem Deadlock (Einfrieren) des Skripts führt.

## 🏠 3. Home Assistant
```mermaid
    graph TD
        classDef core fill:#03a9f4,stroke:#0288d1,stroke-width:2px,color:#fff;
        classDef hw fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
        classDef net fill:#2980b9,stroke:#1c5980,stroke-width:2px,color:#fff;
        classDef cloud fill:#8e44ad,stroke:#2c3e50,stroke-dasharray: 5 5,color:#fff;
        classDef area fill:#27ae60,stroke:#1e7e45,stroke-width:2px,color:#fff;
        classDef broker fill:#34495e,stroke:#2c3e50,stroke-width:2px,color:#fff;

        HA["🔹 Home Assistant OS"]:::core

        subgraph Areas ["🏠 Wohnbereiche (Areas)"]
            direction LR
            Bad["🚿 Bad"]:::area
            Flur["🚪 Flur"]:::area
            Kueche["🍳 Küche"]:::area
            Schlafzimmer["🛏️ Schlafzimmer"]:::area
            Wohnzimmer["🛋️ Wohnzimmer"]:::area
            HomelabArea["⚙️ Homelab"]:::area
        end

        subgraph IoT ["📻 IoT Gateways & Broker"]
            MQTT["Mosquitto Broker<br>(MQTT Integration)"]:::broker
            SLZB["🐝 SLZB-06 Coordinator<br>(Zigbee über LAN via SMLight)"]:::hw
            OEP["🏷️ OpenEPaperLink AP<br>(E-Ink Displays, z.B. Kühlschrank)"]:::hw
        end

        subgraph LocalNet ["🖥️ Lokale Dienste (LAN)"]
            Fritz["📡 FRITZ!Box 6670 Cable<br>(Netzwerk & Internet)"]:::net
            PVE["💻 Proxmox VE Node<br>(Monitoring)"]:::net
            LXC_Docker["📦 LXC & Docker Dienste<br>(Plex, Pi-hole v6, InfluxDB, Portainer)"]:::net
            ATV["📺 Apple TVs<br>(Schlafzimmer, Wohnzimmer)"]:::net
            Appliance["🧺 Haushaltsgeräte<br>(Waschmaschine, Spülmaschine)"]:::net
        end

        subgraph Cloud ["☁️ Cloud & Externe Dienste"]
            G_AI["🤖 Google Generative AI<br>(Assistenz & Conversation)"]:::cloud
            G_Drive["☁️ Google Drive<br>(Home Assistant Backups)"]:::cloud
            Mobile["📱 Companion Apps<br>(iPhones, iPads, MacBook Air)"]:::cloud
            Ext["🌱 Externe APIs<br>(OpenPlantBook, Wetter, DWD)"]:::cloud
        end

        %% Verbindungen
        HA --- Areas
        
        SLZB -->|Zigbee Events| MQTT
        MQTT <-->|MQTT| HA
        OEP <-->|WLAN / REST| HA
        
        HA <-->|API / Integration| LocalNet
        HA <-->|API / Webhooks| Cloud
```

# Homelab (Standort Leinach)
## 🏗️ 1. System- & Container-Architektur

```mermaid
graph TB
    %% Styling
    classDef pveNode fill:#34495e,stroke:#2c3e50,stroke-width:2px,color:#fff;
    classDef vmStyle fill:#2980b9,stroke:#1c5980,stroke-width:2px,color:#fff;
    classDef lxcStyle fill:#27ae60,stroke:#1e7e45,stroke-width:2px,color:#fff;
    classDef dockerStyle fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
    classDef storageStyle fill:#7f8c8d,stroke:#34495e,stroke-width:2px,color:#fff;
    classDef extStyle fill:#8e44ad,stroke:#2c3e50,stroke-dasharray: 5 5,color:#fff;

    subgraph PVE_LEI ["💻 Proxmox Node (pve) [Lenovo ThinkCentre M720q Mini PC Pentium Gold 5400T 8GB RAM 256GB SSD]"]
        
        subgraph Storage_LEI ["💾 Speicher-Infrastruktur"]
            HDD_LEI["🟪 Externe HDD (500GB)<br>Pfad: /mnt/pve/external-storage/dump"]:::storageStyle
        end

        subgraph VMs_LEI ["🖥️ Virtuelle Maschinen (VMs)"]
            HA_LEI["🔹 Home Assistant VM"]:::vmStyle
        end

        subgraph LXCs_LEI ["📦 LXC Container"]
            subgraph DockerLXC_LEI ["🐋 Docker-Host (LXC)"]
                Portainer_LEI["🔸 Portainer"]:::dockerStyle
                Influx_LEI_LEI["🔸 Portainer"]:::dockerStyle
                Uptime_LEI["🔸 Uptime Kuma"]:::dockerStyle
            end
            class DockerLXC_LEI lxcStyle;
        end
    end

    %% Externe Verbindungen
    GDrive_LEI["☁️ Google Drive (Remote)"]:::extStyle
    
    %% Datenpfade
    VMs_LEI -->|03:00 vzdump| HDD_LEI
    LXCs_LEI -->|03:00 vzdump| HDD_LEI
    HDD_LEI -->|04:00 rclone sync| GDrive_LEI
    
    class PVE_LEI pveNode;
```
## 🕓 2. Cronjobs

1. 03:00 Uhr: Proxmox VZDump Backup (pve-Node) $\rightarrow$ Sichert alle VMs und LXC-Container als Backup-Dump auf die externe SSD
2. 04:00 Uhr: Proxmox Rclone Backup (pve-Node) $\rightarrow$ Synchronisiert lokale Dumps zu Google Drive.

```mermaid
sequenceDiagram
    autonumber
    participant PVE as 💻 Proxmox Host (pve)
    participant HDD as 🟪 Externe HDD
    participant GDrive as ☁️ Google Drive
    participant Influx as 🔸 InfluxDB (Docker)

    %% --- ABLAUF 1: PROXMOX VZDUMP BACKUP (03:00 UHR) ---
    rect rgb(52, 73, 94)
        Note over PVE, HDD: ABLAUF 1: Proxmox Node Backup (03:00 Uhr)
        Note over PVE: Integrierter PVE-Backup-Job (vzdump)
        activate PVE
        PVE->>HDD: 1a. Sichert Home Assistant VM & Docker LXC als Dump-Dateien
        deactivate PVE
    end

    %% --- ABLAUF 2: PROXMOX RCLONE CLOUD SYNC (04:00 UHR) ---
    rect rgb(46, 204, 113)
        Note over PVE, GDrive: ABLAUF 2: Proxmox Rclone Backup (04:00 Uhr)
        Note over PVE: Lokale System-Crontab löst Backup aus
        activate PVE
        PVE->>GDrive: 2a. rclone sync von externer HDD nach Google Drive
        GDrive-->>PVE: 2b. Sync erfolgreich beendet
        PVE->>Influx: 2c. cron-monitor.sh sendet duration & exit_code via curl
        deactivate PVE
    end
```

## 🏠 3. Home Assistant
```mermaid
    graph TD
        classDef core fill:#03a9f4,stroke:#0288d1,stroke-width:2px,color:#fff;
        classDef hw fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
        classDef net fill:#2980b9,stroke:#1c5980,stroke-width:2px,color:#fff;
        classDef cloud fill:#8e44ad,stroke:#2c3e50,stroke-dasharray: 5 5,color:#fff;
        classDef area fill:#27ae60,stroke:#1e7e45,stroke-width:2px,color:#fff;
    
        HA["🔹 Home Assistant OS"]:::core
    
        subgraph Floors ["🏠 Etagen & Wohnbereiche"]
            direction TB
            subgraph F_1 ["1. Stock"]
                Bad["🚿 Bad"]:::area
                MaxZimmer["👤 Max Zimmer"]:::area
                Sonstiges["🌡️ Sonstiges"]:::area
            end
            subgraph F_0 ["Erdgeschoss"]
                Flur["🚪 Flur"]:::area
                Kueche["🍳 Küche"]:::area
                Wohnzimmer["🛋️ Wohnzimmer"]:::area
            end
            subgraph F_K ["Keller"]
                KellerArea["🚪 Eingänge"]:::area
            end
            subgraph F_Out ["Außen & Sonstiges"]
                Garten["🌳 Garten"]:::area
                HomelabArea["⚙️ Homelab"]:::area
            end
        end

        subgraph IoT ["📻 IoT Gateways & Dongles"]
            SkyConnect["📡 HA SkyConnect<br>(OpenThread Border Router)"]:::hw
            TP_Link["🔌 TP-Link Kasa<br>(WLAN Steckdosen)"]:::hw
            Tuya["☁️ Tuya Integration"]:::hw
        end

        subgraph LocalNet ["🖥️ Lokale Dienste & Geräte (LAN)"]
            Fritz["📡 FRITZ!Box 7490<br>(Netzwerk & Internet)"]:::net
            PVE["💻 Proxmox VE Node"]:::net
            PS5["🎮 PlayStation 5<br>(Network Integration)"]:::net
        end

        subgraph Cloud ["☁️ Cloud & Externe Dienste"]
            G_AI["🤖 Google Generative AI<br>(Assistenz & Conversation)"]:::cloud
            G_Drive["☁️ Google Drive<br>(Home Assistant Backups)"]:::cloud
            Mobile["📱 Companion Apps<br>(Smartphone, iPad, Mac)"]:::cloud
        end

        %% Verbindungen
        HA --- Floors
        
        SkyConnect <-->|Thread / Zigbee| HA
        TP_Link <-->|WLAN| HA
        Tuya <-->|Cloud API| HA
        
        HA <-->|API| LocalNet
        HA <-->|API / Webhooks| Cloud
```

# 📡 Cross-Site Monitoring
```mermaid
graph LR
    %% Styling
    classDef wueStyle fill:#34495e,stroke:#2980b9,stroke-width:2px,color:#fff;
    classDef leiStyle fill:#34495e,stroke:#27ae60,stroke-width:2px,color:#fff;

    subgraph WURZBURG ["🏰 Standort Würzburg"]
        Uptime_WUE["🔸 Uptime Kuma (Docker)"]:::wueStyle
    end

    subgraph LEINACH ["🏡 Standort Leinach"]
        Uptime_LEI["🔸 Uptime Kuma (Docker)"]:::leiStyle
    end

    %% Heartbeat Tunnel
    Uptime_WUE <-->|🔒 HTTPS Heartbeat| Uptime_LEI

    %% Notiz im Diagramm
    %% Überwachung erfolgt gegenseitig. Keine geteilten Datenbanken oder Datenströme.
```
