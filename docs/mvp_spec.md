# Let Them Pay MVP Specification

## 1. Mandatory VRP/eKasa Functions (MVP+)

### 1.1 Document Types and Operations
- **Supported documents**: Pokladničný doklad (PD), Neplatný doklad (ND), Úhrada faktúry (UF), Vklad (VK), Výber (VY).
- **Additional operations**: Zaevidovanie paragónu, including retroactive registration.
- **PD options**: item-level and document-level discounts, refunds, corrections (UPDATE), voucher exchanges, returned packaging deposit adjustments, and refundable container processing.
- **Rounding**: cash rounding to 0.05 € applies only to cash amounts; card and other non-cash payments are not rounded. For mixed payments, compute the cash amount as the residual after non-cash payments and apply the rounding. Final document totals must reconcile with payment totals including the rounding delta.

### 1.2 VAT Handling
- **Active rates from 1 Jan 2025**: 5% (code C), 19% (code B), 23% (code A).
- **Legacy rates**: 10% (code D) and 20% (code E) disallowed for new sales but required for refund/update operations tied to historical receipts.
- **Mixed-rate refunds**: allow combining legacy and new rates when referencing older documents.

### 1.3 Special Item Types
- **VOUCHER**: single-purpose vouchers with dedicated handling rules.
- **PACKING_REFUND**: refundable packaging; names must be prefixed with `"VO:"` and use 0% VAT.

### 1.4 Operational Features
- Receipt history with copy issuance, X/Z closures, sales reports, catalog and service management, import/export for goods, document and printing settings, messaging, portable cash register location tracking (GPS/address), and theme toggles (light/dark).

### 1.5 Offline Compliance
- All printed output destined for the certified fiscal printer must be sent through OneHub eKasa to ensure CHDÚ (WORM) logging, using `/api/print` for arbitrary content.
- Printer health check initiated via the `PrinterCheck` command.
- Offline receipts temporarily omit fields such as UUID; once connectivity returns, push them to the Financial Administration (FS) through the status API.
- If FS rejects a document, block the register until the receipt is corrected through `/api/document/update`.

## 2. Local Integrations

### 2.1 OneHub eKasa
- **Transport**: HTTP+JSON, port `12412`, Basic Auth header `Authorization: Basic cDNrYXNhOnAza1BX`.
- **Core endpoints**:
  - `merchant/store` & `merchant/get` for uploading Auth/Identity XML and certificate credentials.
  - `/api/document/store` for PD/UF/ND/VK/VY with `clientDocId` UUIDv4 for idempotency. Response includes OKP, PKP, VAT summaries, QR payload, etc.
  - `documents` retrieval via `clientDocId`, `print/document-copy`, and `document/get` (latest).
  - Cash operations for VK/VY, offline document retrieval, printing, and submission.
  - Printer capabilities, arbitrary printing, readiness checks, and location updates for portable registers.
  - CHDÚ usage metrics, certificate expiration info, software metadata.
  - Strict adherence to payment and rounding rules supplied by the API (especially `cashRoundingAmount`).

### 2.2 GP Pay (Local REST)
- **Transport**: HTTP+JSON, port `20008`, base path `/api/v1/pos/transaction`.
- **ECR client responsibility**: provide unique `requestId` (UUIDv4) for idempotent calls.
- **MVP operations**:
  - `CL` handshake delivering `terminalStatus` and `printerStatus`.
  - `CP` card payment (amount in cents, ISO currency code 978) returning merchant/customer receipts, `hostRc`, `approvalCode`, and `hostTransId`.
  - `CC` last transaction void via `previousRequestId`.
  - `CR` refund, `CU` arbitrary transaction void (using `hostTransId` and additional metadata).
  - `CT`, `CS`, `CZ` totals (range/subtotal/summary), `TG` configuration fetch, `TM` POS download.
- **Optional operations**: `CA/CF/CI` preauthorizations, `CX` interrupt, `CD/CM` card/Mifare reads (not required for MVP).
- Focus solely on local integration for MVP; cloud-based use cases are reserved for `cloud.onepos.sk` coordination.

## 3. Cloud Management & Synchronization
- Utilize `cloud.onepos.sk` (per Quickstart) for catalog management, device/user administration, reporting exports, and bidirectional synchronization using last-write-wins rules (`updated_at` + `device_id`).
- **Device pairing**: terminals generate `device_token` (QR). Web admin binds devices to tenant/location, enforcing Owner/Manager/Cashier roles via RLS.

## 4. Data Model Extensions
- **receipts**: `client_doc_id`, `type`, `amount_declared`, `cash_rounding_amount`, `okp`, `pkp`, `uuid`, `qr_payload`, `issue_date`, `create_date`, `process_date`, `electronic`, `status`, `printer_copy_count`, `zdp_flag`.
- **receipt_items**: `item_type`, `reference_document_id`, `vat_rate`, `special_regulation`, `measure_unit`, `plu`, `price_list_id`, `ean`, `voucher_number`.
- **payments**: `method`, `amount`, `label`, `gp_host_trans_id`, `gp_request_id`, `gp_approval_code`, `card_mask`.
- **stock_ledger**: document-driven inventory movements.
- **devices**: `pairing_qr`, `device_token`, `printer_capabilities` (character sizes, line lengths).

## 5. Mobile Cash Register Flows
- **A. Card sale (PD)**: product selection, discount application, VAT computation; execute GP `CP`, store metadata, call eKasa `/api/document/store` with card payment details (no cash rounding); print via eKasa unless `electronicReceipt=true` (then send digitally).
- **B. Cash sale (PD)**: cash tab calculates change using 0.05 € rounding; submit payment detail to eKasa and display returned `cashRoundingAmount` on receipt.
- **C. Mixed payment**: execute GP first, settle remainder in cash; eKasa calculates the cash rounding difference from `amount - card`.
- **D. Refunds/Corrections/Vouchers/Packaging**: locate original document or allow manual entry; apply correct `itemType`, ensure voucher numbers and references are captured.
- **E. Cash deposit/withdrawal (VK/VY)**: dedicated flows generating receipts via eKasa.
- **F. X/Z closures**: include GP totals requests (`CT/CS/CZ`) to reconcile card settlements.
- **G. Offline mode**: persist receipts with `status=PENDING` when `/api/document/store` indicates offline. Provide diagnostic UI for offline report printing and resubmission when online.
- **H. Rejection handling**: on `REJECTED`, block sales, display alert, and open document fix UI. Unlock after successful `/api/document/update`.
- **I. Portable registers**: capture and sync location updates (address, GPS, etc.).

## 6. Web Administration Portal
- Catalog management with grids/tables, bulk actions, CSV import/export.
- Inventory/ledger operations, sales reporting (exports, X/Z, PDF generation).
- User and role administration (Owner/Manager/Cashier, PIN, audit logging).
- Settings for VAT, currency, branding, locations, and device management with QR pairing.
- Low stock badges, CSV/PDF exports.
- Device dashboard showing CHDÚ usage, certificate expiration, and local software version (synced from devices).

## 7. UX/UI Standards
- **Mobile**: high contrast, large touch targets, Roboto typography, monospace for totals, persistent payment bar with tabs for Cash/Card/Other and numeric keypad, visible status badges for sync/printer/GP connectivity. Provide toasts for results (`result`, `hostRc`) and badges for eKasa offline or GP busy states. Run `PrinterCheck` before first print after startup.
- **Web**: sidebar navigation (Catalog, Inventory, Sales, Users, Settings), top bar for location/date/search, efficient bulk actions.

## 8. Technical Best Practices
- Use idempotent identifiers: `clientDocId` for eKasa, `requestId` for GP (reuse returns last known state; handle GP busy with code 5).
- Always print via eKasa (ensures CHDÚ logging); supplement with `/api/print` for additional lines.
- Centralize rounding logic supporting cash-only, card-only, and mixed modes; verify results against eKasa `cashRoundingAmount`.
- VAT safeguard: enforce new rates post-2025 for new sales while permitting legacy rates for refunds/updates linked to historical receipts.
- Enforce sale blocking on `REJECTED` status until resolution through `/api/document/update`.
- Pairing: QR-based `device_token`, persist eKasa credentials and GP network endpoints post-pairing; provide diagnostics for both services (including `CL` and `PrinterCheck`).

## 9. API Contract Examples
- **eKasa PD (card 100 €, cash 0.10 € rounding)**:
  ```json
  {
    "clientDocId": "UUIDv4",
    "type": "PD",
    "amount": 100.10,
    "documentEntries": [
      { "itemType": "SALE", "name": "Tričko", "price": 83.42, "quantity": 1, "vatRate": "VAT_19" }
    ],
    "payments": [
      { "method": "CARD", "amount": 100.00, "label": "VISA ****6335" },
      { "method": "CASH", "amount": 0.10, "label": "Hotovosť" }
    ]
  }
  ```
  Response includes `cashRoundingAmount = 0.10`, `OKP`, `PKP`, `UUID`, and QR payload.

- **GP card payment (10.05 €)**:
  ```json
  {
    "operation": "CP",
    "requestId": "UUIDv4",
    "amount": 1005,
    "currencyCode": "978",
    "language": "sk"
  }
  ```
  Success yields `result = "0"`, `approvalCode`, `hostTransId`, `customerReceipt`, and `merchantReceipt`.

## 10. Security & Row-Level Security (RLS)
- Enforce RLS per tenant/location; require PIN for Cashier role; audit sensitive actions (copy printing, voids, refunds, VK/VY, ND issuance).
- Cloud connections over TLS; local services operate on loopback/private network using Basic Auth for eKasa.

## 11. Testing Requirements
- Rounding scenarios: pure cash (round up/down, ensure 0.02 → 0.05 conversion), pure card (no rounding), mixed payments.
- VAT 2025 enforcement: block new sales with legacy rates; allow refunds/updates referencing old receipts.
- Voucher/packaging validation: enforce item types, 0% VAT, `"VO:"` prefix for packaging.
- Offline recovery: create PD offline, print offline report, and resend upon reconnection.
- Rejection flow: simulate FS rejection and unlock via `/api/document/update`.
- GP idempotency: repeated `requestId` returns identical outcome; cover `CC`, `CU`, `CR` scenarios.
- Printer coverage: `PrinterCheck`, character size changes, fallback to internal printing when necessary.
