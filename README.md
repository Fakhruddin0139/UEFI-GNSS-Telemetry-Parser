# UEFI-GNSS-Telemetry-Parser

A bare-metal UEFI firmware application built in pure C (EDK2) that parses 
real-time quad-constellation GNSS data (GPS, GLONASS, BeiDou, Galileo) 
entirely before an operating system loads.

## Overview

This project is a multi-menu UEFI firmware application (2600+ lines) built 
with EDK2 on Windows 11. It runs entirely in the pre-boot environment — 
no operating system, no standard libraries, no floating-point math.

See it running live: [LinkedIn Demo Post](https://www.linkedin.com/posts/fakhruddin3a_uefi-gnss-firmwaredevelopment-activity-7456901280981241856-W94x?utm_source=share&utm_medium=member_desktop&rcm=ACoAAFLEpmoBjindx0EnffihZonACEmRQ7mkVoI)

The application contains six functional menus:

| Menu | Feature |
|------|---------|
| 1 | Calculator Suite (Basic, Scientific, Currency) |
| 2 | Sorting Algorithms (Bubble, Selection, Insertion, Heap, Quick, Merge) |
| 3 | System Information (CPU, Memory, Firmware, Time) |
| 4 | Encryption Tools (Caesar, XOR, ROT13) |
| 5 | Pattern Renderer (Triangle, Diamond, Checkerboard, Pyramid, Spiral) |
| 6 | GNSS Telemetry Parser ← **This project** (starts at line 1848) |

---

## GNSS Telemetry Parser — Menu 6

### What it does

Simultaneously parses real-time satellite data from four national navigation 
constellations and displays a live 2×2 monitoring grid with a Best Available 
Fix engine that mathematically scores each constellation and selects the most 
accurate position source in real time.

### Four Constellations

| Constellation | Origin | Talker ID | Sentence |
|--------------|--------|-----------|---------|
| GPS | USA 🇺🇸 | GP | $GPGGA |
| GLONASS | Russia 🇷🇺 | GL | $GLGGA |
| BeiDou | China 🇨🇳 | BD | $BDGGA |
| Galileo | EU 🇪🇺 | GA | $GAGGA |

### Live Display

```
+==========================+==========================+
|   GPS     (USA)          |   GLONASS (RUS)          |
|   Sentence : $GPGGA      |   Sentence : $GLGGA      |
|   UTC Time : 123519      |   UTC Time : 123519      |
|   Lat      : 22* 32.54 N |   Lat      : 22* 32.54 N |
|   Lon      : 88* 22.61 E |   Lon      : 88* 22.61 E |
|   Altitude : 15.4 m      |   Altitude : 15.4 m      |
|   Sats     : 8           |   Sats     : 6           |
|   Fix      : Autonomous  |   Fix      : Autonomous  |
|   HDOP     : 0.9         |   HDOP     : 1.2         |
+==========================+==========================+
|   BeiDou  (CHN)          |   Galileo  (EU)          |
|   Sentence : $BDGGA      |   Sentence : $GAGGA      |
|   UTC Time : 123519      |   UTC Time : 123519      |
|   Lat      : 22* 32.54 N |   Lat      : 22* 32.54 N |
|   Lon      : 88* 22.61 E |   Lon      : 88* 22.61 E |
|   Altitude : 15.4 m      |   Altitude : 15.4 m      |
|   Sats     : 10          |   Sats     : 7           |
|   Fix      : DGPS Fix    |   Fix      : Autonomous  |
|   HDOP     : 0.7         |   HDOP     : 1.1         |
+==========================+==========================+
|   BEST AVAILABLE FIX --> BeiDou (CHN)               |
|   Lat : 22* 32.54 N  Lon : 88* 22.61 E  Alt: 15.4m  |
|   Reason : Sats=10  Fix=DGPS Fix  HDOP=0.7          |
+=====================================================+
```

### Best Available Fix Scoring Algorithm

```
Score = (Satellites × 10) + (FixQuality × 5) - (HdopInt × 2)
Tie-break: Lower HDOP wins
```

---

## Technical Highlights

- **Zero standard libraries** — no stdlib, no string.h, no math.h
- **Zero floating-point math** — all decimals handled as split integers
- **Custom NMEA 0183 GGA parser** — written from scratch in pure C
- **Fixed-point integer arithmetic** — for all coordinate and decimal math
- **EFI_SERIAL_IO_PROTOCOL** — hardware serial communication at firmware level
- **memcpy/memset stubs** — compiler generates hidden runtime calls even without
  explicit use; stubs satisfy the linker in no-stdlib UEFI environment
- **Function order discipline** — every function defined before it is called
  (no forward declarations)

---

## Architecture

```
SITL Pipeline:
nmea_feed.txt
      ↓
nmea_sender.ps1 (PowerShell — 250ms throttle per sentence)
      ↓
TCP socket 127.0.0.1:4444
      ↓
QEMU virtual serial port
      ↓
EFI_SERIAL_IO_PROTOCOL
      ↓
ReadSerialLine() → ParseGGASentence() → DisplayGnssScreen()
                                              ↓
                                     CalculateBestFix()

Function Call Tree:
UefiMain()
└── GnssMenu()
    ├── ReadSerialLine()
    ├── ParseGGASentence()
    │   └── ConvertNMEACoord()
    └── DisplayGnssScreen()
        └── CalculateBestFix()
```

---

## Build Environment

| Component | Details |
|-----------|---------|
| OS | Windows 11 |
| Toolchain | Visual Studio 2022 |
| Firmware SDK | EDK2 |
| Emulator | QEMU with OVMF |
| NASM | C:\nasm |

---

## How to Build

```bash
# From C:\edk2 directory after running edksetup.bat
build -p MyApps/MyApps.dsc -a X64 -t VS2022 -b DEBUG
```

Output EFI:
```
C:\edk2\Build\MyApps\DEBUG_VS2022\X64\UEFICalculator.efi
```

---

## How to Run

**Step 1 — Launch QEMU with TCP serial:**
```bash
"C:\Program Files\qemu\qemu-system-x86_64.exe" ^
  -drive if=pflash,format=raw,readonly=on,file="C:\Program Files\qemu\share\edk2-x86_64-code.fd" ^
  -drive file=fat:rw:C:\edk2\Build\MyApps\DEBUG_VS2022\X64,format=raw ^
  -serial tcp:127.0.0.1:4444,server,nowait
```

**Step 2 — Boot into EFI shell and run:**
```
UEFICalculator.efi
→ Press [6] GNSS Telemetry
→ Press [1] Start Live Monitor
```

**Step 3 — Launch SITL emitter in PowerShell:**
```powershell
.\nmea_sender.ps1
```

The PowerShell script throttles NMEA sentences at 250ms intervals — perfectly 
mimicking a physical copper-wire GNSS antenna at 9600 baud. The firmware is 
completely blind to the simulation.

---

## Repository Structure

```
UEFI-GNSS-Telemetry-Parser/
├── UEFICalculator.c       # Main firmware source (2603 lines)
│                          # GNSS Parser starts at line 1848
├── UEFICalculator.inf     # EDK2 module definition file
├── nmea_feed.txt          # Dummy NMEA sentences (Jamshedpur coordinates)
├── nmea_sender.ps1        # PowerShell SITL TCP emitter
├── LICENSE                # MIT License
└── README.md              # This file
```

---

## Key Lessons Learned

| Challenge | Solution |
|-----------|---------|
| No stdlib in UEFI | memcpy/memset stubs written manually |
| No floating point | Split integer arithmetic (AltInt + AltDec) |
| MSVC /GL optimization | /GL- flag in BuildOptions |
| EDK2 namespace collision | Renamed QuickSort to MyQucikSort |
| QEMU VGA font limitation | ASCII borders instead of box-drawing chars |
| Serial data timing | gBS->Stall(500) prevents CPU spin |
| File-based serial exhausted | TCP socket with PowerShell throttle |

---

## What's Coming Next — Phase 2

This GNSS parser is **Layer 1** of an autonomous drone swarm decision engine:

```
Layer 1 → GNSS Parser           ✅ Complete
           Where are the drones?

Layer 2 → Waypoint Engine        ← In Development
           How far to target?
           Equirectangular approximation
           Fixed-point integer Haversine

Layer 3 → Swarm Struct           ← Planned
           What is each drone's combat value?
           SWARM_DRONE + MissionValue scoring

Layer 4 → Sacrifice Engine       ← Planned
           Who intercepts the threat?
           EvaluateSwarmSacrifice() + tie-break logic
```

The firmware knows where they are.
The algorithm will decide what they do.

---

## Author

**MD Fakhruddin**  
B.Tech CSE '26 | Ex-DRDO Intern (ITR, Chandipur)  
Telemetry Division — Directorate of Telemetry  

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://linkedin.com/in/fakhruddin3a)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black)](https://github.com/Fakhruddin0139)

---


*Developed entirely in the pre-boot firmware environment — before any OS loads.*



