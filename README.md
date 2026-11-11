> 🇧🇷 [Leia em português](README.pt-BR.md)

# KIX POS

A **Lightning Network point-of-sale (PoS) terminal** for the **ESP32**. It uses a
240×320 TFT display (portrait), a **GT911** capacitive touchscreen and the
**LNbits** API to create and verify BOLT11 invoices.

> The operator types an amount in **satoshis**, the device creates a Lightning
> invoice, shows it as a **QR code**, polls until the customer pays, then shows a
> "PAID" screen and returns to the keypad.

---

## Features

- 240×320 portrait touch UI with a numeric keypad (3×4).
- Lightning invoice creation via LNbits (`POST /api/v1/payments`).
- QR code rendering of the BOLT11 invoice (local `qrcode.c` library).
- Automatic payment polling every 3 s (`GET /api/v1/payments/{hash}`).
- Wallet balance check (`GET /api/v1/wallet`).
- On-device **setup portal** — a small `WIFI` button opens a captive portal to
  configure Wi-Fi, LNbits URL, API key and Bluetooth pairing PIN from a phone
  browser.
- A **`BT` toggle button** next to `WIFI` turns the Bluetooth serial server on/off.
- Serial (UART) **and Bluetooth SPP** command interface — control the device from a
  USB serial monitor or a phone's Bluetooth terminal app (pair with **`KIX-POS`**,
  "Just Works" pairing — no PIN).
- Vendored Arduino libraries (no external `lib install` required).

---

## Hardware

- **ESP32** dev board with integrated display + touch.
- TFT **240×320** (ST7789 driver), portrait rotation.
- Capacitive touch **GT911**.
- The wiring below matches the **ESP32-2432S028R** ("Cheap Yellow Display" / CYD)
  pinout (ST7789 variant).

| Signal            | GPIO |
|-------------------|------|
| TFT MOSI (SDA)    | 13   |
| TFT SCLK          | 14   |
| TFT CS            | 15   |
| TFT DC            | 2    |
| TFT RST           | -1 (not connected) |
| TFT backlight BL  | 27   |
| Touch SDA         | 33   |
| Touch SCL         | 32   |
| Touch INT         | 21   |
| Touch RST         | 25   |

The display driver is configured in
[`pos_kix_capacitivo_tamc_gt911/libraries/TFT_eSPI/User_Setup.h`](pos_kix_capacitivo_tamc_gt911/libraries/TFT_eSPI/User_Setup.h) (`ST7789_DRIVER`,
240×320). The firmware sets `tft.setRotation(2)` for portrait orientation.

---

## Repository structure

```
.
├── README.md                               # this file
├── comandos_arduino_cli_gravar.txt         # original arduino-cli commands (PT-BR)
└── pos_kix_capacitivo_tamc_gt911/          # Arduino sketch (folder name == .ino name)
    ├── pos_kix_capacitivo_tamc_gt911.ino   # main firmware
    ├── qrcode.c / qrcode.h                 # local QR-code generator (MIT)
    ├── architecture.md                     # detailed architecture (EN)
    ├── arquitetura.md                      # detailed architecture (PT-BR)
    └── libraries/                          # vendored libraries
        ├── ArduinoJson/     7.3.0          # JSON parsing/serialization
        ├── TFT_eSPI/        2.5.43         # display driver
        ├── TAMC_GT911/      1.0.2          # GT911 touch driver
        ├── QRCode/          0.0.1          # QR (alternative; firmware uses qrcode.c)
        └── WiFiManager/     2.0.17         # AP-mode captive portal for Wi-Fi setup
```

> `WiFiManager` is used by the on-device `WIFI` button to open a captive portal
> (AP mode) and store the chosen Wi-Fi credentials in NVS.

---

## Getting started

### 1. Install Arduino CLI

```bash
# Linux/macOS (adapt the URL/version as needed)
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh

# or download a release from:
# https://arduino.github.io/arduino-cli/installation/
```

Verify:

```bash
arduino-cli version
```

### 2. Install the ESP32 core

```bash
arduino-cli core update-index
arduino-cli core install esp32:esp32
```

The libraries are vendored in `pos_kix_capacitivo_tamc_gt911/libraries/`, so you do **not** need
`arduino-cli lib install`.

### 3. Configure credentials

Edit the top of `pos_kix_capacitivo_tamc_gt911.ino`:

```cpp
const char* ssid     = "YOUR_WIFI";
const char* password = "YOUR_PASSWORD";

String lnbits_url        = "https://YOUR-LNBITS/api/v1/payments";
String lnbits_wallet_url  = "https://YOUR-LNBITS/api/v1/wallet";
String lnbits_api_key     = "YOUR_X-API-KEY";
```

> ⚠️ The credentials in the file are **placeholders** (`YOUR_WIFI_SSID`,
> `YOUR_WIFI_PASSWORD`, `YOUR_X-API-KEY`, `https://YOUR-LNBITS/…`). Replace them
> with your own values before flashing. Never commit a real `X-Api-Key`.

> 💡 **Bluetooth + HTTPS heap warning:** the ESP32's TLS handshake needs a large
> heap chunk, and running **Bluetooth Classic at the same time** can make HTTPS
> calls fail (invoice creation works only with Bluetooth off). The firmware now
> auto-detects `http://` vs `https://` in the URL: `http://` uses a lightweight
> client and works alongside Bluetooth. If you want to use the Bluetooth terminal
> **and** create invoices at the same time, point the endpoint at **`http://`**.
> With `https://`, turn Bluetooth off (the `BT` button) before creating an invoice.

**Over-the-air setup (no re-flash):** the `WIFI` button on the keypad opens a
captive portal (`KIX-POS-Setup` / `kixpos123`). Connect a phone to that network,
browse to `192.168.4.1` and enter the local Wi-Fi. The same page has fields for
the **LNbits URL**, **API key** and **Bluetooth PIN** (pre-filled with the current
values — leave them blank to keep the current ones). Everything is stored in NVS
and used automatically on the next boot, falling back to the hardcoded values
above only when nothing was saved.

> ⚠️ **Bluetooth PIN note:** ESP32 Arduino core **3.x removed legacy fixed-PIN
> pairing**, so the `Bluetooth PIN` field is currently **saved but not enforced** —
> pairing uses "Just Works" (no PIN). A fixed PIN only works if the firmware is
> rebuilt as an ESP-IDF component with `CONFIG_BT_SSP_ENABLED` disabled.

---

## Flashing with Arduino CLI

Run these commands from the **repository root** (the folder that contains
`pos_kix_capacitivo_tamc_gt911/`).

> `--libraries` points to the vendored libraries, so all dependencies compile
> straight from this repo.

### Find the serial port

```bash
arduino-cli board list
```

Typical values: `/dev/ttyUSB0` (Linux), `/dev/cu.usbserial-*` (macOS), `COM3` (Windows).

### Compile

```bash
arduino-cli compile \
  --fqbn esp32:esp32:esp32:PartitionScheme=huge_app \
  --libraries pos_kix_capacitivo_tamc_gt911/libraries \
  --jobs 4 \
  --verbose \
  --build-property compiler.c.extra_flags="-Wno-error=implicit-function-declaration -Wno-error=implicit-int -Wno-implicit-function-declaration" \
  pos_kix_capacitivo_tamc_gt911
```

- `PartitionScheme=huge_app` reserves the maximum app space (the display/JSON
  libraries are large).
- The `compiler.c.extra_flags` relax the `implicit-function-declaration` /
  `implicit-int` diagnostics required by some of the bundled C sources.

### Upload (flash the board)

```bash
arduino-cli upload \
  --verbose \
  -p /dev/ttyUSB0 \
  --fqbn esp32:esp32:esp32:PartitionScheme=huge_app \
  pos_kix_capacitivo_tamc_gt911
```

Replace `/dev/ttyUSB0` with the port from `arduino-cli board list`.

### Compile + upload in one step

```bash
arduino-cli compile \
  --fqbn esp32:esp32:esp32:PartitionScheme=huge_app \
  --libraries pos_kix_capacitivo_tamc_gt911/libraries \
  --jobs 4 \
  --verbose \
  --build-property compiler.c.extra_flags="-Wno-error=implicit-function-declaration -Wno-error=implicit-int -Wno-implicit-function-declaration" \
  pos_kix_capacitivo_tamc_gt911 \
&& arduino-cli upload \
  --verbose \
  -p /dev/ttyUSB0 \
  --fqbn esp32:esp32:esp32:PartitionScheme=huge_app \
  pos_kix_capacitivo_tamc_gt911
```

### Serial monitor

```bash
arduino-cli monitor -p /dev/ttyUSB0 -c baudrate=115200
```

### Export a compiled binary (optional)

```bash
arduino-cli compile \
  --fqbn esp32:esp32:esp32:PartitionScheme=huge_app \
  --libraries pos_kix_capacitivo_tamc_gt911/libraries \
  --jobs 4 \
  --verbose \
  --build-property compiler.c.extra_flags="-Wno-error=implicit-function-declaration -Wno-error=implicit-int -Wno-implicit-function-declaration" \
  --output-dir ./build_output \
  pos_kix_capacitivo_tamc_gt911
```

The original one-line command reference is also kept in
[`comandos_arduino_cli_gravar.txt`](comandos_arduino_cli_gravar.txt) (PT-BR).

---

## Usage

### On the touchscreen

1. Type the amount in **sats** on the keypad.
2. Press **OK** → the invoice is created and shown as a QR code.
3. Customer scans and pays with a Lightning wallet.
4. The device auto-checks every 3 s (or press **CHECAR**) → shows **PAGO COM SUCESSO!**.
5. Press **VOLTAR** to go back to the keypad.

Tap **WIFI** (top-right of the keypad) to enter the Wi-Fi setup portal described
above. The **`BT`** button next to it toggles the Bluetooth serial server on/off
(green `BT ON` = on, grey `BT OFF` = off).

### Over serial (USB or Bluetooth)

The same commands work over the USB serial monitor (115200 baud) and over a
**Bluetooth SPP** terminal — pair with the device named **`KIX-POS`** (pairing is
"Just Works", no PIN; see the note in the setup section above).

| Command  | Action                                       |
|----------|----------------------------------------------|
| `I:1500` | Create an invoice for 1500 sats              |
| `C` / `C:`| Check the active invoice's payment status   |
| `L` / `L:`| Query wallet balance                        |
| `P` / `P:`| Print the active BOLT11 invoice (copy/paste) |
| `H`      | Return to the home (keypad) screen           |
| `H:`     | Print system status (Wi-Fi, IP, BT, heap…)   |
| `help`   | List all available commands                  |

> 📱 **Bluetooth + TLS workaround:** Bluetooth and HTTPS can't share the ESP32's
> memory at the same time, so before any TLS call (invoice, balance, check) the
> device **turns Bluetooth off automatically** and warns you over the terminal to
> reconnect in 10 s. Any operation requested **over Bluetooth** (`I:` invoice,
> `C` check, `L` balance) turns Bluetooth back on by itself after **10 s**; for an
> invoice, the automatic payment check also runs only for those 10 s and then
> stops. Operations from the touchscreen or USB serial leave Bluetooth **off** —
> tap the **`BT`** button to turn it back on. When a Bluetooth client reconnects,
> the device reports the result over Bluetooth: `PAGO COM SUCESSO!` / the **BOLT11**
> for an invoice, or the check/balance result for `C` / `L`. Type `help` for the
> command list.

Example:

```
> I:1500
< [INVOICE] ===== INICIO =====
< ...
< [INVOICE] ===== FIM OK =====
> C
< [CHECK] PAGAMENTO CONFIRMADO!
```

---

## How the code works

The firmware is a **state machine** driven by `currentScreen` (an `enum` with six
states):

| State               | Screen                         |
|---------------------|--------------------------------|
| `SCREEN_CONNECTING` | "Conectando WiFi…"             |
| `SCREEN_KEYPAD`     | Numeric keypad (3×4)           |
| `SCREEN_LOADING`    | "CRIANDO INVOICE…"             |
| `SCREEN_QR_CODE`    | Invoice QR code + value        |
| `SCREEN_PAID`       | "PAGO COM SUCESSO!"            |
| `SCREEN_ERROR`      | Title + detail + VOLTAR button |

The main `loop()` is **non-blocking** and runs seven phases every iteration:

1. **Serial reception** — reads USB UART **and Bluetooth SPP**, dispatching
   `I:` / `C` / `L` / `P` / `H` / `H:` commands to the same handler.
2. **Touch** — reads the GT911 with 180 ms debounce and hit-tests the current screen.
3. **Debug heartbeat** — prints a "loop alive" line every 30 s.
4. **Auto-check** — if on the QR screen, polls the invoice every 3 s.
5. **Auto-return** — 5 s after "PAID", returns to the keypad.
6. **BT auto-re-enable** — for a Bluetooth-requested operation (invoice/check/balance), turns Bluetooth back on after 10 s (for invoices, the auto-check also stops at 10 s; USB/touch keep auto-checking).
7. **BT report** — once a client reconnects, reports `PAGO` / not-paid + BOLT11.

### Network flow (LNbits)

- `createInvoice(sats)` → `POST /api/v1/payments` with
  `{"out": false, "amount": sats, "memo": "KIX-POS"}` → parses `payment_request`
  (BOLT11) and `payment_hash` → renders the QR code.
- `verificarPagamentoSilencioso()` / `manualCheck()` → `GET /api/v1/payments/{hash}`
  → checks the `paid` field.
- `checkBalance()` → `GET /api/v1/wallet` → reads `balance_msat`, divides by 1000.

All requests authenticate with the `X-Api-Key` header.

> ⚠️ **Security note:** the code uses `client.setInsecure()` (no TLS certificate
> validation) and hardcodes credentials at the top of the file. This is acceptable
> for diagnostics but **not** for production — validate the TLS certificate and move
> secrets to config/NVS.

For a full walkthrough (ASCII diagrams, function reference, security notes), see
[`pos_kix_capacitivo_tamc_gt911/architecture.md`](pos_kix_capacitivo_tamc_gt911/architecture.md) (EN) or
[`pos_kix_capacitivo_tamc_gt911/arquitetura.md`](pos_kix_capacitivo_tamc_gt911/arquitetura.md) (PT-BR).

---

## License

The main firmware and bundled `qrcode.c` (based on Nayuki's QR-Code-generator) are
provided under the MIT license. Individual vendored libraries retain their own
licenses (see each `pos_kix_capacitivo_tamc_gt911/libraries/*` folder). Add a top-level `LICENSE` file if you
plan to distribute this project.
