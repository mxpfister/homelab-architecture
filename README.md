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

    subgraph PVE ["💻 Proxmox Node (pve)"]
        
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
        PVE->>GDrive: 1a. rclone sync von externer SSD (--delete-before)
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

    subgraph PVE_LEI ["💻 Proxmox Node (Leinach)"]
        
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
    GDrive_LEI["☁️ Cloud Backup (Remote)"]:::extStyle
    
    %% Datenpfade
    HDD_LEI -->|rclone sync| GDrive_LEI
    
    class PVE_LEI pveNode;
```
## 🕓 2. Cronjobs

1. 04:00 Uhr: Proxmox Rclone Backup (pve-Node) $\rightarrow$ Synchronisiert lokale Dumps zu Google Drive.

## 📡 3. Cross-Site Monitoring
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
    Uptime_WUE <-->|🔒 HTTPS / WAN Heartbeat| Uptime_LEI

    %% Notiz im Diagramm
    %% Überwachung erfolgt gegenseitig. Keine geteilten Datenbanken oder Datenströme.
```
