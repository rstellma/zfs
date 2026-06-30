Type | Version/Name
--- | ---
Distribution Name	| OpenSuSE Leap
Distribution Version	| 16.0
Kernel Version	| 6.12.0-160000.35-default
Architecture	| x86\_64
OpenZFS Version	| zfs-2.4.3-1

<!-- toc -->
- [1. POOL ERSTELLEN](#1-pool-erstellen)
  - [1.1 Pool erstellen mit Wunsch-Haupt-Mountpoint](#11-pool-erstellen-mit-wunsch-haupt-mountpoint)
  - [1.2 Verschlüsseltes Basis-Dataset](#12-verschlüsseltes-basis-dataset)
  - [1.3. Dataset Tuning](#13-dataset-tuning)
- [2.  DATASETS MOUNTEN](#2--datasets-mounten)
- [2.1. Rechte für unprivilierten Nutzer anpassen](#21-rechte-für-unprivilierten-nutzer-anpassen)
  - [2.2. Direkt in $HOME mounten](#22-direkt-in-home-mounten)
  - [2.3. Nach Benutzeranmeldung mounten](#23-nach-benutzeranmeldung-mounten)
- [3. KONTROLLE](#3-kontrolle)
  - [3.1. zpool list](#31-zpool-list)
  - [3.2. zpool status](#32-zpool-status)
  - [3.3. zfs list](#33-zfs-list)
  - [3.4. Fehlerkorrektur aus 3.2](#34-fehlerkorrektur-aus-32)
    - [3.4.1. Abkürzung](#341-abkürzung)
- [4. SCHREIB- / LESECACHE-GRÖßE ANPASSEN](#4-schreib---lesecache-größe-anpassen)
  - [4.1. /etc/modprobe.d/zfs.conf](#41-etcmodprobedzfsconf)
  - [4.2. Initrd neu bauen](#42-initrd-neu-bauen)
- [5. ERWEITERUNG DES POOLS](#5-erweiterung-des-pools)
  - [5.1. Neue Platte hinzufügen](#51-neue-platte-hinzufügen)
  - [5.2. Daten auf breitere Parität umschreiben](#52-daten-auf-breitere-parität-umschreiben)
  - [5.3. ZFS drosseln, um im laufenden Betrieb ein flüssiges System zu behalten](#53-zfs-drosseln-um-im-laufenden-betrieb-ein-flüssiges-system-zu-behalten)
  - [5.4. Diskrepanz zwischen "zpool list" und "zfs list"](#54-diskrepanz-zwischen-zpool-list-und-zfs-list)
- [6. SNAPSHOTS](#6-snapshots)
  - [6.1 Tägliche Snapshots erstellen](#61-tägliche-snapshots-erstellen)
    - [6.1.1. `bash` Skript](#611-bash-skript)
    - [6.1.2. Cron Job](#612-cron-job)
  - [6.2. Snapshot zurückrollen](#62-snapshot-zurückrollen)
    - [6.2.1. Den letzten Snapshot zurückrollen](#621-den-letzten-snapshot-zurückrollen)
    - [6.2.2. Einen aelteren Snapshot zurückrollen](#622-einen-aelteren-snapshot-zurückrollen)
  - [6.3. Snapshot manuell erstellen](#63-snapshot-manuell-erstellen)
  - [6.4. Einzelne Dateien aus Snapshot zurückholen](#64-einzelne-dateien-aus-snapshot-zurückholen)
- [7. FAQ / Bekannte Probleme](#7-faq--bekannte-probleme)
- [8. ZFS-Best-Practices für Home-Labs](#8-zfs-best-practices-für-home-labs)
  - [8.1. Workload-basierte Optimierung von ZFS-Datasets](#81-workload-basierte-optimierung-von-zfs-datasets)
    - [8.1.1. Container (Root-Dateisysteme für distrobox / podman / incus)](#811-container-root-dateisysteme-für-distrobox--podman--incus)
    - [8.1.2. Container-Logs](#812-container-logs)
    - [8.1.3. VirtualBox- / VM-Festplatten-Images (.vdi)](#813-virtualbox---vm-festplatten-images-vdi)
  - [8.2. Allgemeine Faustregeln](#82-allgemeine-faustregeln)
  - [8.3. Wichtiges ZFS-Verhalten](#83-wichtiges-zfs-verhalten)
<!-- tocstop -->

# 1. POOL ERSTELLEN

## 1.1 Pool erstellen mit Wunsch-Haupt-Mountpoint
```bash
sudo zpool create -f -o ashift=12 -m /storage storage raidz1 /dev/disk/by-id/foo /dev/disk/by-id/bar /dev/disk/by-id/baz
```

## 1.2 Verschlüsseltes Basis-Dataset

> [!IMPORTANT]
> Passphrase wird hier abgefragt

```bash
sudo zfs create -o encryption=aes-256-gcm -o keyformat=passphrase storage/secured
```

## 1.3. Dataset Tuning

```bash
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=128k storage/secured/distrobox
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=64k storage/secured/incus
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=64k storage/secured/podman
sudo zfs create -o compression=zstd-5 -o atime=off -o recordsize=1M -o sync=disabled storage/secured/backup
sudo zfs create -o compression=zstd-1 -o atime=off -o recordsize=1M -o sync=disabled storage/secured/vbox
```

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 2.  DATASETS MOUNTEN

# 2.1. Rechte für unprivilierten Nutzer anpassen

```bash
sudo chown -R <user>>:<group> /storage/secured
sudo chmod 700 /storage/secured

sudo chown -R <user>:vboxusers /storage/secured/vbox
sudo chmod 770 /storage/secured/vbox
```

## 2.2. Direkt in $HOME mounten

```bash
sudo zfs set mountpoint=/home/<user>/__data/backup torage/secured/backup
sudo zfs set mountpoint=/home/<user>/__data/distrobox storage/secured/distrobox
sudo zfs set mountpoint=/home/<user>/__data/incus storage/secured/incus
sudo zfs set mountpoint=/home/<user>/__data/podman storage/secured/podman
sudo zfs set mountpoint=/home/<user>/__data/vbox storage/secured/vbox
```

## 2.3. Nach Benutzeranmeldung mounten

> [!IMPORTANT]
> Datasets werden nicht automatisch gemountet!

> [!WARNING]
> Starten von Virtualbox, podman, incus, ... würde direkt auf die Partition schreiben, die /home hält (abhängig von der Konfiguration). Das wollen wir nicht.

> [!TIP]
> Bash Alias erstellen, um sich Tipparbeit zu ersparen.

```bash
alias zfs-mount='sudo zfs load-key storage/secured; sudo zfs mount -a'
```

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 3. KONTROLLE

## 3.1. zpool list

```bash
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
storage  10.9T  2.66T  8.25T        -         -     0%    24%  1.00x    ONLINE  -
```

## 3.2. zpool status

```bash
  pool: storage
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-9P
  scan: scrub repaired 0B in 01:26:08 with 1 errors on Sat Jun 27 23:33:05 2026
expand: expanded raidz1-0 copied 2.71T in 03:12:16, on Sat Jun 27 22:06:57 2026
config:

        NAME                               STATE     READ WRITE CKSUM
        storage                            ONLINE       0     0     0
          raidz1-0                         ONLINE       0     0     0
            ata-TOSHIBA_HDWD130_Y28GXZ9AS  ONLINE       0     0     4
            ata-TOSHIBA_HDWD130_Y28GYBSAS  ONLINE       0     0     4
            ata-TOSHIBA_HDWD130_Y28GVHGAS  ONLINE       0     0     2
            ata-TOSHIBA_HDWD130_Y28GXUYAS  ONLINE       0     0     2
      
errors: No known data errors
```

## 3.3. zfs list

```bash
NAME                        USED  AVAIL  REFER  MOUNTPOINT
storage                    1.77T  5.37T   128K  /storage
storage/secured            1.77T  5.37T   234K  /storage/secured
storage/secured/backup     1.51T  5.37T  1.51T  $HOME/__data/backup
storage/secured/distrobox   828M  5.37T   828M  $HOME/__data/distrobox
storage/secured/incus      13.3G  5.37T  13.3G  $HOME/__data/incus
storage/secured/podman     1.50G  5.37T  1.50G  $HOME/__data/podman
storage/secured/vbox        247G  5.37T   247G  $HOME/__data/vbox
```

## 3.4. Fehlerkorrektur aus 3.2

> [!IMPORTANT]
> Nur notwendig, falls Fehler auftreten

> [!TIP]
> Fehlerhafte Datei(en) aus Backup zurückholen oder ganz löschen

* ZFS Fehlerspeicher zurücksetzen

```bash
sudo zpool clear storage
```

* ZFS Scrub erzwingen (zwingend erforderlich)

```bash
sudo zpool scrub storage
```

### 3.4.1. Abkürzung

> [!CAUTION]
> nicht empfohlen, quick'n'dirty

* ZFS Scrub starten
* ZFS Scrub abbrechen

```bash
sudo zpool scrub -s storage
```

* ZFS Fehlerspeicher leeren

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 4. SCHREIB- / LESECACHE-GRÖßE ANPASSEN

## 4.1. /etc/modprobe.d/zfs.conf

```
# Minimaler Cache: 2 GB (2147483648 Bytes)
options zfs zfs_arc_min=2147483648

# Maximaler Cache: 6 GB (6442450944 Bytes)
options zfs zfs_arc_max=6442450944
```

## 4.2. Initrd neu bauen

```bash
sudo dracut -f --regenerate-all
```

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 5. ERWEITERUNG DES POOLS

## 5.1. Neue Platte hinzufügen

```bash
sudo zpool attach storage storage-0 /dev/disk/by-id/neue-platte
```

## 5.2. Daten auf breitere Parität umschreiben

Der Befehl muss immer auf einen Pfad im Dateisystem (Dateien oder Ordner) angewendet werden, nicht auf den logischen Dataset-Namen.

```
sudo zfs rewrite -rv $HOME/__data/<mount_point>
```

## 5.3. ZFS drosseln, um im laufenden Betrieb ein flüssiges System zu behalten

Standardwert ist oft 10. Wir drosseln massiv auf maximal 2 gleichzeitige I/Os pro Platte:

> [!NOTE]
> /sys/module/zfs/parameters/zfs\_vdev\_async\_write\_min\_active

```bash
sudo sysctl -w kstat.zfs.misc.zio.zio_vdev_async_write_max_active=2
```

Zusätzlich begrenzen wir die ungespeicherte Datenmenge im RAM, um "Ruckler" im System zu verhindern:

> [!NOTE]
> /sys/module/zfs/parameters/zfs\_vdev\_async\_write\_max\_active

```bash
sudo sysctl -w kstat.zfs.misc.zio.zio_vdev_async_write_min_active=1
```

> [!IMPORTANT]
> Nach 'rewrite' wieder zurück stellen.

## 5.4. Diskrepanz zwischen "zpool list" und "zfs list"

> [!IMPORTANT]
> Zum Zeitpunkt des Schreibens (06.2026) gibt es eine Diskrepanz der gemeldeten, zur Verfügung stehenden Größen des freien Speicherplatzes

Ein Ticket im OpenZFS Bug Tracker ist erstellt: https://github.com/openzfs/zfs/issues/18711

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 6. SNAPSHOTS

## 6.1 Tägliche Snapshots erstellen

### 6.1.1. `bash` Skript

```bash
#!/bin/bash

# 1. Hier genau eintragen, welche Datasets gesichert werden sollen (mit Leerzeichen getrennt)
DATASETS=(
    "storage/secured/foo"
    "storage/secured/bar"
    "storage/secured/baz"
)

DATE=$(date +%Y-%m-%d)
RETENTION_DAYS=30

# Schleife durch alle definierten Datasets
for DS in "${DATASETS[@]}"; do
    # Prüfen, ob das Dataset überhaupt existiert
    if zfs list "$DS" >/dev/null 2>&1; then
        
        # Snapshot für das einzelne Dataset erstellen (ohne -r)
        zfs snapshot "${DS}@daily-${DATE}"
        
        # Alte Snapshots für dieses spezifische Dataset löschen
        zfs list -t snapshot -o name -s creation | grep "^${DS}@daily-" | while read -r SNAPSHOT; do
            SNAP_DATE=$(echo "$SNAPSHOT" | sed "s/.*@daily-//")
            SNAP_EPOCH=$(date -d "$SNAP_DATE" +%s 2>/dev/null)
            NOW_EPOCH=$(date +%s)
            
            if [ -n "$SNAP_EPOCH" ]; then
                AGE_DAYS=$(( (NOW_EPOCH - SNAP_EPOCH) / 86400 ))
                if [ "$AGE_DAYS" -ge "$RETENTION_DAYS" ]; then
                    zfs destroy "$SNAPSHOT"
                fi
            fi
        done
    fi
done
```

### 6.1.2. Cron Job

* zeitgesteuert

```bash
0 2 * * * /path/to/script
```

* @daily (entspricht 00:00 Uhr)

## 6.2. Snapshot zurückrollen

### 6.2.1. Den letzten Snapshot zurückrollen

Format aus "/root/zfs-daily-snapshot.sh"

```bash
sudo zfs rollback storage/secured/vbox@daily-<YYYY>-<MM>-<DD>
```

### 6.2.2. Einen aelteren Snapshot zurückrollen

> [!CAUTION]
> löscht alle Snapshots seit YYYY-MM-DD bis heute

```bash
sudo zfs rollback -r storage/secured/vbox@daily-<YYYY>-<MM>-<DD>
```

## 6.3. Snapshot manuell erstellen

(bspw. vor Update oder Konfigurationsänderungen)

```bash
sudo zfs snapshot storage/secured/vbox@vor-update
```

Alles lief gut?

```bash
sudo zfs snapshot storage/secured/vbox@vor-update
```

Etwas ging schief?

```bash
sudo zfs snapshot storage/secured/vbox@vor-update
```

## 6.4. Einzelne Dateien aus Snapshot zurückholen

Datei aus

```
<mountpoint>/.zfs/<YYYY>-<MM>-<DD>
```

zurück an seinen Platz kopieren

[↑ Zurück zum Inhaltsverzeichnis](#inhaltsverzeichnis)

# 7. FAQ / Bekannte Probleme

Warum zeigen `zpool list`, `zfs list` und `df -h` unterschiedliche Werte an?

* `zpool list` gibt die physische Kapazität des Pools an, während `zfs list` und `df` den logisch für Datasets verfügbaren Speicherplatz melden. Es ist nicht zu erwarten, dass diese Werte übereinstimmen. Nach einer RAIDZ-Erweiterung kann der Unterschied aufgrund der aktuellen Speicherplatzberechnung von OpenZFS größer als erwartet ausfallen. Einzelheiten dazu findest Du unter Issue [#18711](https://github.com/openzfs/zfs/issues/18711).

Wann sollte ich `zfs rewrite` ausführen?

* Ein „Rewrite“ (Neuschreiben) ist nur erforderlich, wenn Dataset-Eigenschaften geändert wurden, die sich auf neu geschriebene Daten auswirken (z. B. `recordsize` oder `checksum`), oder nach einer RAIDZ-Erweiterung, wenn vorhandene Daten von der neuen Stripe-Breite profitieren sollen. Das bloße Ändern der Eigenschaft führt nicht dazu, dass vorhandene Blöcke neu geschrieben werden.

Was passiert, wenn ich ein Laufwerk durch ein größeres ersetze?

* Die zusätzliche Kapazität steht nicht sofort zur Verfügung. Ein RAIDZ-vdev kann erst wachsen, wenn alle Laufwerke durch größere ersetzt wurden (es sei denn, Du nutzt die neue RAIDZ-Erweiterungsfunktion zum Hinzufügen von Festplatten). Bis dahin bleibt der zusätzliche Speicherplatz auf dem größeren Laufwerk ungenutzt.

Welche Dataset-Eigenschaften wirken sich auf vorhandene Daten aus?

* Die meisten Dataset-Eigenschaften wirken sich nur auf zukünftige Schreibvorgänge aus. Vorhandene Blöcke behalten ihr ursprüngliches Layout bei, bis sie neu geschrieben werden. Dies betrifft Eigenschaften wie `compression`, `recordsize` und `checksum`.

Warum gilt mein Pool bei einer Auslastung von 80 % bereits als fast voll?

* Im Gegensatz zu herkömmlichen Dateisystemen kann die Leistung von ZFS erheblich nachlassen, wenn Pools zu voll werden. Es wird allgemein empfohlen, mindestens 20 % freien Speicherplatz vorzuhalten – insbesondere bei Pools, die virtuelle Maschinen oder Datenbanken beherbergen.

Warum wirken sich Änderungen an der `recordsize` oder der Kompression nicht auf vorhandene Dateien aus?

* Weil diese Eigenschaften erst beim Schreiben von Datenblöcken angewendet werden. Vorhandene Daten behalten ihr ursprüngliches Layout bei, bis sie kopiert, neu geschrieben oder neu erstellt werden.

[↑ Zurück zum Inhaltsverzeichnis](#table-of-contents)

# 8. ZFS-Best-Practices für Home-Labs

## 8.1. Workload-basierte Optimierung von ZFS-Datasets

Die ZFS-Leistung hängt stark davon ab, wie gut die Dataset-Eigenschaften auf die Workload-Muster abgestimmt sind.

Im Folgenden findest Du praktische Voreinstellungen für typische Anwendungsfälle im Home-Lab.

### 8.1.1. Container (Root-Dateisysteme für distrobox / podman / incus)

Ideal für: viele kleine Dateien, häufige Metadaten-Operationen, Random-I/O

```bash
sudo zfs create \
  -o compression=zstd-3 \
  -o atime=off \
  -o recordsize=64K \
  storage/secured/containers
```

> [!NOTE]
>
> * 64K verbessert die Leistung bei kleinen Dateien und metadatenintensiven Workloads
> * zstd-3 bietet ein gutes Gleichgewicht zwischen Geschwindigkeit und Kompression
> * atime=off reduziert unnötige Schreibvorgänge

### 8.1.2. Container-Logs

Ideal für: Text-Logs mit vielen Anhänge-Operationen (Append), hohe Schreibrate, stark komprimierbare Daten

```bash
sudo zfs create \
  -o compression=zstd-3 \
  -o atime=off \
  -o recordsize=128K \
  storage/secured/logs
```

> [!NOTE]
>
> * Sequenzielle Schreibvorgänge profitieren von einer größeren Recordsize
> * Logs lassen sich extrem gut komprimieren
> * Außer Kompression und atime-Deaktivierung ist kaum weitere Optimierung nötig

### 8.1.3. VirtualBox- / VM-Festplatten-Images (.vdi)

Ideal für: Random-Block-I/O, Workloads virtueller Festplatten

```bash
sudo zfs create \
  -o compression=zstd-1 \
  -o atime=off \
  -o recordsize=64K \
  -o sync=disabled \
  storage/secured/vbox
```

> [!NOTE]
>
> * VM-Workloads verhalten sich wie Blockgeräte
> * 64K reduziert die Schreibverstärkung (Write Amplification)
> * sync=disabled steigert die Leistung (nur verwenden, wenn es für Ihr Risikomodell akzeptabel ist)
> * zstd-1 vermeidet CPU-Overhead bei hoher I/O-Last

## 8.2. Allgemeine Faustregeln

* Kleine Random-I/O-Zugriffe → 64K Recordsize
* Sequenzielle Daten → 128K oder größer
* atime immer deaktivieren, sofern nicht ausdrücklich benötigt
* Kompression ist fast immer vorteilhaft (lz4 oder zstd)

## 8.3. Wichtiges ZFS-Verhalten

* Änderungen an Dataset-Eigenschaften wirken sich nur auf neue Schreibvorgänge aus.
* Vorhandene Daten werden nicht automatisch neu geschrieben.
* Verwende `zfs rewrite` (oder Kopieren/Neuschreiben), wenn auch alte Daten davon profitieren sollen.

[↑ Zurück zum Inhaltsverzeichnis](#table-of-contents)
