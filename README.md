# blue-batter-can

MicroPython code for an **ESP32-S3** that reads battery status from a battery
protection board (BMS) over **CAN**, and a PC script that reads the result
over serial.

It uses the ESP32-S3's built-in TWAI (CAN 2.0) controller through a
register-level driver ([`twai.py`](twai.py)), because mainline MicroPython has
no CAN class for the ESP32 port. You don't need to rebuild the firmware.

> **Status:** the driver and transceiver path have been tested on hardware
> (loopback and self-test modes pass). Reading from a real battery has not
> been confirmed yet.

## Files

| File | Runs on | Purpose |
|---|---|---|
| [`twai.py`](twai.py) | ESP32-S3 | Register-level TWAI/CAN driver (`TWAI`, `CANMessage`) |
| [`battery_sdk.py`](battery_sdk.py) | ESP32-S3 | Battery protocol SDK: queries, CRC-16 check, decoding (`BatteryBMS`) |
| [`battery_read.py`](battery_read.py) | ESP32-S3 | Example that polls the battery and prints a status block once a second. Copy it as `main.py` to run at boot |
| [`serial_read.py`](serial_read.py) | PC | Reads the board's serial output with timestamps and can log it to CSV |

## Hardware

- ESP32-S3 board running MicroPython (tested on v1.29.0, `ESP32_GENERIC_S3` / `SPIRAM_OCT`)
- 3.3 V CAN transceiver such as an SN65HVD230 or TJA1051

| ESP32-S3 | Transceiver |
|---|---|
| GPIO4 (TX) | TXD / CTX |
| GPIO5 (RX) | RXD / CRX |
| 3V3 | VCC |
| GND | GND (also connect it to the battery's CAN ground) |

Connect the transceiver's CANH/CANL to the battery's CANH/CANL, with a 120 Ω
terminating resistor at each end of the bus. On an SN65HVD230, tie the Rs
pin to GND or the chip stays in standby.

## Protocol

The protocol runs at 500 kbit/s with 11-bit standard IDs. It works by query:
the host sends a data frame on an ID (the data bytes are ignored), and the
BMS replies on the same ID. Values are sent high byte first. The last 2 bytes
of each reply are a CRC-16 (polynomial `0xA001`, initial value `0xFFFF`,
Modbus style) over the bytes before them.

| ID | Contents |
|---|---|
| `0x100` | Total voltage (10 mV), current (signed, 10 mA, charging is positive), remaining capacity (10 mAh) |
| `0x101` | Full capacity (10 mAh), cycle count, RSOC (%) |
| `0x102` | Balance flags for cells 1–16 and 17–32, protection flags |
| `0x103` | MOS status (bit0 charge, bit1 discharge), production date, software version |
| `0x104` | Number of cell strings, number of NTC sensors |
| `0x105` | NTC1–NTC3 temperatures (0.1 K; °C = (raw − 2731) / 10) |
| `0x106` | NTC4–NTC6 temperatures (no reply if fewer NTCs are fitted) |

The production date packs the day in bits 0–4, the month in bits 5–8, and
the year minus 2000 in bits 9–15.

Protection flag bits, from bit0 up: cell overvoltage, cell undervoltage, pack
overvoltage, pack undervoltage, charge over-temperature, charge low
temperature, discharge over-temperature, discharge low temperature, charge
overcurrent, discharge overcurrent, short circuit, front-end IC error,
software MOS lock.

## Setup

1. Flash MicroPython onto the ESP32-S3 if it isn't there already.
2. Copy the files to the board:
   ```sh
   mpremote cp twai.py battery_sdk.py battery_read.py :
   mpremote cp battery_read.py :main.py     # optional: start at boot
   ```
3. Run it from Thonny, or with `mpremote run battery_read.py`.

## Reading the output on a PC

```sh
pip install pyserial
python3 serial_read.py                          # /dev/ttyUSB0 @ 115200
python3 serial_read.py --port /dev/ttyACM0      # native USB port
python3 serial_read.py --csv battery_log.csv    # also log one row per status block
python3 serial_read.py --reset                  # reset the board first
```

Close Thonny first, because only one program can hold the serial port.

Example status block:

```
=============================================
          BATTERY STATUS
=============================================
Pack voltage : 52.31 V
Current      : -3.2 A
RSOC         : 87 %
...
CAN: frames rx=42 crc_err=0 tx_err=0 rx_err=0 bus_off=False tx_fail=0 bus_off_count=0
=============================================
```

## Using the SDK

```python
from battery_sdk import BatteryBMS

bms = BatteryBMS(tx=4, rx=5)

# Blocking: query every ID once
bms.read_all()
print(bms.voltage, bms.current, bms.rsoc)
print(bms.status())          # everything as a dict

# Non-blocking, from your own loop
while True:
    bms.poll()               # sends the next query when due and decodes replies
```

Useful options: `BatteryBMS(check_crc=False)` accepts replies even when the
CRC fails, and `on_frame=callback` gives you every raw frame received.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| All values `None`, `tx_fail` rising, "No reply/ACK" | Nothing on the bus is acknowledging. Check the battery CANH/CANL wiring and ground, whether the battery is awake (many packs need a charger, load or wake pin), and the termination (about 60 Ω between CANH and CANL with power off) |
| `bus_off_count` rising | The ESP32 reads back different bits from the ones it sends. Check transceiver power and the TX/RX pins |
| `crc_err` rising | Try `check_crc=False` and print the raw frames (`PRINT_RAW_FRAMES = True` in `battery_read.py`) |
| `mpremote` hangs | The board isn't running MicroPython, or another program (such as Thonny) holds the port |
