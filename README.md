# SignalDecoder

Go application that reads raw capture packets from the STM32 receiver over serial, decodes UART, SPI, and I2C signals from the raw samples, and forwards results to the Python GUI over TCP. CAN frames are decoded in hardware by the STM32 bxCAN peripheral and forwarded as-is.

This is the `feature/SignalDecoder` branch of the ECE692 Final Project.
GitHub: https://github.com/ragusauce4357/ECE692-Final-Project.git

---

## Packet Protocol

| Type  | Header      | Format                                                        | Size       |
|-------|-------------|---------------------------------------------------------------|------------|
| Logic | `0xAA 0xBB` | `[seq][512 raw GPIO samples][XOR checksum]`                   | 516 bytes  |
| CAN   | `0xCC 0xDD` | `[seq][ID_H][ID_L][DLC][0-8 data bytes][XOR checksum]`        | 7-15 bytes |

Each sample byte encodes all 8 channels: bit 0 = CH1 (PC0) through bit 7 = CH8 (PC7).

---

## File Structure

```
SignalDecoder/
    main.go
    go.mod
    internal/
        config/
            config.go       Config struct, GetConfig(), Print()
            parse.go        CLI flag parsing, pin validation, ProtocolPins encoding
        serial/
            serial.go       Serial read loop, packet sync, checksum, drop detection
        decoder/
            uart.go         UART software decoder
            spi.go          SPI software decoder
            i2c.go          I2C software decoder
        logging/
            logging.go      Log prefix helpers
    test/
        test.py             Sends synthetic packets over a virtual serial port
        generate_bin.py     Generates a .bin capture file
        socat.sh            Creates a virtual serial pair in WSL2
```

---

## CLI Flags

| Flag         | Description                                   | Example           |
|--------------|-----------------------------------------------|-------------------|
| `--port`     | Serial port                                   | `--port COM3`     |
| `--baud`     | Baud rate (default 115200)                    | `--baud 115200`   |
| `--duration` | Capture duration in seconds                   | `--duration 10`   |
| `--protocol` | `uart`, `spi`, `i2c`, or `can`                | `--protocol uart` |
| `--pins`     | Channel assignments for the selected protocol | `--pins 1,2`      |
| `--rate`     | Sample rate in Hz (default 1000000)           | `--rate 1000000`  |

Pin assignments: UART `tx,rx` -- SPI `clk,mosi,miso,cs` -- I2C `scl,sda` -- CAN no pins needed.

---

## Build and Run

```bash
go build ./...
go run main.go --port COM3 --protocol uart --pins 1,2 --duration 10
```

---

## Testing Without Hardware

Use socat in WSL2 to create a virtual serial pair:

```bash
# Terminal 1
socat -d -d pty,raw,echo=0,link=/tmp/ttyV0 pty,raw,echo=0,link=/tmp/ttyV1

# Terminal 2 -- write test packets
python3 test/test.py --port /tmp/ttyV0 --protocol uart

# Terminal 3 -- run decoder
go run main.go --port /tmp/ttyV1 --protocol uart --pins 1,2 --duration 10
```

If socat.sh has Windows line endings: `sed -i 's/\r//' test/socat.sh`

To test without the Python GUI, swap the TCP connection in `main.go` for `DummyConn` (defined in `serial.go`). It implements `net.Conn` and silently discards all writes.

---

## Notes

- Sample rate must be greater than baud rate or `samplesPerBit` evaluates to 0.
- The ST-Link USART path caps at ~10kHz. USB CDC at 1MHz is the intended path.
- CAN AutoRetransmission on the transmitter must be set to DISABLE -- without a real receiver to ACK frames, ENABLE causes the STM32 to retransmit indefinitely.
