# hufo-video-player

Remotely controllable, multi-instance synchronized video player based on `mpv` and UDP broadcast.

## Architecture

- **Leader** (`video_player_1`): Plays the video and broadcasts playback position/state via UDP broadcast (`12338/udp`).
- **Followers** (`video_player_2` ... `video_player_6`): Listen on `12338/udp` and automatically adjust playback speed (and hard seek if drift > 5s) to stay in sync.
- **Control Bridge** (`mpv-udp-bridge`): Listens on `12339/udp` for simple commands (`play`, `pause`) and forwards them to mpv IPC socket (`/tmp/mpvsocket`).

## Files

- [`mpv-sync.lua`](mpv-sync.lua): The Lua sync engine for mpv.
- [`mpv.conf.j2`](mpv.conf.j2): Jinja2 template for `mpv.conf` deployed by Ansible (configures leader for host #1 and follower for the rest).
- [`systemd/mpv.service`](systemd/mpv.service): User systemd service that launches mpv once the desktop (XFCE) is ready.
- [`systemd/mpv-udp-bridge.service`](systemd/mpv-udp-bridge.service): User systemd service bridging UDP control commands to `/tmp/mpvsocket`.
