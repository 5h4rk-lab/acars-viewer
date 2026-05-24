# 5H4RK/ACARS — Flight Tracker

A browser-based ACARS log visualizer built from a $25 RTL-SDR dongle and an acarsdec log file. No API keys, no subscriptions, no backend — just drop your log file and fly.

![5H4RK ACARS](https://img.shields.io/badge/SDR-RTL--SDR-00d4ff?style=flat-square) ![Static](https://img.shields.io/badge/hosted-GitHub%20Pages-39ff14?style=flat-square) ![License](https://img.shields.io/badge/license-MIT-ff6b35?style=flat-square)

## What it does

Load an acarsdec log file and get:

- **Interactive map** — confirmed flight paths plotted with altitude color gradient, projected dashed route to destination
- **Show All mode** — all flights with position data plotted simultaneously, each in a different color
- **Decoded tab** — plain English breakdown of every ACARS message (altitude, speed, heading, fuel, ETA, waypoints)
- **Timeline tab** — chronological event log per flight
- **Raw tab** — original decoded bytes for each message

Supports all major ACARS position label formats: `80`, `16`, `21`, `22`, `36`, `37`, `83`.

## Hardware used

- **Nooelec NESDR SMArt v5** or **RTL-SDR Blog V3** — any RTL-SDR dongle works
- Dipole antenna (~57cm per element for 131 MHz)
- Laptop running Linux (Manjaro/Arch)

## Setup — capture your own data

Install acarsdec:

```bash
# Arch/Manjaro
yay -S acarsdec-rtl-sdr
# or build from source
git clone https://github.com/TLeconte/acarsdec.git
cd acarsdec && mkdir build && cd build
cmake .. -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -Drtl=ON
make && sudo make install
```

Get your dongle's PPM offset:

```bash
rtl_test -p   # run for 3-4 minutes, note cumulative PPM
```

Start logging (8 channels, overnight):

```bash
acarsdec -g 49.6 -p YOUR_PPM -o 2 -e -r 0 \
  130.025 130.450 131.125 131.425 131.475 131.525 131.550 131.725 \
  2>&1 | tee ~/acars_log_$(date +%Y%m%d).txt
```

## Usage

Serve locally (tiles need HTTP not file://):

```bash
cd ~/Downloads
python3 -m http.server 8080
# open http://localhost:8080/index.html
```

Or just open the [live site](https://5h4rk-lab.github.io/acars-viewer) and drag your log file onto the map.

## Message types decoded

| Label | Type |
|-------|------|
| `80` | Position report (full FANS format) |
| `16` | Position report |
| `21` | Position report (POSN format) |
| `22` | Position report (DDMMSS format) |
| `36/37` | Position report (CSV format) |
| `83` | Position report (DDMM.M format) |
| `H1` | FMC uplink / engine health / ACK |
| `SA` | Fuel & weight state |
| `B2` | Arrival/departure |
| `5Z` | ATC clearance |
| `35` | Distress |

## Background

Built during a session exploring what a $25 RTL-SDR dongle can receive — starting with ACARS message decoding, moving through signal analysis and protocol reverse engineering (KeeLoq key fob analysis), and ending with this flight tracker. Full writeup on the key fob RF security research at [blog.5h4rk.me](https://blog.5h4rk.me).

## License

MIT
