# Estimate-to-Invoice Conversion Flow

A walkthrough of how estimate→invoice conversion works across mobile and web.

---

## Overview

There are **two platforms** (mobile, web) and **two trigger points** on mobile. All paths funnel into the same core conversion logic, with minor behavioral differences.

---

## Mobile

### Trigger 1: "Make Invoice" button (manual)

The button lives at the bottom of the Estimate editor screen (visible in Edit tab).

**Stack:**
```
MakeInvoiceButton (make-invoice-button.tsx)
  → makeInvoice() in invoice.navigator.tsx:1034
    → withPaywallProtection() — checks subscription, shows upgrade if needed
      → convertEstimateToInvoice() in invoice.navigator.tsx:1044
```

**`convertEstimateToInvoice` does 5 things in order:**

1. **Calls `makeInvoiceFromEstimate()`** — builds the new invoice object in memory (no save yet)
2. **Marks estimate as `fullyPaid: true`** and saves it via `handleSave()`
3. **Saves the new invoice** via `saveInvoice(syncStore, newInvoice)`
4. **Updates `InvoiceLastNo`** in settings so the next invoice increments correctly
5. **Navigates**: `pop(2)` off the estimate, then opens the new invoice editor

---

### Trigger 2: Post-email modal (A/B experiment)

A one-time "You're on a roll! Convert to invoice" modal that appears after emailing an estimate.

**Stack:**
```
invoice-email.screen.tsx:551
  → dialogsStore.maybeShowConvertEstimateDialog(estimate.remoteId)
    → checks hasShownConvertEstimateModal() in AsyncStorage
      → if not shown: sets shouldShowConvertEstimateDialog = true
        → ConvertEstimateToInvoiceModal (always mounted in ModalManager) becomes visible
```

**Gate conditions** (all must be true):
- `isTrialAllSetExperimentEnabled` Optimizely flag is on (A/B, ~50/50 split)
- Document being emailed is an estimate (not an invoice)
- Modal has never been shown on this device (one-time, stored in AsyncStorage)

**`handleConvert` in the modal does:**
1. Calls `makeInvoiceFromEstimate()` — same function as Trigger 1
2. Saves the new invoice via `saveInvoice(syncStore, newInvoice)`
3. Navigates to the new invoice

**Notable gaps vs Trigger 1:**
- No paywall check
- Does NOT mark estimate as `fullyPaid`
- Does NOT update `InvoiceLastNo` setting

---

### Core conversion: `makeInvoiceFromEstimate` (use-cases.ts:22)

This is the heart of mobile conversion. Takes the estimate, returns a new invoice object (not saved).

```
makeInvoiceFromEstimate(estimate, userInfo, intl)
  1. Pulls fields to carry over: client, company, items, payments, poNumber,
     signature, signDate, subtotal, photos
  2. Resolves the notes/comment:
     - parseComment() checks if estimate's comment contains the default estimate note
     - If yes: swaps it for the default invoice note
     - If custom note: appends the default invoice note below it
  3. Duplicates photos + signature in Realm (new remoteIds, same content)
  4. Builds new Invoice object:
     - Base: defaultInvoice(DOCTYPE_INVOICE) — fresh remoteId, invoice defaults
     - Spreads estimate fields on top
     - Settings merge:
       - Start from defaultInvoiceSetting
       - Spread estimate.setting over it
       - Strip estimate-specific fields:
         estimateSignatureRequired, estimateSignedText, estimateSignedAt,
         feesType, depositType, depositRate, depositAmount, termsDay
       - Set estimateId = estimate.remoteId (back-reference)
       - Set fullyPaid = false
  5. Fires analytics event: 'estimate-to-invoice'
  6. Calls updateTotals() — recalculates tax, balanceDue, etc.
  7. Returns new invoice (caller is responsible for saving)
```

---

### Save: `saveInvoice` (repository.ts:46)

```
saveInvoice(syncStore, invoice)
  1. Opens a Realm write transaction
  2. db.create(RInvoiceTableName, invoice, UpdateMode.All)
     — inserts if new, overwrites if remoteId exists
  3. After local write resolves: syncStore.syncFireAndForget()
     — kicks off server sync in background, doesn't wait
```

**Local-first**: invoice is immediately in Realm and visible in the app. Sync to Parse Server happens asynchronously.

---

### Invoice numbering

- When `defaultInvoice()` is called, `incrementLastInvoiceNo()` runs immediately
- Reads last used number from Realm settings (e.g. `InvoiceLastNo = INV0012`)
- Falls back to `INV0000` if nothing stored
- Splits into prefix (`INV`) + number (`0012`), increments, pads back to same length → `INV0013`
- Estimate counter (`EstimateLastNo`) is completely separate
- After conversion, Trigger 1 writes the new invoice number back to `InvoiceLastNo`
  (Trigger 2 / modal skips this step — potential bug)

---

### Navigation after Trigger 1

```
navigation.dispatch(StackActions.pop(2))   // off the estimate editor
navigateToInvoice({
  docType: DOCTYPE_INVOICE,
  remoteId: newInvoice.remoteId,
  convertedDocType: true,         // tells invoice screen to show "converted" banner
  convertedDocId: estimate.invoiceNo  // e.g. "EST0003"
})
```

---

## Web

### Trigger: "Make Invoice" button

Entry points (both call `handleMakeInvoice`):
- `ConvertToInvoiceButton.tsx` — button inside the invoice editor toolbar
- `InvoiceRow.tsx` — "Make Invoice" item in the kebab menu on the document list

**`handleMakeInvoice` (estimateToInvoice.ts:24):**
1. Checks `user.canCreateNewDoc(DOCTYPE_INVOICE)` — doc limit / subscription check
2. Calls `estimate.convertEstimate()` on the InvoiceModel
3. Fires analytics events (`estimate-to-invoice`, `invoice-create`)
4. Navigates to new invoice with `?email=true` — opens ready to send

---

### Core conversion: `InvoiceModel.convertEstimate` (InvoiceModel.ts:1016)

Two-step: `_convertEstimateData()` builds the data, then one of two paths assembles the invoice.

**`_convertEstimateData` (line 1296):**
```
- docType → DOCTYPE_INVOICE
- setting: new remoteId + estimateId = estimate.remoteId
- company, client, items, photos: all get new remoteIds
- items: visibleItems only (hidden items excluded — differs from mobile)
- updatedAt: new Date()
```

**Two paths based on notes:**

**Path A — custom note** (estimate has non-default comment):
- Keeps the estimate's comment as-is
- Marks estimate `fullyPaid = true`, saves estimate
- Saves new invoice

**Path B — no custom note / default note**:
- Merges `invoiceDefaults.setting` on top of converted data
  (user's default invoice settings win — tax rate, discount, etc.)
- Marks estimate `fullyPaid = true`, saves estimate
- Saves new invoice

Web's comment handling is binary (keep or overwrite), vs mobile which does a smarter note swap.

---

## Key Differences: Mobile vs Web

| | Mobile (Trigger 1) | Mobile (Trigger 2 / modal) | Web |
|---|---|---|---|
| **Paywall check** | Yes | No | Yes (doc limit) |
| **Marks estimate fullyPaid** | Yes (caller) | No | Yes (inside model) |
| **Hidden items** | Included | Included | Excluded |
| **Notes handling** | Smart swap (parseComment) | Smart swap | Binary (keep or overwrite) |
| **Invoice number update** | Yes | No (bug?) | Server-side (Parse) |
| **Save mechanism** | Realm → async Parse sync | Realm → async Parse sync | Parse SDK direct |
| **After save** | Opens invoice editor | Opens invoice editor | Opens invoice + email flow |
| **Deposit fields stripped** | Yes (explicit omit) | Yes | N/A (don't exist on estimates) |

---

## Files Reference

| File | What |
|------|------|
| `is-mobile/src/features/documents/components/invoice-actions/make-invoice-button.tsx` | "Make Invoice" button UI |
| `is-mobile/src/features/documents/navigators/invoice.navigator.tsx:1034` | Trigger 1 entry — paywall + conversion orchestration |
| `is-mobile/src/features/documents/modals/convert-estimate-to-invoice-modal.tsx` | Trigger 2 modal UI + handleConvert |
| `is-mobile/src/features/documents/screens/invoice-email.screen.tsx:551` | Where Trigger 2 fires after email send |
| `is-mobile/src/stores/Dialogs.store.ts:245` | `maybeShowConvertEstimateDialog` — one-time guard |
| `is-mobile/src/features/first-time-user-experience/convert-estimate-utils.ts` | AsyncStorage read/write for one-time guard |
| `is-mobile/src/ui/modal/modal-manager.tsx:96` | Where modal is mounted in the app |
| `is-mobile/src/services/realm/entities/invoice/use-cases.ts:22` | `makeInvoiceFromEstimate` — core mobile conversion |
| `is-mobile/src/services/realm/entities/invoice/repository.ts:46` | `saveInvoice` — Realm write + sync |
| `is-mobile/src/services/realm/entities/settings/repository.ts:164` | `incrementLastInvoiceNo` — invoice numbering |
| `is-web-app/client/src/util/estimateToInvoice.ts:24` | Web entry — `handleMakeInvoice` |
| `is-web-app/client/src/models/InvoiceModel.ts:1016` | `convertEstimate` — web conversion + two note paths |
| `is-web-app/client/src/models/InvoiceModel.ts:1296` | `_convertEstimateData` — web field mapping |
| `is-web-app/client/src/components/Invoice/ConvertToInvoiceButton.tsx` | Web "Make Invoice" button (editor toolbar) |
| `is-web-app/client/src/components/DocList/InvoiceRow.tsx:241` | Web "Make Invoice" (doc list kebab menu) |
