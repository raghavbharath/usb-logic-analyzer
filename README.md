# ECE692 Final Project - USB Digital Logic Analyzer

**Group Members:** Arnav Revankar, Raghav Bharathan, Gonzalo Villanueva, William Dougherty, Bryan Galecio

This project is a custom USB logic analyzer built around two STM32F446RE Nucleo boards, developed as our final project for ECE 692 at NJIT. The system captures digital signals from an external device under test, decodes UART, SPI, I2C, and CAN protocols, and visualizes the results in real time through a Python GUI.

## How It Works

The transmitter Nucleo generates test signals over UART, SPI, I2C, and CAN. The receiver Nucleo samples up to 8 digital channels simultaneously at 1MHz using DMA-driven GPIO capture, packages the raw samples into packets, and streams them to a host PC over USB CDC. A Go application reads the packet stream, decodes UART, SPI, and I2C from the raw samples, and forwards the results to a Python GUI over TCP. CAN frames are decoded inside the STM32 hardware itself by the bxCAN peripheral and forwarded directly. 

## Repository and Branch Structure

Note: This repository is organized across several feature branches, with each of us making contributions in different areas. The `master` branch contains the integrated codebase including the firmware configured for 10kHz sampling over ST-Link USART, which was used during final testing and demonstration. The 1MHz USB CDC configuration and other work can be found in the individual feature branches (`feature/stm32-firmware`, `feature/SignalDecoder`, `feature/GUI`, `feature/PCB`, `feature/3DEnclosure`). The `feature/test` branch contains the final snapshot used during the live demo.

```
firmware-receiver/     # STM32 receiver firmware (STM32CubeIDE)
firmware-transmitter/  # STM32 transmitter firmware
SignalDecoder/         # Go signal decoder application
gui/src/               # Python GUI
PCB/                   # KiCad schematic and PCB layout
enclosure/             # 3D printable STL files
docs/                  # Datasheets and reference manuals
```


### Firmware

The receiver firmware runs on an STM32F446RE at 168MHz. TIM1 generates a 1MHz update event that triggers DMA2 on each tick. The DMA reads the lower byte of GPIOC (PC0-PC7) into a 1024-byte circular ping-pong buffer. When the first half fills, an interrupt fires and the firmware packages bytes 0-511 into a 516-byte logic packet with a 2-byte header, sequence number, and XOR checksum, then sends it over USB CDC. The second half fires the same way. This design keeps sampling continuous with no gaps.

CAN is handled separately. The bxCAN peripheral runs at 500kbps and decodes incoming frames entirely in hardware, delivering a clean frame (ID, DLC, data bytes) to an RX interrupt handler. The firmware packages this into a variable-length CAN packet and sends it over USB CDC alongside the logic packets.

The transmitter firmware runs the same clock config and outputs test signals in a loop every second: UART at 115200 baud, SPI at ~164kHz, I2C to a dummy address (0x52), and a CAN frame with ID 0x123 and payload `0xCAFEBABEDEADBEEF`.

Packet formats:
- Logic: `[0xAA][0xBB][seq][512 bytes][XOR checksum]` = 516 bytes
- CAN: `[0xCC][0xDD][seq][ID_H][ID_L][DLC][0-8 data bytes][XOR checksum]` = variable length

### Go SignalDecoder

The SignalDecoder reads the USB serial stream from the receiver. A `ReadByteStream` goroutine reads into a 1024-byte buffer and pushes raw byte slices into a buffered channel. A `Run` coordinator consumes from the channel, scans for `0xAA 0xBB` or `0xCC 0xDD` headers using a pending accumulation slice to handle packets that span across reads, validates XOR checksums, and detects dropped packets via sequence number wraparound. Logic packets go to the UART, SPI, and I2C decoders. CAN packets are forwarded directly to Python over TCP.

CLI flags: `--port`, `--protocol`, `--pins`, `--duration`

### Python GUI

The GUI launches the Go binary as a subprocess with the appropriate CLI arguments, then connects to its TCP socket to receive decoded data. A background thread parses incoming packets and emits Qt signals to the main thread.

The waveform view shows 8 digital channels plus a synthetic CAN row reconstructed from decoded frame timestamps and DLC. Features include a crosshair cursor, a permanent multi-pair measurement tool with delta-time and frequency readout, a vertical scrollbar, protocol decode annotations color-coded by type, and a CAN decode table. Captures can be saved and loaded as JSON.

Run in simulator mode (no hardware needed, but a few tweaks might need to be made to the code):
```
python gui/src/gui.py --test
```

### PCB

The PCB is a breakout shield that plugs onto both Nucleo boards simultaneously via their morpho headers. It carries a TXS0108E 8-channel bidirectional level shifter for the probe inputs, two SN65HVD230 CAN transceivers (one per Nucleo), per-channel 1kohm series resistors and BAT85 Schottky diodes for overvoltage protection, 4.7kohm I2C pull-ups, a 10-pin probe header, a CANH/CANL screw terminal, and 120ohm CAN termination resistors at each bus end. The schematic is split into receiver and transmitter sub-sheets under a hierarchical root sheet in KiCad.

### Enclosure

3D-printable STL files for a custom enclosure are in `enclosure/`.

## Dependencies

Python:
```
pip install -r gui/src/requirements.txt
```

Go:
```
cd SignalDecoder && go build
```

