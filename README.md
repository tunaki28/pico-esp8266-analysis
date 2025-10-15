# Pico-ESP8266 Firmware Analysis

Notes on flashing and analyzing the Waveshare **Pico-ESP8266** module firmware.

Reference: https://www.waveshare.com/wiki/Pico-ESP8266

---

## ⚠️ Disclaimer
This document is for **educational / personal research** purposes only.  
All operations were performed on hardware owned by the author. Do not use these procedures on devices you do not own or without explicit permission.

---

## Pinout and connection

- `#1` GP0 → TXD0 (UART TX)  
- `#2` GP1 → RXD0 (UART RX)  
- `#3` **GND**  
- ...  
- `#38` GND → GND  
- `#39` VSYS → 5V (→ 3.3V regulated)

---

## Quick start

### Hardware connection

1. Press and hold the **BOOTSEL** button on the Pico. The Pico will be recognized as a removable disk.  
   Copy `dev_cdc_ports.uf2` from the demo codes to the Pico (see Waveshare demo). The Pico will restart and act as a USB↔serial adapter.

   Demo codes: https://files.waveshare.com/upload/4/41/Pico-ESP8266.zip

2. Attach the Pico-ESP8266 board to the Pico, then connect the Pico to your PC via USB.

3. Open an SSCOM software(screen cm), choose the correct COM port, and set the baud rate to 115200.M
    *  ```screen /dev/ttyUSB0 115200```
4. Test the module by AT command.
    * ```AT Ctrl+M Ctrl+J```
    * ```AT+GMR Ctrl+M Ctrl+J```
    * ```AT+RST Ctrl+M Ctrl+J```


## Identify flash
1. Put the board in flash/read mode (IO0 → RST according to board/hardware button).

1. flash_id
     1. IO0->RST (On bord, hardware button.)
     2. esptool --port ttyUSB0 --baud 115200 flash_id
     3. -> Detected flash size: **4MB**
     
2. read_flash
    ```bash
    esptool --port ttyUSB0 --baud 115200 read_flash 0x00000 0x400000 ~/Documents/flash_dump.bin
    ```

## Read full flash
```bash
esptool --port /dev/ttyUSB0 --baud 115200 read_flash 0x00000 0x400000 ~/Documents/flash_dump.bin
```
## Analyze flash_dump.bin
1) Binwalk scan
```bash
binwalk ./flash_dump.bin
Example (excerpt) output:

vbnet
Copy code
DECIMAL       HEXADECIMAL     DESCRIPTION
341876        0x53774         SHA256 hash constants, little endian
357514        0x5748A         JBOOT STAG header, image id: 12, ...
423892        0x677D4         SHA256 hash constants, little endian
425284        0x67D44         AES S-Box
425540        0x67E44         AES Inverse S-Box
434372        0x6A0C4         PEM RSA private key
434444        0x6A10C         PEM PKCS#8 private key
434472        0x6A128         PEM certificate
```
Note: binwalk reports markers that look like PEM key/certificate regions at those offsets.

## Try to extract suspected PEM regions
- Extract around 0x6A0C4 (example 16 KiB block):
```bash
dd if=flash_dump.bin of=around_rsa.bin bs=1 skip=$((0x6A0C4)) count=$((16384)) status=progress
```

- Extract PKCS#8 region (example):
```bash
dd if=flash_dump.bin of=around_pkcs8.bin bs=1 skip=$((0x6A10C)) count=$((16384)) status=progress
```
- Extract certificate region (example):
```bash
dd if=flash_dump.bin of=around_cert.bin bs=1 skip=$((0x6A128)) count=$((16384)) status=progress
```

## Quick string / hex inspection
```bash
strings -t x flash_dump.bin | egrep -i 'BEGIN|END|DEK-Info|Proc-Type|certificate|private_key'
hexdump -C -s 0x6a0a0 -n 512 flash_dump.bin | less
```
- Observed fragments (example):
```bash
6a0a0  certificate
6a0ac  private_key
6a0b8  -----BEGIN
6a0c4  -----BEGIN RSA PRIVATE KEY-----
6a0e4  -----BEGIN ENCRYPTED PRIVATE KEY-----
6a10c  -----BEGIN PRIVATE KEY-----
6a128  -----BEGIN CERTIFICATE-----
...
6a144  -----END RSA PRIVATE KEY-----
6a164  -----END ENCRYPTED PRIVATE KEY-----
6a188  -----END PRIVATE KEY-----
6a1a4  -----END CERTIFICATE-----
```
- Hex excerpt (example):
```bash 
0006a090: 7770 6162 7566 206f 7665 7266 6c6f 7700  wpabuf overflow.
0006a0a0: 6365 7274 6966 6963 6174 6500 7072 6976  certificate.priv
...
0006a1c0: 4445 4b2d 496e 666f 3a20 4145 532d 3132  DEK-Info: AES-12
0006a1e0: 3a20 4145 532d 3235 362d 4342 432c 0000  : AES-256-CBC,..
0006a1f0: 5072 6f63 2d54 7970 653a 0000 342c 454e  Proc-Type:..4,EN
...
```
## Findings / Interpretation
- The flash contains PEM-like markers and DEK-Info / Proc-Type strings that are typical of (possibly encrypted) PEM blocks.

- However, no usable plaintext private key or certificate was recovered in the extracted regions — the markers appear to be placeholders, stripped/shortened data, or encrypted blobs (requires passphrase).

- Possible explanations:

    - The firmware includes only template strings or debug labels.
    - The key material is encrypted (DEK-Info suggests encryption) and not recoverable without the passphrase.
    - The actual key is stored elsewhere (secure element, separate partition, or removed).

## Recommendations / Next steps
- Run binwalk -e flash_dump.bin to try automated extraction of embedded filesystems.
- Extract a larger window around the detected offsets (e.g., ± 8 KiB) and re-check for continuous Base64 blocks ([A-Za-z0-9+/=]).
- Compare multiple firmware versions (if available) to spot persistent templates vs real keys.
- If DEK-Info indicates encrypted PEM, understand that recovering the private key requires the passphrase—do not attempt brute-forcing on devices you don't own.

## References
- Waveshare Pico-ESP8266 Wiki: https://www.waveshare.com/wiki/Pico-ESP8266
- esptool: https://github.com/espressif/esptool
- binwalk: https://github.com/ReFirmLabs/binwalk                                                                                
