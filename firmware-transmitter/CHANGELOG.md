# Changelog for Firmware

Put all software notes/changelog stuff here. Try to format something like below to minimize git conflicts:

```
## <date, or commit hash>

<notes>

### Added

### Changed

### Removed

## <older date/hash>

...
```

This format is adopted from [keepachangelog.com](https://keepachangelog.com).

## 87d3902
- Added the looping code for transmitting UART, SPI, I2C, and CAN. 
- Changed the AutoRetransmission from Enable to Disable. I learned that this basically waits for an ACK signal from the CAN's "slave," but since we have no slave and are just sniffing the bus with the logic analyzer, we need to disable this so that it doesn't infinitely retransmit immediately when ACK is missing. 
- I kept the transmitting messages simple in strings instead of in HEX (except for CAN, since the max is 8 bytes). 
- Added the CAN filter
- We actually don't need interrupts. The blocking loop with HAL_MAX_DELAY actually will wait 49.7 days for a response on timeout lol, so this is just a blocking loop.  

## bb930f8

### Added
- Just did the initial CubeMX configuration for the transmitter STM32. 
- 168MHz HCLK: PLLM=4, PLLN=168, PLLP=2, HSE=8MHz - same as receiver
- CAN1: PB8/PB9, 500kbps (PSC=6, BS1=11TQ, BS2=2TQ), Normal mode, AutoBusOff, AutoRetransmission
- USART1: PA9 TX, 115200 baud, 8-bit, no parity, TX only
- SPI2: PB10 SCK, PC1 MOSI, PC2 MISO, Full-Duplex Master, 8-bit, Prescaler=256 (~164kHz)
- I2C1: PB6 SCL, PB7 SDA, Standard Mode 100kHz
- USART2: PA2/PA3, 115200 baud, ST-Link debug
- No USB or DMA since the transmitter only generates test signals in a loop
- Firmware (main.c loop) not yet writte