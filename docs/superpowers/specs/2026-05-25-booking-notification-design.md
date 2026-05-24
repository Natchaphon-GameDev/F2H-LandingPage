# Booking Notification Design
**Farm to Table House — Landing Page**  
Date: 2026-05-25

## Overview

When a guest submits a booking request via the website form, three notifications fire automatically:

1. **Guest confirmation email** — sent to the guest's email address (if provided)
2. **Host notification email** — sent to `nsp.gamedev@gmail.com` with full booking details
3. **Host LINE Notify message** — a short push notification to the host's LINE app

All notifications are triggered inside `doPost()` in `google-apps-script.js`, after the booking row is saved to the Google Sheet and the Google Calendar is blocked.

---

## Scope of Changes

| File | Change |
|------|--------|
| `index.html` | Add email input field to booking form + i18n labels |
| `google-apps-script.js` | Add `OWNER_EMAIL`, `LINE_NOTIFY_TOKEN` to CONFIG; add email column to sheet; add 3 notification functions called from `doPost` |

---

## 1. Booking Form (index.html)

### New field: Email

- Placed directly below the phone number field
- Input type: `email`
- **Optional** (not required) — guests without email can still book
- Submitted in the POST payload as `email`

### i18n labels (`const T`)

| Key | TH | EN | ZH |
|-----|----|----|-----|
| `bk.email` | อีเมล (สำหรับยืนยันการจอง) | Email (for booking confirmation) | 邮箱（用于预订确认） |
| `bk.email.ph` | your@email.com | your@email.com | your@email.com |

---

## 2. Google Sheet (google-apps-script.js)

### New column K: Email

- Added after the existing **J: Status** column
- `setupSheet()` updated to include `'Email'` in the headers array and set column K width to 200px
- Existing rows unaffected — column stays blank for old bookings
- Column index constant added: `const COL_EMAIL = 10;`

---

## 3. GAS CONFIG Additions

```js
const OWNER_EMAIL        = 'nsp.gamedev@gmail.com';
const LINE_NOTIFY_TOKEN  = 'YOUR_TOKEN_HERE'; // from notify-bot.line.me
```

`LINE_NOTIFY_TOKEN` defaults to empty string `''` — if empty, LINE notification is silently skipped.

---

## 4. Notification Functions

### 4.1 `sendGuestConfirmation(data)`

- **Triggered by:** `doPost`, only when `data.email` is a non-empty string
- **Channel:** Email via `MailApp.sendEmail()`
- **Subject:** `ยืนยันคำขอจอง — Farm to Table House`
- **Body:** Bilingual (Thai + English) booking summary:
  - Guest name, check-in date, check-out date, number of guests
  - Note: "เจ้าของบ้านจะติดต่อกลับเพื่อยืนยัน / The host will contact you to confirm."
  - Farm to Table House contact: Facebook URL + phone
- **Error handling:** Wrapped in try/catch; failure is logged, never throws

### 4.2 `sendHostEmail(data, ts)`

- **Triggered by:** `doPost`, always fires
- **Channel:** Email via `MailApp.sendEmail()` to `OWNER_EMAIL`
- **Subject:** `🏡 การจองใหม่: [Name] · [Check-in] → [Check-out]`
- **Body:** Full booking details:
  - Timestamp, name, phone, email, Facebook, check-in, check-out, guests, notes
  - Direct link to Google Sheet for status management
- **Error handling:** Wrapped in try/catch; failure is logged, never throws

### 4.3 `sendHostLine(data)`

- **Triggered by:** `doPost`, fires only when `LINE_NOTIFY_TOKEN` is non-empty
- **Channel:** LINE Notify API via `UrlFetchApp.fetch('https://notify-api.line.me/api/notify', ...)`
- **Message:** Single-line summary:
  ```
  🏡 จองใหม่! [Name] · [Check-in]→[Check-out] · [Guests] คน · 📞 [Phone]
  ```
- **Error handling:** Wrapped in try/catch; failure is logged, never throws

---

## 5. Updated `doPost` Sequence

```
POST from booking form
  ↓
Parse JSON body
  ↓
appendRow to Sheet (cols A–K, including email in K)
  ↓
blockOnCalendar(ts, checkIn, checkOut, name)
  ↓
sendGuestConfirmation(data)   ← skipped if data.email is empty
  ↓
sendHostEmail(data, ts)       ← always fires
  ↓
sendHostLine(data)            ← skipped if LINE_NOTIFY_TOKEN is empty
  ↓
return { ok: true }
```

**Key principle:** Notifications are best-effort. Any notification failure is caught, logged via `Logger.log()`, and does not affect the booking save or the `{ ok: true }` response.

---

## 6. LINE Notify Setup (one-time, ~2 minutes)

1. Go to [notify-bot.line.me](https://notify-bot.line.me) → log in with LINE account
2. **My page** → **Generate token**
3. Token name: `F2H Bookings`
4. Notify: **1-on-1 chat with LINE Notify** (sends to your personal LINE)
5. Copy the generated token
6. Paste into `LINE_NOTIFY_TOKEN` in `google-apps-script.js`
7. Re-deploy the GAS script (New deployment → Web app)

---

## 7. Error Handling Summary

| Scenario | Behaviour |
|----------|-----------|
| Guest submits form with no email | Guest confirmation skipped silently; host still notified |
| `LINE_NOTIFY_TOKEN` is empty/not set | LINE notification skipped silently |
| MailApp quota exceeded (100/day free) | Error logged; booking still saved; `ok: true` returned |
| LINE API error (bad token, network) | Error logged; booking still saved; `ok: true` returned |
| Any notification throws | try/catch prevents it from breaking the booking flow |

---

## 8. Out of Scope

- Status-change notifications (e.g., host confirms → guest notified) — future enhancement
- SMS notifications
- WhatsApp notifications
- Notification when Airbnb booking syncs in
