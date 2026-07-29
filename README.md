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
    GDrive["☁️ Google Drive (Proxmox_Wue_Backups)"]:::extStyle
    
    %% Datenpfade
    SSD -->|rclone sync| GDrive
    
    class PVE pveNode;
```

## 🕓 2. Cronjobs

1. 04:00 Uhr: Proxmox Rclone Backup (pve-Node) $\rightarrow$ Synchronisiert lokale Dumps zu Google Drive.
2. 05:15 Uhr: Plex Leere Ordner löschen (Plex-LXC) $\rightarrow$ Entfernt verwaiste Verzeichnisse.
3. 05:30 Uhr: Plex Smart Cleanup (Plex-LXC) $\rightarrow$ Löscht gesehene YouTube-Videos nach einer Frist von 5 Tagen.

```mermaid
sequenceDiagram
    autonumber
    participant HA as 🔹 Home Assistant (VM)
    participant PVE as 💻 Proxmox Host (pve)
    participant PlexLXC as 🟢 Plex LXC
    participant Influx as 🔸 InfluxDB (Docker)
    participant GDrive as ☁️ Google Drive

    %% --- ABLAUF 1: PROXMOX BACKUP (04:00 UHR) ---
    rect rgb(39, 174, 96)
        Note over PVE, GDrive: ABLAUF 1: Proxmox Rclone Backup (04:00 Uhr)
        Note over PVE: Lokale System-Crontab<br>löst Backup aus
        PVE->>GDrive: 1a. rclone sync von externer SSD
        activate PVE
        GDrive-->>PVE: 1b. Sync erfolgreich beendet
        PVE->>Influx: 1c. cron-monitor.sh sendet duration & exit_code via curl
        deactivate PVE
    end

    %% --- ABLAUF 2: PLEX LEERE ORDNER LÖSCHEN (05:15 UHR) ---
    rect rgb(44, 62, 80)
        Note over HA, PlexLXC: ABLAUF 2: Plex Leere Ordner löschen (05:15 Uhr)
        HA->>PlexLXC: 2a. Trigger via SSH (HA Shell Command)
        activate PlexLXC
        Note over PlexLXC: Skript räumt leere Verzeichnisse auf
        PlexLXC->>Influx: 2b. cron-monitor.sh meldet Werte an InfluxDB
        deactivate PlexLXC
    end

    %% --- ABLAUF 3: PLEX SMART CLEANUP (05:30 UHR) ---
    rect rgb(142, 68, 173)
        Note over HA, PlexLXC: ABLAUF 3: Plex Smart Cleanup (05:30 Uhr)
        HA->>PlexLXC: 3a. Trigger via SSH: plex_cleaner.py
        activate PlexLXC
        Note over PlexLXC: Prüft 'lastViewedAt'<br>& löscht geschaute Videos (>5 Tage)
        PlexLXC->>Influx: 3b. cron-monitor.sh meldet Werte an InfluxDB
        PlexLXC->>HA: 3c. Python sendet Webhook an HA (Titel-Liste für iPhone)
        deactivate PlexLXC
    end

    %% --- ABFRAGE-SCHLEIFE (DEAD-MAN-SWITCH) ---
    rect rgb(41, 128, 185)
        Note over HA, Influx: KONTINUIERLICHE ÜBERWACHUNG (Dead-Man-Switch)
        loop Alle paar Minuten
            HA->>Influx: 4a. Flux Query (Prüfe letzten Eintrag im Fenster -26h)
            Influx-->>HA: 4b. Liefert Daten für ALLE 3 Jobs (Dauer & Status)
        end
    end
```

⚠️ Wichtiger Hinweis zum Linux-Pipe-Buffer: > Bei Aufrufen über die Home Assistant SSH-Schnittstelle müssen textintensive Skripte zwingend mittels > /dev/null 2>&1 umgeleitet werden. Andernfalls läuft der 64 KB große OS-Pipe-Buffer voll, was zu einem Deadlock (Einfrieren) des Skripts führt.

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
            HDD_LEI["🟪 Externe HDD (500GB)<br>Pfad: /mnt/pve/external-storage"]:::storageStyle
        end

        subgraph VMs_LEI ["🖥️ Virtuelle Maschinen (VMs)"]
            HA_LEI["🔹 Home Assistant VM"]:::vmStyle
        end

        subgraph LXCs_LEI ["📦 LXC Container"]
            subgraph DockerLXC_LEI ["🐋 Docker-Host (LXC)"]
                Portainer_LEI["🔸 Portainer"]:::dockerStyle
                Uptime_LEI["🔸 Uptime Kuma"]:::dockerStyle
            end
            class DockerLXC_LEI lxcStyle;
        end
    end

    %% Externe Verbindungen
    GDrive_LEI["☁️ Google Drive (Remote)"]:::extStyle
    
    %% Datenpfade
    HDD_LEI -->|rclone sync| GDrive_LEI
    
    class PVE_LEI pveNode;
```
## 🕓 2. Cronjobs

1. 04:00 Uhr: Proxmox Rclone Backup (pve-Node) $\rightarrow$ Synchronisiert lokale Dumps zu Google Drive.

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
