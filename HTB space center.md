# HTB Space Center --- Automated Visibility Solver

> **CTF / Lab Use Only:** This project is intended for authorized Hack
> The Box challenges and other environments where you have permission to
> test. It automates interaction with the HTB Space Center challenge
> service.

## Overview

The HTB Space Center challenge provides:

-   A satellite **TLE (Two-Line Element)** set
-   A ground-station **latitude/longitude**
-   A visibility threshold of **30° elevation**

The objective is to determine the satellite's rise/set visibility
windows over the required 24-hour period and submit the timestamps.

Normally, the workflow is:

``` text
nc → copy TLE → copy coordinates → run calculator → copy answer → paste into nc
```

The script in this README automates that workflow:

``` text
Connect
   ↓
Receive challenge
   ↓
Parse TLE + coordinates
   ↓
Propagate satellite orbit
   ↓
Calculate 30° visibility windows
   ↓
Generate UTC timestamps
   ↓
Submit answer to the same TCP session
   ↓
Continue to the next challenge
```

## Requirements

-   Python 3
-   Network access to the authorized HTB challenge
-   `skyfield`

Install:

``` bash
python3 -m venv .venv
source .venv/bin/activate
pip install skyfield
```

## Usage

Save the script below as:

``` bash
auto_solve.py
```

Then run:

``` bash
python3 auto_solve.py
```

The script creates its own TCP connection to the challenge service. **Do
not run a separate `nc` session at the same time**, because the
challenge data can change between connections.

------------------------------------------------------------------------

# Automated Solver

``` python
#!/usr/bin/env python3

import socket
import re
import time
from datetime import timedelta

from skyfield.api import load, wgs84, EarthSatellite


HOST = "154.57.164.82"
PORT = 32061

# HTB defines visibility as elevation >= 30 degrees.
THRESHOLD = 30.0


def calculate_windows(tle1, tle2, lat, lon):
    """
    Calculate satellite visibility windows for
    the 24 hours following the TLE epoch.
    """

    ts = load.timescale()

    satellite = EarthSatellite(
        tle1,
        tle2,
        "DIGITWIN HTB",
        ts
    )

    station = wgs84.latlon(lat, lon)

    # Use the TLE epoch as the propagation start.
    start = satellite.epoch

    end = ts.from_datetime(
        start.utc_datetime() + timedelta(hours=24)
    )

    times, events = satellite.find_events(
        station,
        start,
        end,
        altitude_degrees=THRESHOLD
    )

    windows = []
    rise = None

    for t, event in zip(times, events):

        timestamp = t.utc_strftime(
            "%Y-%m-%dT%H:%M:%SZ"
        )

        if event == 0:
            # Satellite crosses above 30 degrees.
            rise = timestamp

        elif event == 2:
            # Satellite crosses below 30 degrees.
            if rise is not None:
                windows.append(
                    (rise, timestamp)
                )
                rise = None

    return windows


def parse_challenge(text):
    """
    Extract the TLE and ground-station coordinates
    from the HTB challenge output.
    """

    tle_match = re.search(
        r"TLE:\s*"
        r"(?:DIGITWIN HTB\s*)?"
        r"(1\s+\d+U[^\n]+)\s*\n"
        r"(2\s+\d+[^\n]+)",
        text
    )

    if not tle_match:
        raise ValueError(
            "Could not find TLE in server response"
        )

    tle1 = tle_match.group(1).strip()
    tle2 = tle_match.group(2).strip()

    station_match = re.search(
        r"\(Lat,Long\):\s*"
        r"([-+]?\d+(?:\.\d+)?)\s*,\s*"
        r"([-+]?\d+(?:\.\d+)?)",
        text
    )

    if not station_match:
        raise ValueError(
            "Could not find station coordinates"
        )

    lat = float(station_match.group(1))
    lon = float(station_match.group(2))

    return tle1, tle2, lat, lon


def main():

    print(f"[+] Connecting to {HOST}:{PORT}")

    sock = socket.create_connection(
        (HOST, PORT),
        timeout=30
    )

    sock.settimeout(2)

    buffer = ""

    while True:

        try:
            data = sock.recv(65535)

        except socket.timeout:
            data = b""

        if data:
            decoded = data.decode(
                errors="ignore"
            )

            print(decoded, end="")
            buffer += decoded

        # Check for a flag.
        if "HTB{" in buffer:

            match = re.search(
                r"HTB\{[^}]+\}",
                buffer
            )

            if match:
                print("\n[+] FLAG:")
                print(match.group(0))

            break

        # Wait until challenge information is present.
        if (
            "Challenge Sat" in buffer
            and "Station location:" in buffer
        ):

            try:

                tle1, tle2, lat, lon = parse_challenge(
                    buffer
                )

                print("\n[+] Parsed challenge")
                print("[+] TLE 1:", tle1)
                print("[+] TLE 2:", tle2)
                print("[+] Latitude :", lat)
                print("[+] Longitude:", lon)

                print(
                    "\n[+] Calculating visibility windows..."
                )

                windows = calculate_windows(
                    tle1,
                    tle2,
                    lat,
                    lon
                )

                if not windows:
                    print("[-] No visibility windows found.")

                else:

                    print(
                        f"[+] Found {len(windows)} window(s)"
                    )

                    answer = []

                    for rise, set_time in windows:

                        print(
                            f"    {rise} -> {set_time}"
                        )

                        answer.append(rise)
                        answer.append(set_time)

                    answer_text = " ".join(answer)

                    print("\n[+] Answer:")
                    print(answer_text)

                    # Wait for the actual input prompt.
                    while (
                        "When will it be visible next?>"
                        not in buffer
                    ):

                        try:
                            data = sock.recv(65535)

                        except socket.timeout:
                            continue

                        if not data:
                            break

                        decoded = data.decode(
                            errors="ignore"
                        )

                        print(decoded, end="")
                        buffer += decoded

                    if (
                        "When will it be visible next?>"
                        in buffer
                    ):

                        print(
                            "\n[+] Sending answer..."
                        )

                        sock.sendall(
                            (answer_text + "\n").encode()
                        )

                        # Keep the same TCP connection.
                        prompt = (
                            "When will it be visible next?>"
                        )

                        position = buffer.rfind(prompt)

                        if position != -1:
                            buffer = buffer[
                                position + len(prompt):
                            ]

                        time.sleep(1)

                        continue

            except Exception as e:

                print(
                    f"\n[-] Solver error: {e}"
                )

                break

        if not data:
            time.sleep(0.1)

    sock.close()


if __name__ == "__main__":
    main()
```

------------------------------------------------------------------------

# How It Works

## 1. Connect to the Challenge

The script opens a TCP connection:

``` python
sock = socket.create_connection(
    (HOST, PORT),
    timeout=30
)
```

This is equivalent to:

``` bash
nc 154.57.164.82 32061
```

but keeps the connection inside Python so the same challenge data can be
processed and answered automatically.

## 2. Parse the TLE

The server provides two TLE lines:

``` text
1 01337U ...
2 01337 ...
```

The script extracts both lines with a regular expression and passes them
to Skyfield:

``` python
satellite = EarthSatellite(
    tle1,
    tle2,
    "DIGITWIN HTB",
    ts
)
```

## 3. Parse the Ground Station

The server provides:

``` text
Station location:
(Lat,Long): -48.24,129.46
```

The script converts those coordinates into a Skyfield observer:

``` python
station = wgs84.latlon(lat, lon)
```

## 4. Calculate Visibility

The challenge says:

``` text
A satellite is visible if it is above 30 degrees in the horizon.
```

The script therefore calls:

``` python
satellite.find_events(
    station,
    start,
    end,
    altitude_degrees=30.0
)
```

Skyfield identifies:

``` text
0 = Rise
1 = Maximum elevation
2 = Set
```

Only the rise and set events are needed.

## 5. Build Visibility Windows

For example:

``` text
RISE
2026-08-17T10:13:14Z

MAX
2026-08-17T10:15:09Z

SET
2026-08-17T10:17:03Z
```

becomes:

``` text
2026-08-17T10:13:14Z 2026-08-17T10:17:03Z
```

If there are multiple passes:

``` text
RISE1 SET1 RISE2 SET2 RISE3 SET3
```

all timestamps are submitted in chronological order.

------------------------------------------------------------------------

# Why the Same Connection Matters

The challenge can generate different TLEs and ground-station coordinates
on different connections.

For example:

``` text
Connection A
    ↓
TLE A + Station A
    ↓
Answer A
```

is different from:

``` text
Connection B
    ↓
TLE B + Station B
    ↓
Answer B
```

Therefore, calculating an answer from one connection and submitting it
to another can result in:

``` text
Incorrect!
```

The automated solver avoids this by:

``` text
connect
  ↓
receive current challenge
  ↓
calculate using current challenge
  ↓
submit to the same socket
```

------------------------------------------------------------------------

# Example Output

``` text
[+] Connecting to 154.57.164.82:32061

Welcome to the HTB Space Center.

Challenge Sat 1
TLE:
DIGITWIN HTB
1 01337U ...
2 01337 ...

Station location:
(Lat,Long): -17.99,82.79

When will it be visible next?>

[+] Parsed challenge
[+] TLE 1: 1 01337U ...
[+] TLE 2: 2 01337 ...
[+] Latitude : -17.991179601112805
[+] Longitude: 82.79721721497765

[+] Calculating visibility windows...
[+] Found 2 window(s)

    2026-08-17T10:13:14Z -> 2026-08-17T10:17:03Z
    2026-08-17T15:17:25Z -> 2026-08-17T15:20:42Z

[+] Answer:
2026-08-17T10:13:14Z 2026-08-17T10:17:03Z 2026-08-17T15:17:25Z 2026-08-17T15:20:42Z

[+] Sending answer...
```

If the challenge returns another satellite, the solver can continue
processing the same connection.

------------------------------------------------------------------------

# Troubleshooting

### `ModuleNotFoundError: No module named 'skyfield'`

``` bash
pip install skyfield
```

### Kali blocks system-wide pip installation

Use a virtual environment:

``` bash
python3 -m venv .venv
source .venv/bin/activate
pip install skyfield
```

### `Could not find TLE`

The server response format may have changed. Inspect the received
challenge output and update the regular expression in
`parse_challenge()`.

### `Could not find station coordinates`

Check the server's exact `Station location:` format.

### `Incorrect!`

Check:

-   The TLE and coordinates belong to the same connection.
-   The threshold is exactly 30°.
-   Timestamps are UTC.
-   Every required rise/set pair is included.
-   The timestamps are in chronological order.
-   The challenge's required 24-hour reference period matches the
    solver's propagation start.

### `Wrong number of windows`

The server expects one rise/set pair for each visibility window. For
three windows, submit six timestamps:

``` text
RISE1 SET1 RISE2 SET2 RISE3 SET3
```

------------------------------------------------------------------------

# Project Structure

A minimal repository can contain:

``` text
htb-space-center-autosolver/
└── README.md
```

The complete solver is intentionally included directly in this README so
the challenge methodology and implementation are documented in one
place.

------------------------------------------------------------------------

# Key Concepts Learned

This challenge combines:

-   TLE orbital data
-   SGP4 satellite propagation
-   Ground-station geometry
-   Elevation-angle calculations
-   Rise/set event detection
-   UTC timestamp handling
-   TCP socket programming
-   Regex-based protocol parsing
-   CTF automation

The central idea is:

``` text
TLE + Ground Station
        ↓
Orbit Propagation
        ↓
Satellite Elevation
        ↓
30° Threshold
        ↓
Rise / Set Events
        ↓
Visibility Windows
        ↓
TCP Submission
```

------------------------------------------------------------------------

# Disclaimer

This code is provided for educational purposes and authorized CTF/lab
environments such as Hack The Box.

Do not use the automation against systems, services, satellites, or
ground stations without explicit authorization.
