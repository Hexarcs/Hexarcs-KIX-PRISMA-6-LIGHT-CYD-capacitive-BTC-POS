> 🇧🇷 [Leia em português](arquitetura.md)

# KIX POS — Architecture Documentation

> Documentation generated from `pos_kix_capacitivo_tamc_gt911.ino`.
> The firmware implements a **Lightning Network point-of-sale (PoS)** on an
> **ESP32**, using a 240×320 TFT display (portrait), a **GT911** capacitive
> touchscreen and the **LNbits** API to create/verify BOLT11 invoices.

---

## 1. Overview

The device is a Lightning payment terminal:

1. The operator types the amount in **sats** on the on-screen keypad.
2. The firmware calls LNbits to **create an invoice** (BOLT11).
3. The invoice is shown as a **QR code** on the display.
4. The customer pays with a Lightning wallet (scanning the QR).
5. The firmware **polls the invoice every 3 s** until it detects the payment.
6. On confirmation, it shows the **"PAGO COM SUCESSO!"** screen and returns to the keypad.

Besides the touch interface, there is a **serial interface** (UART) to control the
same flow via text commands.

---

## 2. Hardware stack

```
                        +----------------------------------------+
                        |                 ESP32                  |
                        |                                        |
   I²C (touch)          |  GPIO33  SDA ──────────────┐           |
   ┌──────────────────► |  GPIO32  SCL ──────────────┼──► GT911  |
   │                    |  GPIO21  INT ──────────────┤   Touch  |
   │                    |  GPIO25  RST ──────────────┘           |
   │                    |                                        |
   │                    |  SPI (bus) ──────────────────► TFT_eSPI|
   │                    |  TFT_BL   ──────────────────► Backlight|
   │                    |                                        |
   │                    |  WiFi/HTTPS (insecure) ──────► LNbits  |
   │                    |         YOUR-LNBITS           |
   └────────────────────┘                                        |
                        +----------------------------------------+

   ┌───────────────┐        ┌───────────────┐        ┌───────────────┐
   │  TFT 240×320  │        │  GT911 Touch  │        │   LNbits API  │
   │  (portrait)   │        │  240×320 pts  │        │  (Lightning)  │
   └───────────────┘        └───────────────┘        └───────────────┘
```

| Resource          | Configuration                                        | Pins                        |
|-------------------|------------------------------------------------------|-----------------------------|
| Display           | `TFT_eSPI`, rotation `2` (portrait), 240×320         | via SPI + `TFT_BL`          |
| Capacitive touch  | `TAMC_GT911`, 240×320                                | SDA=33, SCL=32, INT=21, RST=25 |
| Network           | Wi-Fi + `HTTPClient` (adaptive `WiFiClient`/`WiFiClientSecure`) | —             |
| Bluetooth         | `BluetoothSerial` (SPP), Just Works pairing          | —                           |
| JSON              | `ArduinoJson`                                        | —                           |
| QR code           | `qrcode.h` (version 14, `ECC_LOW`)                   | —                           |

> The transport is chosen by the URL: `http://` uses `WiFiClient` (lightweight)
> and `https://` uses `WiFiClientSecure` with `setInsecure()` (no certificate
> validation — diagnostic mode). **HTTPS + Bluetooth on can exhaust the ESP32's
> heap** and make invoice creation fail; to use Bluetooth and invoices at the same
> time, point at an `http://` endpoint (like the baremetal project) or turn BT off.

---

## 3. Software architecture

### 3.1 State machine (screens)

The firmware is driven by a **state enum** (`ScreenState`) and a global variable
`currentScreen` (lines 31–40):

```
                 ┌──────────────┐
    boot ──────► │ CONNECTING   │  connects Wi-Fi (15 s timeout)
                 └──────┬───────┘
                        │ (always)
                        ▼
                 ┌──────────────┐
         ┌──────►│   KEYPAD     │  amount entry in sats
         │       └──────┬───────┘
         │              │ OK / "I:VALOR"
         │              ▼
         │       ┌──────────────┐
         │       │   LOADING    │  "CRIANDO INVOICE..."
         │       └──────┬───────┘
         │              │ POST /payments
         │        ┌─────┴─────┐
         │        ▼           ▼
         │  ┌──────────┐  ┌──────────┐
         │  │ QR_CODE  │  │  ERROR   │  HTTP/JSON/offline
         │  └────┬─────┘  └────┬─────┘
         │       │ payment     │ BACK / tap
         │       │ confirmed   │
         │       ▼             │
         │  ┌──────────┐       │
         │  │  PAID    │       │
         │  └────┬─────┘       │
         │       │ 5 s / tap   │
         └───────┴─────────────┘
```

| State               | Screen                            | Drawing function           |
|---------------------|-----------------------------------|----------------------------|
| `SCREEN_CONNECTING` | "Conectando WiFi…"                | `connectWiFi()`            |
| `SCREEN_KEYPAD`     | 3×4 numeric keypad                | `desenharTelaTeclado()`    |
| `SCREEN_LOADING`    | "CRIANDO INVOICE…"                | `desenharTelaCarregando()` |
| `SCREEN_QR_CODE`    | Invoice QR code + value           | `desenharTelaQRCode()`     |
| `SCREEN_PAID`       | "PAGO COM SUCESSO!"               | `desenharTelaPago()`       |
| `SCREEN_ERROR`      | Title + detail + VOLTAR button    | `desenharTelaErro()`       |

---

### 3.2 Main flow (`loop`)

The `loop()` is **non-blocking** and runs seven phases every iteration (lines 846–995):

```
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │                            loop()                            │
   │                                                              │
   │  1. SERIAL RECEPTION  (non-blocking, UART + BT SPP)          │
   │          │                                                   │
   │          ▼                                                   │
   │  2. TOUCH             (180 ms debounce + hit-test)           │
   │          │                                                   │
   │          ▼                                                   │
   │  3. DEBUG             (heartbeat every 30 s on Serial)       │
   │          │                                                   │
   │          ▼                                                   │
   │  4. AUTO-CHECK        (every 3 s if on QR_CODE)              │
   │          │                                                   │
   │          ▼                                                   │
   │  5. AUTO-RETURN       (5 s after PAID → back to keypad)      │
   │          │                                                   │
   │          ▼                                                   │
   │  6. BT AUTO-REENABLE  (BT-originated op → +10 s)             │
   │          │                                                   │
   │          ▼                                                   │
   │  7. BT REPORT         (reconnect → PAGO + BOLT11)            │
   └──────────────────────────────────────────────────────────────┘
```

Phases 6 and 7 only apply to operations requested over **Bluetooth**: phase 6 turns Bluetooth back on after 10 s (and, for invoices, also stops the auto-check at 10 s — touch/USB keep auto-checking), and phase 7 reports the result (`PAGO` / not-paid + BOLT11) to the client once it reconnects.

Timers used (global `unsigned long` variables, lines 48–51):

| Timer                | Interval | Use                                          |
|----------------------|----------|----------------------------------------------|
| `lastTouchTime`      | 180 ms   | Touch debounce                                |
| `lastAutoCheckTime`  | 3000 ms  | Invoice poll on the QR screen                 |
| `paidScreenTime`     | 5000 ms  | Automatic return after payment                |
| `lastDebugPrintTime` | 30000 ms | Debug heartbeat on Serial                     |

---

## 4. Communication with LNbits

Endpoints (lines 14–16):

| Operation          | Method | URL                                                        |
|--------------------|--------|------------------------------------------------------------|
| Create invoice     | `POST` | `https://YOUR-LNBITS/api/v1/payments`                      |
| Verify invoice     | `GET`  | `.../api/v1/payments/{payment_hash}`                       |
| Wallet balance     | `GET`  | `https://YOUR-LNBITS/api/v1/wallet`                        |

Authentication via the `X-Api-Key` header (hardcoded key, line 16).

### 4.1 Invoice creation — `createInvoice(sats)`

```
  createInvoice(sats)
        │
        ├─ Wi-Fi connected? ── no ──► ERROR screen "SEM WIFI"
        │
        ├─ POST /api/v1/payments
        │     payload: {"out": false, "amount": sats, "memo": "KIX-POS"}
        │
        ├─ HTTP 200/201? ── no ──► ERROR screen "HTTP <code>"
        │
        ├─ parse JSON → payment_request (BOLT11) + payment_hash
        │
        └─ desenharTelaQRCode()  (renders QR from BOLT11)
```

### 4.2 Payment verification

```
   verificarPagamentoSilencioso()  (auto, every 3 s)
   manualCheck()                   (CHECAR button or serial "C" command)
        │
        └─ GET /payments/{hash}
              │
              ├─ 200 + paid==true  ─► desenharTelaPago()
              ├─ 200 + paid==false ─► status "NAO PAGO AINDA"
              └─ error             ─► status "ERRO: <code>"
```

### 4.3 Balance query — `checkBalance()`

`GET /wallet` → extracts `balance_msat` → divides by 1000 → shows "SALDO DA CARTEIRA".

---

## 5. Touch interface (GT911)

- `verificarTouch()` reads `tp.points[0]`, applies 180 ms debounce and `constrain`s
  to the 240×320 rectangle (lines 795–814).
- `tratarToque(px, py)` does the **hit-test** by region, depending on the current screen.

### 5.1 Numeric keypad (3×4 vertical layout)

The drawn coordinates (lines 142–160) and the hit-test (lines 747–772) must match
each other:

```
      x:  0    79    156    240
      ┌───────┬───────┬───────┐
  y   │   1   │   2   │   3   │  130..173
      ├───────┼───────┼───────┤
 130  │   4   │   5   │   6   │  174..221
      ├───────┼───────┼───────┤
 174  │   7   │   8   │   9   │  222..269
      ├───────┼───────┼───────┤
 222  │  DEL  │   0   │  OK   │  270..320
      └───────┴───────┴───────┘
```

> ⚠️ **Note:** the hit-test `y` ranges use slightly different boundaries than the
> drawn positions (e.g. key "1" is drawn at `y=130` with height 42 → ends at 172,
> but the test considers `130..173`). Overall the regions are contiguous, so the
> hit accuracy is acceptable, but there is a small 1–2 px offset on some edges.

### 5.2 QR / Paid / Error screens

| Screen         | Tap action                                             |
|----------------|--------------------------------------------------------|
| `QR_CODE`      | `px<120` → `manualCheck()`; otherwise → back to keypad |
| `PAID`/`ERROR` | Any tap → clears state and returns to the keypad        |

---

## 6. Serial interface (UART 115200 baud + Bluetooth SPP)

The same syntax is accepted by the **USB serial** and by a **Bluetooth SPP**
terminal (pair with the device **`KIX-POS`**; "Just Works" pairing, no PIN — core
3.x does not support a fixed PIN). Commands are terminated by `\n` or `\r`:

| Command      | Action                                       | Function called            |
|--------------|----------------------------------------------|----------------------------|
| `I:<value>`  | Creates an invoice for the value in sats     | `createInvoice()`          |
| `C` / `C:`   | Checks the active invoice's payment          | `manualCheck()`            |
| `L` / `L:`   | Queries the wallet balance                   | `checkBalance()`           |
| `P` / `P:`   | Prints the active BOLT11 invoice             | (directly on the stream)   |
| `H`          | Returns to the home (keypad) screen          | `desenharTelaTeclado()`    |
| `H:`         | Shows system status                          | `mostrarInformacoesSistema()` |
| `help`       | Lists the available commands                 | (directly on the stream)   |

Example serial flow:

```
  > I:1500
  < [SERIAL RX] Comando recebido: [I:1500]
  < [INVOICE] ===== INICIO =====
  ...
  < [INVOICE] ===== FIM OK =====
  > C
  < [CHECK] PAGAMENTO CONFIRMADO!
```

---

## 7. Function reference

| Function                            | Role                                                        |
|-------------------------------------|-------------------------------------------------------------|
| `kickstartHardware()`               | Turns on the backlight (`TFT_BL`) if defined                |
| `desenharBotao(...)`                | Draws a rounded button with centered text                   |
| `atualizarVisor()`                  | Redraws the typed value on the keypad display               |
| `desenharTelaTeclado()`             | Keypad screen (KIX POS header + ON/OFFLINE + 3×4)           |
| `desenharTelaCarregando()`          | "CRIANDO INVOICE…" screen                                   |
| `desenharTelaQRCode()`              | Renders the BOLT11 QR + CHECAR/VOLTAR buttons               |
| `desenharTelaPago()`                | Green success screen                                        |
| `desenharTelaErro(titulo, det)`     | Error screen with a VOLTAR button                           |
| `connectWiFi()`                     | Connects to Wi-Fi (15 s timeout) and goes to the keypad     |
| `iniciarConfiguracaoWiFi()`         | Opens the AP portal (WiFiManager) for Wi-Fi, API and PIN    |
| `carregarConfigLNbits()`            | Loads URL/key/PIN from NVS (fallback: hardcoded)            |
| `salvarConfigLNbits()`              | Saves URL/key/PIN to NVS                                    |
| `iniciarBluetooth()`                | Turns on Bluetooth SPP (Just Works pairing, no PIN)         |
| `alternarBluetooth()`               | Toggles Bluetooth (BT button) and redraws the keypad        |
| `prepararHTTP(...)`                 | Picks `WiFiClient`/`WiFiClientSecure` based on the URL      |
| `desligarBTParaTLS(avisarVoltar10s)` | Turns BT off (if on) before an HTTPS operation; warns "come back in 10s" if the request came from BT |
| `solicitarInvoice(sats, origemBT)`  | Turns BT off, creates the invoice; re-enables in 10s (and stops auto-check) if from BT |
| `reportarStatusInvoiceBT()`         | Reports PAID / not-paid + BOLT11 to the reconnected BT client |
| `agendarResultadoBT(msg)`           | Schedules a result to send to the BT client after reconnect |
| `checkBalance(origemBT)`            | Queries the wallet balance; re-enables BT in 10s if from BT |
| `createInvoice(sats)`               | Creates the invoice on LNbits and shows the QR              |
| `manualCheck(origemBT)`             | Manual payment check; re-enables BT in 10s if from BT       |
| `verificarPagamentoSilencioso()`    | Automatic (silent) payment poll                            |
| `adicionarDigito(c)` / `apagarDigito()` | Edit the value on the keypad                           |
| `processarOK()`                     | Validates the value, saves `current_sats` and creates invoice |
| `tratarToque(px, py)`               | Touch hit-test by screen                                    |
| `verificarTouch()`                  | GT911 read + debounce                                       |
| `mostrarInformacoesSistema(out)`    | Status dump (Wi-Fi, IP, Bluetooth, heap…) on the stream     |
| `despacharComando(input, out)`      | Routes a serial command to the matching action              |
| `lerComando(in, buf, out)`          | Reads a line from a stream (USB or BT) and dispatches       |

---

## 8. Global state variables

| Variable              | Type           | Description                                          |
|-----------------------|----------------|------------------------------------------------------|
| `currentScreen`       | enum           | Current screen (state machine)                       |
| `input_amount`        | `String`       | Value being typed (in sats)                          |
| `current_sats`        | `long`         | Active invoice value                                 |
| `current_payment_hash`| `String`       | `payment_hash` of the active invoice                 |
| `current_bolt11`      | `String`       | BOLT11 invoice (shown as QR)                         |
| `btHabilitado`        | `bool`         | Bluetooth on/off (BT button)                         |
| `bt_pin`              | `String`       | Pairing PIN (saved, not enforced on core 3.x)        |
| `pagamentoConfirmado` | `bool`         | Result of the last invoice (paid?)                   |
| `relatorioBTpendente` | `bool`         | there is a result to report to the BT client after reconnect |
| `btAutoReligarEm`     | `unsigned long`| Re-enables BT automatically (BT-originated invoice only) |
| `autoCheckAte`        | `unsigned long`| Auto-check runs only until this instant (0 = no limit; only BT-originated invoices have a limit) |
| `bolt11Relatorio`     | `String`       | Copy of the BOLT11 for the post-BT report            |
| `relatorioBTTexto`    | `String`       | Result to report to the BT client after reconnect (C/L) |

---

## 9. Notes / security

- **Hardcoded credentials** (SSID, Wi-Fi password and LNbits `X-Api-Key`) are in the
  source code (lines 11–16). In production they should come from config/NVS/secret store.
- **Insecure TLS**: in `https://` mode, `client.setInsecure()` disables certificate
  validation. The code itself recommends validating the certificate after confirming
  the communication. To coexist with Bluetooth, prefer an `http://` endpoint (see the
  heap note in the resources section).
- **`String` JSON concatenation**: the payload is assembled manually (line 486). Using
  `ArduinoJson` to serialize would avoid injection/format-corruption risk (the value
  comes from a `long`, so the risk here is low).
- **QR limited to 450 characters** (line 196): longer BOLT11 invoices are rejected with
  the "INVOICE MUITO LONGA" message.
- **8-digit limit** on the keypad (`adicionarDigito`, line 277).

---

## 10. File map

```
pos_kix_capacitivo_tamc_gt911/
├── pos_kix_capacitivo_tamc_gt911.ino   ← main firmware (this document)
├── qrcode.c / qrcode.h                 ← local QR-code library
└── libraries/
    ├── TFT_eSPI/        ← display driver
    ├── TAMC_GT911/      ← capacitive touch driver
    ├── ArduinoJson/     ← JSON parse/serialize
    ├── WiFiManager/     ← AP portal (WIFI button) to configure Wi-Fi
    └── QRCode/          ← QR (alternative to the local qrcode.c)
```
