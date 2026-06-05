# CodePay / ECR Hub Protocol Reference

> **SNAPSHOT — last reconciled 2026-06-05 (may be stale).** CodePay updates its docs/SDKs.
> Per SKILL.md **Step 0**, WebFetch the live docs + the official SDK and reconcile
> BEFORE trusting any value here for code you write. If a live value differs, the
> live value wins — then update this file. Live docs/SDK links are at the bottom.

Reference for the CodePay (a.k.a. Wisehub / PayCloud) card-terminal integration.
**These wire details cannot be guessed — agents that reconstruct a "reasonable" REST
contract get it wrong.** The live docs/SDK are the source of truth.

---

## Two API families — pick the right one first

| Family | Used for | Success code | Auth | Transport |
|---|---|---|---|---|
| **ECR Hub** | On-terminal app + external POS talking to a terminal **locally** | `response_code` `"000"` (Intent) **or** `"0"` (LAN) — accept both | none / device pairing | Android Intent · LAN WebSocket · USB · Serial |
| **Cloud Gateway** (`wisehub.cloud.*`) | POS talking to the terminal **through CodePay's cloud** | `code == "0"` | **RSA2 signature** | HTTPS REST + MQTT push |

Most counter/restaurant POS work uses **ECR Hub** (terminal is on the same counter
or *is* the device). Use Cloud Gateway only for web POS / remote / multi-store where
the POS cannot reach the terminal on a LAN.

---

## Three integration topologies (ECR Hub)

| Topology | When | How you call it |
|---|---|---|
| **A. On-terminal** | POS app **runs on** the CodePay Android terminal | Android **Intent** `com.codepay.transaction.call` + `startActivityForResult` |
| **B. External POS — local** | POS on a tablet/PC, terminal is a **separate** device on the LAN/USB | Official SDK (`EcrClient`) or raw **WebSocket/USB/Serial** speaking the ECR Hub JSON envelope |
| **C. External POS — cloud** | POS cannot reach the terminal locally | Cloud Gateway REST (`wisehub.cloud.pay.order`) + RSA2 + MQTT result push |

The **business/transaction layer is identical across all three** — only the transport
differs. Design one `PaymentTerminal` interface + per-topology transport adapters.

---

## ECR Hub message envelope (topologies A & B)

Every request and response is one JSON object:

```json
{
  "topic": "ecrhub.pay.order",
  "request_id": "a-unique-id-per-request",
  "app_id": "your_app_id",
  "timestamp": "1698302992263",
  "version": "2.0",
  "biz_data": { /* transaction-specific */ }
}
```

| Field | M/C/O | Notes |
|---|---|---|
| `topic` | M | which operation (table below) |
| `request_id` | C | **unique per request; required for terminal confirmation mode.** This is the idempotency / recovery anchor. |
| `app_id` | M | identifies the sending POS app — issued **once per POS app / integrator** at CodePay onboarding (shared across that app's merchants), **not per-merchant**. Normally a configured/build-time constant; must be the real registered value (never a placeholder). |
| `timestamp` | O | epoch millis as string (live `api-structure` marks it Optional; SDKs still set it) |
| `version` | O | protocol version, e.g. `"2.0"` |
| `biz_data` | M | the transaction payload |

### Topics

| Topic | Purpose |
|---|---|
| `ecrhub.pay.order` | Submit a transaction — **sale, void, refund** (distinguished by `trans_type`) |
| `ecrhub.pay.query` | Query a previous transaction by `merchant_order_no` / `request_id` (recovery) |
| `ecrhub.pay.close` | Cancel/close the terminal's waiting screen |
| `ecrhub.pay.batch.close` | Batch settlement (end-of-day) |
| `ecrhub.pay.tip.adjustment` | Adjust the tip on an authorized-but-unsettled transaction |

---

## biz_data — request fields

| Field | Used by | Notes |
|---|---|---|
| `trans_type` | order | **`1` = sale, `2` = void/cancel, `3` = refund**. (Query uses `2` under the query topic.) |
| `merchant_order_no` | all | **Your unique reference, ≤32 chars.** The key for void/refund/query. Generate uniquely per attempt; persist BEFORE sending. |
| `order_amount` | sale/refund | **Decimal STRING in major units, e.g. `"2.15"` — NOT cents/integer.** See amount gotcha below. |
| `price_currency` | sale (cloud) | ISO-4217, e.g. `"USD"`. |
| `pay_scenario` | sale | entry mode — see values below |
| `tip_amount` | sale (optional) | decimal string; used when tip is collected POS-side |
| `on_screen_tip` | sale (optional) | boolean — terminal prompts the customer for a tip on its own screen |
| `on_screen_signature` | sale (optional) | boolean (default often `true`) — capture signature on terminal |
| `receipt_print_mode` | sale (optional) | `0`=none, `1`=merchant, `2`=customer, `3`=both |
| `orig_merchant_order_no` | void/refund | references the original `merchant_order_no` |
| `isConfirm_on_terminal` | order (SDK) | two-step confirmation flag (see recovery) |

### `pay_scenario` values

Confirmed: `SWIPE_CARD`, `SCANQR_PAY`, `BSCANQR_PAY` (merchant-scan QR).
Card-present entry modes commonly also include insert/tap variants — **verify the
exact enum for your processor/region in the docs or SDK before relying on them.**

---

## biz_data / response — result fields

Response envelope carries `response_code` and `response_msg` (error description).

> **⚠️ SUCCESS CODE DIFFERS BY TRANSPORT (verified on real hardware 2026-06-05):
> on-terminal Intent (topology A) returns `response_code "000"`; LAN WebSocket
> (topology B) returns `response_code "0"`.** Treat BOTH `"000"` and `"0"` as
> success, or the LAN path wrongly reads an approved sale as a decline (terminal
> shows success, POS shows an error). Anything else = decline.

Transaction result fields:

| Field | Meaning |
|---|---|
| `trans_no` | CodePay's transaction number |
| `trans_status` | terminal status (approved / voided / refunded — verify exact strings; the LAN demo returned `"2"` for an approved sale) |
| `trans_type` | echoes 1/2/3 |
| `order_amount` / `tip_amount` / `total_amount` | decimal strings (LAN may omit `total_amount`/`tip_amount`) |
| `card_no` | **masked** PAN (last 4 only — never full PAN), e.g. `456331******3919` |
| `card_brand` / `card_type` | Visa / Mastercard / … — **but the LAN WebSocket reports the brand as `pay_method_id` (e.g. `"Visa"`), not `card_brand`.** Accept either. |
| `auth_code` | issuer authorization code (print on receipt) |
| `ref_no` | reference number for reconciliation |
| `signature_url` | (LAN) URL to the captured signature image — do NOT log it in prod |
| `trans_end_time` | completion time |

Cloud Gateway responses use `code == "0"` for success and wrap the result in a
`data` object instead of `biz_data`.

---

## Recovery & idempotency (THE most important correctness concern)

A card flow can take 1–2 minutes (insert/tap, PIN, tip, signature) and the
connection can drop mid-transaction. **Never assume "no response == not charged."**

CodePay's native mechanism:

1. Generate a **unique `merchant_order_no`** (and `request_id`) per attempt and
   **persist it BEFORE sending** (write-ahead).
2. Use **two-step confirmation** (`isConfirm_on_terminal` / confirmation mode keyed
   by `request_id`) so the terminal holds the result until the POS acknowledges.
3. On timeout / lost response, send **`ecrhub.pay.query`** keyed by
   `merchant_order_no` to learn the true final state, then record accordingly.
4. Only record the payment in your POS after a confirmed `"000"`/`"0"`. A **decline**
   is a clean "no money moved" — do NOT void it. Only a genuinely **unknown** state
   warrants an auto-reversal (void) before giving up.

### When to recover — be precise (don't query on every failure)

The `ecrhub.pay.query` recovery is a fallback for a genuinely **UNKNOWN** state,
NOT for ordinary declines. Trigger it ONLY when:

- the request **threw** (no response at all): timeout, socket/connection loss,
  parse failure — i.e. you never learned the result; **or**
- the terminal **returned a non-success response whose `response_msg` mentions a
  timeout** (e.g. "Transaction timeout"). Map that to `unknown`, not a clean
  decline — the money state is genuinely uncertain.

Do **NOT** query on a real decline (insufficient funds, "Invalid application
invoke", etc.): the terminal told you no money moved. Querying those just adds
load and a confusing second prompt.

**A user CANCEL is a decline, not an unknown — keep both transports consistent.**
On the LAN WebSocket path a cancel comes back as a normal non-success response
(→ decline, no query). On-terminal Intent, **VERIFIED on real hardware
(2026-06-05): a cancel comes back as `RESULT_OK` (resultCode `-1`, NOT
`RESULT_CANCELED`) with `response_code "110"` / `response_msg "Transaction
failed[[K026]Manual cancelation by operator]"` and an EMPTY `biz_data`.** So a
cancel is just a non-success result — read `"110"` as a clean decline. Two
native-side pitfalls that both turn this cancel into a false `unknown` →
unwanted query:

- ⚠️ **Never `jsonDecode` the `biz_data` extra blindly.** A cancel (and other
  responses) return `biz_data` as an **empty string**; `jsonDecode("")` throws
  `FormatException` → caught upstream as `unknown` → query. Guard: empty /
  malformed `biz_data` → empty map, never throw. (General bug, not cancel-only.)
- For any non-`RESULT_OK` resultCode (e.g. an early `RESULT_CANCELED` before the
  card flow starts), have the native shim return a forced non-success result map
  (`response_msg` "Cancelled"), **NOT** `result.error(...)` — an error becomes a
  Dart `PlatformException` → `unknown` → query.

(A genuine no-response — the Intent never returns at all — still times out →
`unknown` → query, which is correct.)

### One continuous overlay until the FINAL result (UX)

Show ONE blocking "follow the prompts on the terminal" overlay and keep it up for
the **whole** transaction — the sale AND (when the sale is `unknown`) the
recovery query — dismissing it only once the FINAL, post-recovery result is
known. Two rules that sound contradictory but aren't:

- **Don't spawn a SECOND/separate overlay** (or loading window) for the recovery
  query — the cashier must never see the prompt pop up twice.
- **Don't drop the overlay before the query either** — leaving the POS
  interactive mid-transaction lets the cashier tap other things while the money
  state is still unresolved.

So: **reuse the same overlay** across sale → recovery → final result. Keep the
sale and the recovery as **separate** calls (so recovery fires only on a genuine
`unknown`, per the trigger rules above), but run **both under that one overlay**;
show the resolved result only after it closes (approved → record + receipt;
declined → message; still-unknown → "could not confirm, check the terminal").

Reconcile/settle with `ecrhub.pay.batch.close` at end-of-day / shift close.

---

## Amount format gotcha

- POS systems typically store money as **integer cents**.
- CodePay wire `order_amount` is a **decimal string in major units** (`1.00`, `2.15`).
- Convert at the transport boundary only: `cents → (cents/100) formatted "0.00"` on
  the way out; parse decimal string → cents on the way back. Centralize + unit-test
  this. (Baseline agents wrongly put `amount_cents` on the wire.)

---

## Tips — three models (make it configurable)

| Model | How | When |
|---|---|---|
| **Terminal screen** (`on_screen_tip=true`) | POS sends sale amount; terminal prompts %; returns actual tip + total | US default, quick-service |
| **POS-entered** | POS adds tip first, sends sale+tip total (and/or `tip_amount`) | cashier-entered |
| **Post-auth adjustment** (`ecrhub.pay.tip.adjustment`) | authorize first, add tip later before settlement | bar tabs / sit-down + signature line |

> The on-terminal Intent docs name the tip-adjust amount field **`tip_adjustment_amoun`**
> — that is the spelling the official doc uses (a typo, missing the trailing `t`).
> Verify against the SDK build you target before relying on it; send what the device
> actually parses.

---

## PCI scope

Semi-integrated: the **terminal** handles PAN / track / PIN / EMV; the POS only sees
masked `card_no` (last4), brand, `auth_code`, `trans_no`. This keeps the POS out of
full PCI-DSS SAQ-D scope. **Never store or transmit full PAN / track / CVV.**

---

## Cloud Gateway auth (topology C only)

RSA2 signature: generate a 2048-bit RSA key pair, upload the **public** key in the
Paypilot dashboard (payment app → API security → RSA). Each request includes
`app_id`, `merchant_no`, `sign_type=RSA2`, `timestamp`, `method`, `format=JSON`,
`charset=UTF-8`, `version`, and `sign`. Build `sign` by: drop `sign` + empty values,
sort params alphabetically, join with `&`, sign with **SHA256WithRSA**, base64-encode.

---

## Official sources of truth (verify field names / enums here)

Docs:
- Overview — https://developer.codepay.us/docs/guides/overview
- On-terminal (Intent) — https://developer.codepay.us/docs/guides/integrate-with-codepay-terminal
- External POS (local + cloud) — https://developer.codepay.us/docs/guides/integrate-with-external-pos
- API structure (envelope/topics) — https://developer.codepay.us/docs/guides/api-structure
- API security (RSA2) — https://developer.codepay.us/docs/guides/api-secure
- Cloud API reference — https://developer.codepay.us/docs/CloudAPI
- USB mode — https://developer.codepay.us/docs/UsbMode
- WLAN debug — https://developer.codepay.us/docs/WLANDebugResources

SDKs / demos (read these for exact classes, fields, and enums):
- Android ECR Hub client — https://github.com/paycloud-open/ecrhub-client-sdk-android
- Windows register SDK — https://github.com/codepay-us/codepay-register-sdk-windows
- Java cloud gateway SDK — https://github.com/codepay-us/gateway-api-sdk-java
- Cross-terminal demo (Java) — https://github.com/codepay-us/codepay-register-cross-terminal-integration-demo-java
- Orgs — https://github.com/codepay-us · https://github.com/paycloud-open

### Local WebSocket connection (topology B) — port + current SDK shape

Verified against live `local-communication` docs (2026-06-04):

- **WebSocket endpoint: `ws://<terminal-ip>:35779`** (fixed port **35779**).
- Enable on the terminal: **Settings > General > ECR Hub > WLAN/LAN** — the screen
  then shows the terminal's IP + port.
- **For non-Android POS (Flutter on tablet/Windows/iOS), do NOT use the Android SDK.
  Speak the ECR Hub JSON envelope directly over a raw WebSocket to that URL.** Send
  `{topic, request_id, app_id, timestamp, version, biz_data}`; match the response by
  `request_id`.

> **⚠️ Non-standard `101` handshake — silently breaks strict WebSocket clients
> (verified on real hardware 2026-06-04).** The terminal's WS server is
> **TooTallNate Java-WebSocket**, which replies with
> `HTTP/1.1 101 Web Socket Protocol Handshake` — status code `101` is correct, but the
> **reason phrase is `Web Socket Protocol Handshake`, NOT the RFC-6455 standard
> `Switching Protocols`.** Lenient clients (curl, the Android SDK, Dart `dart:io`, most
> Node/Go libs) ignore the reason phrase and connect fine. **Strict clients reject it —
> most notably Apple's `URLSessionWebSocketTask` (iOS/macOS), which gives no error and
> hangs forever** waiting for `Switching Protocols`.
> **Fix (Apple platforms / any strict client):** do NOT use `URLSessionWebSocketTask`.
> Hand-roll a minimal RFC-6455 client over a raw TCP socket — Swift: `NWConnection`
> (Network.framework) — that accepts *any* `101` and does its own frame masking; no
> third-party dependency. (A lenient third-party lib such as Starscream also works.)
> Flutter's `web_socket_channel` on iOS uses `dart:io` and is usually fine; native
> Swift/Apple-URLSession clients are the ones that get bitten. Diagnose with
> `curl -i -N --http1.1 -H "Connection: Upgrade" -H "Upgrade: websocket"
> -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ=="
> http://<ip>:35779/` and read the status line.

**Two SDK generations exist — current docs use `ECRHubClient`; older GitHub README uses
`EcrClient`.** Current shape (live docs):

```kotlin
val mClient = ECRHubClient.getInstance()
mClient.init(config, context)
mClient.connect("ws://192.168.100.2:35779")   // disconnect() to close
// operations grouped under mClient.payment:
mClient.payment.sale(params, callback)          // refund / cancel / query / tipAdjustment / batchClose
// callback = ECRHubResponseCallBack { onSuccess(data); onError(code, msg) }
```

Older `EcrClient` shape (GitHub `ecrhub-client-sdk-android` README — kept for reference):

```kotlin
val client = EcrClient.getInstance()
client.init(context, connectListener)
client.connectWifi(terminalIp)          // discover/pair via EcrWifiDiscoveryService

val params = PaymentRequestParams().apply {
  app_id = "your_app_id"
  topic = Constants.PAYMENT_TOPIC        // ecrhub.pay.order
  timestamp = System.currentTimeMillis().toString()
  biz_data = PaymentRequestParams.BizData().apply {
    trans_type = Constants.TRANS_TYPE_SALE   // 1
    order_amount = "1.00"                     // decimal string!
    merchant_order_no = uniqueRef
    pay_scenario = "SWIPE_CARD"
    isConfirm_on_terminal = false
  }
}
client.doTransaction(JSON.toJSONString(params), responseCallback)
// recovery:
client.queryTransaction(/* topic = QUERY_TOPIC, biz_data.merchant_order_no = ref */)
client.cancelTransaction(/* topic = CLOSE_TOPIC */)
```

### Android on-terminal Intent shape (topology A, reference)

```java
Intent i = new Intent("com.codepay.transaction.call");
i.putExtra("version", "2.0");
i.putExtra("app_id", appId);   // ⚠️ snake_case "app_id" — NOT "appId" (see below)
i.putExtra("topic", "ecrhub.pay.order");
i.putExtra("biz_data", bizDataJsonString);   // {"trans_type":"1","order_amount":"2.15",...}
startActivityForResult(i, REQ_PAY);
// onActivityResult: data.getStringExtra("response_code"/"response_msg"/"biz_data")
```

> **⚠️ VERIFIED ON REAL HARDWARE (2026-06-05): the app-id Intent extra key is
> `app_id` (snake_case), NOT `appId`.** The official developer docs
> (`integrate-with-codepay-terminal`) say `putExtra("appId", ...)` — that is
> WRONG. With `appId` the terminal can't find the app id and rejects the
> transaction with **`response_code` 106 / `response_msg` "Transaction
> failed[[M009]Invalid application invoke]"** — even with a valid, registered
> app id. Switching the extra key to `app_id` fixes it. It is consistent with
> the ECR Hub envelope, whose fields are all snake_case (`version`, `topic`,
> `biz_data`, `app_id`). The Intent action (`com.codepay.transaction.call`) and
> the result extras (`response_code` / `response_msg` / `biz_data`) ARE correct
> as documented.
>
> **M009 "Invalid application invoke" troubleshooting** (real-world): it means
> the terminal won't let the *calling app* invoke a transaction. Check, in
> order: (1) wrong app-id extra key (`appId` vs `app_id`, above — this was the
> actual cause once); (2) app id not registered / not bound to the terminal's
> merchant; (3) the calling app's package + signing certificate not authorized
> in PayPilot (debug-signed builds may be rejected — try a release build).
> CodePay does not publicly document M009 — confirm app-authorization rules with
> CodePay support (support@codepay.us).
