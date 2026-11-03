# mpv-sync - Synchronize Multiple mpv Instances via UDP Broadcast

Synchronisiert mehrere mpv-Instanzen über UDP Broadcast. Ein Leader-Player steuert beliebig viele Follower-Player gleichzeitig.

## Features

- **UDP Broadcast**: Ein Leader synchronisiert beliebig viele Followers gleichzeitig
- **Dual Backend**: Unterstützt lua-socket UND socat (mit automatischem Fallback)
- **Adaptive Geschwindigkeitsanpassung**: Progressive Synchronisation ohne Oszillation
- **Manuelle Offset-Anpassung**: Feintuning in 5ms-Schritten mit Ö/Ä-Tasten
- **Hard Seeking**: Automatisches Springen bei großen Zeitunterschieden (>5s)
- **OSD-Anzeige**: Live-Sync-Info direkt im Player

## Installation

### Voraussetzungen

**Option A: lua-socket** (funktioniert auf Debian 13, schneller)
```bash
sudo apt install lua-socket
```

**Option B: socat** (funktioniert auf Debian 11, 12, 13)
```bash
sudo apt install socat
```

**Empfehlung**: Beide installieren, Script wählt automatisch das beste Backend!

### Script installieren

```bash
# Systemweit (für alle User):
sudo cp mpv-sync-1.1.lua /usr/share/mpv/scripts/mpv-sync.lua

# Oder pro User:
mkdir -p ~/.config/mpv/scripts
cp mpv-sync-1.1.lua ~/.config/mpv/scripts/mpv-sync.lua
```

## Verwendung

### Einfachste Verwendung (empfohlen)

**Leader** (1x):
```bash
mpv --script-opts=sync-role=leader,sync-target=192.168.10.255:12345 video.mp4
```

**Followers** (beliebig viele):
```bash
mpv --script-opts=sync-role=follower,sync-target=192.168.10.255:12345 video.mp4
```

- Leader sendet Commands über UDP Broadcast
- Alle Followers auf dem gleichen Netzwerk empfangen automatisch
- Backend wird automatisch gewählt (lua-socket oder socat)

### Backend explizit wählen

```bash
# Nur lua-socket verwenden:
mpv --script-opts=sync-role=leader,sync-backend=socket,sync-target=192.168.10.255:12345 video.mp4

# Nur socat verwenden:
mpv --script-opts=sync-role=leader,sync-backend=socat,sync-target=192.168.10.255:12345 video.mp4

# Auto-Modus (Standard - versucht socket, dann socat):
mpv --script-opts=sync-role=leader,sync-backend=auto,sync-target=192.168.10.255:12345 video.mp4
```

### Leader und Followers mit verschiedenen Backends

**Funktioniert problemlos!** UDP-Pakete sind identisch, egal welches Backend:

```bash
# Leader mit lua-socket (Debian 13):
mpv --script-opts=sync-role=leader,sync-backend=socket video.mp4

# Follower mit socat (Debian 12):
mpv --script-opts=sync-role=follower,sync-backend=socat video.mp4

# Follower mit auto (wählt bestes verfügbares):
mpv --script-opts=sync-role=follower,sync-backend=auto video.mp4
```

## Konfiguration

Alle Optionen via `--script-opts=sync-OPTION=WERT`:

| Option | Standard | Beschreibung |
|--------|----------|--------------|
| `role` | `leader` | `leader` oder `follower` |
| `target` | `192.168.10.255:12345` | Broadcast-Adresse:Port (anpassen an dein Netzwerk!) |
| `backend` | `auto` | `auto`, `socket` oder `socat` |
| `sync_interval` | `0.5` | Sekunden zwischen Position-Updates |
| `seek_threshold` | `5.0` | Sekunden Differenz für Hard Seek |
| `speed_adjust_threshold` | `0.02` | Sekunden - unter 20ms = "in sync" |
| `max_speed_adjust` | `0.5` | Maximale Geschwindigkeitsänderung (50%) |
| `initial_offset` | `0.015` | Initialer Offset in Sekunden (15ms) |
| `show_osd` | `true` | Sync-Info auf dem Bildschirm anzeigen |

### Beispiel mit Optionen

```bash
mpv --script-opts=sync-role=follower,sync-target=192.168.1.255:12345,sync-initial_offset=0.025,sync-show_osd=false video.mp4
```

### Permanente Konfiguration

Erstelle `~/.config/mpv/script-opts/sync.conf`:

```ini
role=follower
target=192.168.10.255:12345
backend=auto
initial_offset=0.020
show_osd=yes
```

## Tastenbefehle (nur Follower)

- **Ö** (Shift+ö): Offset +5ms erhöhen (Follower läuft später)
- **Ä** (Shift+ä): Offset -5ms verringern (Follower läuft früher)

Nützlich für Audio/Video-Lippensync oder wenn verschiedene Hardware unterschiedliche Latenz hat.

## Broadcast-Adresse finden

```bash
# Zeigt alle Netzwerk-Interfaces mit Broadcast-Adressen:
ip addr show | grep -E "inet.*brd"

# Typische Broadcast-Adressen:
# 192.168.1.255 für Netzwerk 192.168.1.0/24
# 192.168.10.255 für Netzwerk 192.168.10.0/24
# 10.0.0.255 für Netzwerk 10.0.0.0/24
```

**Für lokale Tests:**
```bash
sync-target=127.0.0.1:12345
```
(nur auf dem gleichen Rechner, kein Broadcast)

## Synchronisations-Algorithmus

### Progressive Geschwindigkeitsanpassung

Das Script verwendet adaptive Geschwindigkeitsanpassung ohne Oszillation:

| Zeitdifferenz | Anpassung | Beschreibung |
|---------------|-----------|--------------|
| < 20ms | Normal Speed | **IN SYNC** - keine Anpassung |
| 20-50ms | ±5% | Ultra-feine Anpassung |
| 50-200ms | ±10% | Feine Anpassung |
| 200ms-1s | ±32% | Moderate Anpassung |
| > 1s | ±50% (max) | Größere Anpassung |
| > 5s | Hard Seek | Sofortiges Springen |

**Vorteil**: Sanfte Konvergenz zum Sync-Punkt ohne "Überschwingen".

### Wie funktioniert es?

1. **Leader sendet** alle 0.5s seine Position via UDP Broadcast
2. **Followers empfangen** und vergleichen mit eigener Position
3. **Kleine Differenz** (< 5s): Geschwindigkeit anpassen
4. **Große Differenz** (> 5s): Hard Seek zur Leader-Position
5. **In Sync** (< 20ms): Normale Geschwindigkeit

## Fehlerbehebung

### "No backend available!"

```bash
# Installiere mindestens eins:
sudo apt install lua-socket  # Debian 13
sudo apt install socat       # Debian 11, 12, 13
```

### lua-socket: "unexpected symbol near char(127)"

Das ist ein bekanntes Problem auf Debian 12 - mpv's eingebautes Lua ist nicht mit der lua-socket Binary kompatibel.

**Lösung**: Script nutzt automatisch socat als Fallback!

```bash
sudo apt install socat
# Script automatisch wählt socat auf Debian 12
```

### Followers empfangen keine Nachrichten

**Firewall prüfen:**
```bash
# UDP Port öffnen (z.B. 12345):
sudo ufw allow 12345/udp
```

**Broadcast-Adresse prüfen:**
```bash
# Deine Netzwerk-Info anzeigen:
ip addr show

# Beispiel Output:
# inet 192.168.10.50/24 brd 192.168.10.255
#                             ^^^^^^^^^^^^^^ <- Das ist deine Broadcast-Adresse!
```

**Test mit socat:**
```bash
# Terminal 1 (Empfänger):
socat UDP4-RECVFROM:12345,broadcast,fork STDOUT

# Terminal 2 (Sender):
echo "test" | socat - UDP4-DATAGRAM:192.168.10.255:12345,broadcast

# Terminal 1 sollte "test" anzeigen
```

### "Port already in use"

Nur ein Follower kann auf einem Port lauschen pro Rechner (mit lua-socket). Socat erlaubt mehrere.

**Lösung für mehrere Followers auf einem Rechner:**
```bash
# Alle Followers lauschen auf gleichem Port mit socat:
mpv --script-opts=sync-role=follower,sync-backend=socat video1.mp4 &
mpv --script-opts=sync-role=follower,sync-backend=socat video2.mp4 &
```

### Script wird nicht geladen

```bash
# Prüfen ob mpv das Script sieht:
mpv --msg-level=all=debug video.mp4 2>&1 | grep sync

# Sollte anzeigen:
# [cplayer] Loading scripts...
# [cplayer] Loading script: /path/to/mpv-sync.lua
```

## Beispiel-Setups

### Setup 1: Ein Leader, 120 Followers im LAN

```bash
# Leader (Debian 13 mit lua-socket):
mpv --script-opts=sync-role=leader,sync-target=192.168.1.255:12345 video.mp4

# Alle Followers (gemischte Debian-Versionen):
for i in {1..120}; do
    ssh follower$i "mpv --script-opts=sync-role=follower,sync-target=192.168.1.255:12345 video.mp4" &
done
```

### Setup 2: Zwei Beamer, perfekter Sync

```bash
# Beamer 1 (Leader):
mpv --script-opts=sync-role=leader --fs video.mp4

# Beamer 2 (Follower mit 25ms Offset wegen Hardware-Latenz):
mpv --script-opts=sync-role=follower,sync-initial_offset=0.025 --fs video.mp4

# Live-Anpassung: Ö/Ä Tasten für Feintuning
```

### Setup 3: Stereo-Audio mit zwei Lautsprechern

```bash
# Links (Leader):
mpv --script-opts=sync-role=leader --audio-channels=stereo --lavfi-complex='[aid1]pan=stereo|c0=c0|c1=c0[ao]' video.mp4

# Rechts (Follower):
mpv --script-opts=sync-role=follower --audio-channels=stereo --lavfi-complex='[aid1]pan=stereo|c0=c1|c1=c1[ao]' video.mp4
```

## Performance

### lua-socket vs socat

| Backend | CPU | Latenz | Kompatibilität |
|---------|-----|--------|----------------|
| lua-socket | ~0.1% | ~1ms | Debian 13 ✓, Debian 12 ✗ |
| socat | ~0.5% | ~5ms | Debian 11/12/13 ✓ |

**Empfehlung**: Nutze `backend=auto` - Script wählt automatisch das beste!

### Netzwerk-Traffic

- **Leader**: ~20 Bytes/s (Position-Updates alle 0.5s)
- **Pro Follower**: ~0 Bytes/s gesendet, ~20 Bytes/s empfangen
- **120 Followers**: Immer noch nur ~20 Bytes/s vom Leader (Broadcast!)

## Technische Details

### Nachrichten-Format

Einfaches Text-Format über UDP:

```
play                    # Play
pause                   # Pause
seek|123.45             # Seek zu 123.45 Sekunden
position|123.45         # Leader Position (alle 0.5s)
speed|1.5               # Geschwindigkeit geändert
```

### Port-Nutzung

- **Leader**: Bindet an zufälligen Port (nur senden)
- **Followers**: Binden an konfigurierten Port (z.B. 12345) zum Empfangen
- **Broadcast**: Alle Followers auf dem Netzwerk empfangen gleichzeitig

## Lizenz

Frei verwendbar für private und kommerzielle Zwecke.

## Changelog

### Version 1.1
- Dual Backend: lua-socket + socat Support
- Automatisches Backend-Fallback (auto-Modus)
- Progressive Geschwindigkeitsanpassung ohne Oszillation
- Verbesserte Timing-Parameter (20ms Sync-Threshold)

### Version 1.0
- Initial Release
- lua-socket Backend
- Basic Synchronisation

## Support

Bei Problemen:
1. Log-Output prüfen: `mpv --msg-level=all=info video.mp4 2>&1 | grep sync`
2. Backend manuell wählen: `sync-backend=socket` oder `sync-backend=socat`
3. Firewall/Netzwerk prüfen (siehe Fehlerbehebung)

**Script-Versionen**:
- `mpv-sync-1.0.lua`: Nur lua-socket
- `mpv-sync-1.1.lua`: Dual Backend (lua-socket + socat)
- `mpv-sync-nc.lua`: Nur socat (deprecated, nutze 1.1!)
