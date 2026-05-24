# Booking Notification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a guest submits a booking, fire three notifications — guest confirmation email, host alert email, and host LINE Notify message — all from Google Apps Script, with zero impact on booking reliability.

**Architecture:** The booking form gains an optional email field; the frontend payload already POSTs JSON to GAS, so only the payload object needs `email` added. All three notification functions live in `google-apps-script.js` and are called sequentially at the end of `doPost()`, each wrapped in try/catch so a failed notification never blocks the booking save.

**Tech Stack:** Vanilla HTML/CSS/JS (no build tool), Google Apps Script (`MailApp`, `UrlFetchApp`), LINE Notify API (`https://notify-api.line.me/api/notify`)

---

## File Map

| File | Changes |
|------|---------|
| `index.html` | Add email `<input>` field in form HTML (after phone); add `bk.form.email` i18n key in TH/EN/ZH; add `email` to JS payload object |
| `google-apps-script.js` | Add `OWNER_EMAIL` + `LINE_NOTIFY_TOKEN` constants; add `COL_EMAIL = 10`; update `setupSheet()` headers + column width; update `doPost` appendRow (11 cols); add `sendGuestConfirmation`, `sendHostEmail`, `sendHostLine`; wire all three into `doPost` |

---

## Task 1: Add Email Field to Booking Form

**Files:**
- Modify: `index.html:950–954` (after phone field, before Facebook field)
- Modify: `index.html:1094–1095` (TH i18n)
- Modify: `index.html:1181–1182` (EN i18n)
- Modify: `index.html:1268–1269` (ZH i18n)
- Modify: `index.html:1586–1592` (JS payload)

- [ ] **Step 1: Add the email input field to the form HTML**

In `index.html`, find the block ending at line 950 (`</div>` closing the name/phone `form-row`). Add this new `form-group` div **between** the closing `</div>` of the name/phone row (line 950) and the opening `<div class="form-group">` of the Facebook field (line 951):

```html
            <div class="form-group">
                <label data-i18n="bk.form.email">อีเมล (สำหรับยืนยันการจอง)</label>
                <input type="email" id="bk-email" autocomplete="email" placeholder="your@email.com">
            </div>
```

After the edit, the form section should read (lines ~950–954):
```html
            </div>
            <div class="form-group">
                <label data-i18n="bk.form.email">อีเมล (สำหรับยืนยันการจอง)</label>
                <input type="email" id="bk-email" autocomplete="email" placeholder="your@email.com">
            </div>
            <div class="form-group">
                <label data-i18n="bk.form.facebook">Facebook</label>
```

- [ ] **Step 2: Add TH i18n key**

In `index.html`, find the TH section line that reads:
```js
            'bk.form.name': 'ชื่อ-นามสกุล', 'bk.form.phone': 'เบอร์โทรศัพท์',
```
Change it to:
```js
            'bk.form.name': 'ชื่อ-นามสกุล', 'bk.form.phone': 'เบอร์โทรศัพท์',
            'bk.form.email': 'อีเมล (สำหรับยืนยันการจอง)',
```

- [ ] **Step 3: Add EN i18n key**

In `index.html`, find the EN section line that reads:
```js
            'bk.form.name': 'Full Name', 'bk.form.phone': 'Phone Number',
```
Change it to:
```js
            'bk.form.name': 'Full Name', 'bk.form.phone': 'Phone Number',
            'bk.form.email': 'Email (for booking confirmation)',
```

- [ ] **Step 4: Add ZH i18n key**

In `index.html`, find the ZH section line that reads:
```js
            'bk.form.name': '姓名', 'bk.form.phone': '手机号码',
```
Change it to:
```js
            'bk.form.name': '姓名', 'bk.form.phone': '手机号码',
            'bk.form.email': '邮箱（用于预订确认）',
```

- [ ] **Step 5: Add email to the JS form payload**

In `index.html`, find the payload object (~line 1585):
```js
        const payload = {
            name:     document.getElementById('bk-name').value.trim(),
            phone:    document.getElementById('bk-phone').value.trim(),
            checkIn:  ci, checkOut: co,
            guests:   document.getElementById('bk-guests').value,
            notes:    document.getElementById('bk-notes').value.trim(),
            facebook: document.getElementById('bk-facebook').value.trim(),
        };
```
Change it to:
```js
        const payload = {
            name:     document.getElementById('bk-name').value.trim(),
            phone:    document.getElementById('bk-phone').value.trim(),
            email:    document.getElementById('bk-email').value.trim(),
            checkIn:  ci, checkOut: co,
            guests:   document.getElementById('bk-guests').value,
            notes:    document.getElementById('bk-notes').value.trim(),
            facebook: document.getElementById('bk-facebook').value.trim(),
        };
```

- [ ] **Step 6: Verify in browser**

Open `index.html` directly in a browser (no server needed). Click **สนใจจอง** to open the booking modal. Confirm:
- Email field appears between the phone field and the Facebook field
- Switching language (TH/EN/ZH) updates the email label text
- Field is not marked required (can be left blank)

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add email field to booking form with i18n"
```

---

## Task 2: GAS — CONFIG, Constants, Sheet Schema, doPost Row

**Files:**
- Modify: `google-apps-script.js:32–48` (CONFIG + column constants)
- Modify: `google-apps-script.js:133–159` (doPost appendRow)
- Modify: `google-apps-script.js:388–425` (setupSheet)

- [ ] **Step 1: Add OWNER_EMAIL and LINE_NOTIFY_TOKEN constants**

In `google-apps-script.js`, find:
```js
const GCAL_ID = '65d66736a7caceaca276e2f91e3fdf7d5531f64f7fe5ec940098ffd77eb4b9b2@group.calendar.google.com';
```
Add two lines immediately after it:
```js
const OWNER_EMAIL       = 'nsp.gamedev@gmail.com';
const LINE_NOTIFY_TOKEN = ''; // paste your token from notify-bot.line.me
```

- [ ] **Step 2: Add COL_EMAIL constant**

In `google-apps-script.js`, find:
```js
const COL_STATUS   = 9;
```
Add one line immediately after it:
```js
const COL_EMAIL    = 10;
```

- [ ] **Step 3: Update doPost appendRow to include email (column K)**

In `google-apps-script.js`, find the `appendRow` call inside `doPost`:
```js
    sheet.appendRow([
      ts,                  // A: Timestamp
      data.name     || '', // B: Name
      data.phone    || '', // C: Phone
      data.checkIn  || '', // D: Check-in
      data.checkOut || '', // E: Check-out
      data.guests   || '', // F: Guests
      data.notes    || '', // G: Notes
      data.facebook || '', // H: Facebook
      'Direct',            // I: Source
      'Pending',           // J: Status
    ]);
```
Replace it with:
```js
    sheet.appendRow([
      ts,                  // A: Timestamp
      data.name     || '', // B: Name
      data.phone    || '', // C: Phone
      data.checkIn  || '', // D: Check-in
      data.checkOut || '', // E: Check-out
      data.guests   || '', // F: Guests
      data.notes    || '', // G: Notes
      data.facebook || '', // H: Facebook
      'Direct',            // I: Source
      'Pending',           // J: Status
      data.email    || '', // K: Email
    ]);
```

- [ ] **Step 4: Update setupSheet to add column K header and width**

In `google-apps-script.js`, find:
```js
  const headers = ['Timestamp','Name','Phone','Check-in','Check-out','Guests','Notes','Facebook','Source','Status'];
```
Replace with:
```js
  const headers = ['Timestamp','Name','Phone','Check-in','Check-out','Guests','Notes','Facebook','Source','Status','Email'];
```

Then find the last `setColumnWidth` call in `setupSheet`:
```js
  sheet.setColumnWidth(10, 110);
```
Add one line after it:
```js
  sheet.setColumnWidth(11, 200);
```

- [ ] **Step 5: Test in GAS Editor**

In the Apps Script editor (script.google.com), select function `setupSheet` from the dropdown and click **Run**. Then open the Google Sheet and confirm:
- Column K header reads **Email**
- Column K is ~200px wide
- Existing rows are unaffected (column K is blank for them)

- [ ] **Step 6: Commit**

```bash
git add google-apps-script.js
git commit -m "feat: add email column K to GAS sheet schema and doPost row"
```

---

## Task 3: GAS — sendGuestConfirmation

**Files:**
- Modify: `google-apps-script.js` — add function after `blockOnCalendar`; wire into `doPost`

- [ ] **Step 1: Add sendGuestConfirmation function**

In `google-apps-script.js`, add this function immediately after the closing `}` of `blockOnCalendar` (around line 368):

```js
// ──────────────────────────────────────────────
// ส่งอีเมลยืนยันให้ผู้จอง (ถ้ามี email)
// ──────────────────────────────────────────────
function sendGuestConfirmation(data) {
  if (!data.email) return;
  try {
    const subject = 'ยืนยันคำขอจอง — Farm to Table House';
    const body = [
      'สวัสดีคุณ ' + data.name + ',',
      '',
      'ขอบคุณที่สนใจจอง Farm to Table House @ เขาใหญ่',
      'เราได้รับคำขอจองของคุณแล้ว รายละเอียดดังนี้:',
      '',
      '📅 วันเข้าพัก: ' + data.checkIn,
      '📅 วันออก:    ' + data.checkOut,
      '👥 จำนวนผู้เข้าพัก: ' + data.guests,
      '',
      'เจ้าของบ้านจะติดต่อกลับเพื่อยืนยันผ่าน Facebook หรือโทรศัพท์',
      '',
      '---',
      'Hello ' + data.name + ',',
      '',
      'Thank you for your interest in Farm to Table House @ Khao Yai.',
      'We have received your booking request:',
      '',
      '📅 Check-in:  ' + data.checkIn,
      '📅 Check-out: ' + data.checkOut,
      '👥 Guests:    ' + data.guests,
      '',
      'The host will contact you to confirm via Facebook or phone.',
      '',
      '---',
      '📞 081-431-5092',
      '👍 https://www.facebook.com/farm2tablefamily/',
    ].join('\n');

    MailApp.sendEmail(data.email, subject, body);
    Logger.log('Guest confirmation sent to: ' + data.email);
  } catch (err) {
    Logger.log('sendGuestConfirmation error: ' + err.message);
  }
}
```

- [ ] **Step 2: Wire sendGuestConfirmation into doPost**

In `google-apps-script.js`, find the `doPost` function. After the line:
```js
    blockOnCalendar(ts, data.checkIn || '', data.checkOut || '', data.name || 'Guest');
```
Add:
```js
    sendGuestConfirmation(data);
```

- [ ] **Step 3: Test in GAS Editor**

In the Apps Script editor, add a temporary test function at the bottom of the file:

```js
function testGuestConfirmation() {
  sendGuestConfirmation({
    name: 'Test Guest',
    email: 'nsp.gamedev@gmail.com', // use your own email to receive the test
    checkIn: '2026-06-15',
    checkOut: '2026-06-17',
    guests: '1-4',
  });
}
```

Select `testGuestConfirmation` from the dropdown, click **Run**, grant mail permissions when prompted, then check your inbox for the confirmation email. Verify:
- Subject: `ยืนยันคำขอจอง — Farm to Table House`
- Body contains check-in/out dates and phone number
- Both Thai and English text present

Delete the test function after verifying.

- [ ] **Step 4: Commit**

```bash
git add google-apps-script.js
git commit -m "feat: add sendGuestConfirmation — email confirmation to guest on booking"
```

---

## Task 4: GAS — sendHostEmail

**Files:**
- Modify: `google-apps-script.js` — add function after `sendGuestConfirmation`; wire into `doPost`

- [ ] **Step 1: Add sendHostEmail function**

In `google-apps-script.js`, add this function immediately after the closing `}` of `sendGuestConfirmation`:

```js
// ──────────────────────────────────────────────
// ส่งอีเมลแจ้งเจ้าของบ้านทุกครั้งที่มีการจองใหม่
// ──────────────────────────────────────────────
function sendHostEmail(data, ts) {
  try {
    const subject = '🏡 การจองใหม่: ' + (data.name || 'Guest') +
                    ' · ' + data.checkIn + ' → ' + data.checkOut;
    const sheetUrl = 'https://docs.google.com/spreadsheets/d/' + SHEET_ID;
    const body = [
      '🏡 มีคำขอจองใหม่!',
      '',
      '⏰ เวลา:               ' + ts,
      '👤 ชื่อ:               ' + (data.name     || '-'),
      '📞 โทร:               ' + (data.phone    || '-'),
      '📧 อีเมล:             ' + (data.email    || '-'),
      '👍 Facebook:          ' + (data.facebook || '-'),
      '',
      '📅 วันเข้าพัก:        ' + (data.checkIn  || '-'),
      '📅 วันออก:            ' + (data.checkOut || '-'),
      '👥 จำนวนผู้เข้าพัก:  ' + (data.guests   || '-'),
      '📝 หมายเหตุ:          ' + (data.notes    || '-'),
      '',
      '🔗 จัดการสถานะใน Google Sheet:',
      sheetUrl,
    ].join('\n');

    MailApp.sendEmail(OWNER_EMAIL, subject, body);
    Logger.log('Host email sent to: ' + OWNER_EMAIL);
  } catch (err) {
    Logger.log('sendHostEmail error: ' + err.message);
  }
}
```

- [ ] **Step 2: Wire sendHostEmail into doPost**

In `google-apps-script.js`, find the line you added in Task 3:
```js
    sendGuestConfirmation(data);
```
Add one line after it:
```js
    sendHostEmail(data, ts);
```

- [ ] **Step 3: Test in GAS Editor**

Add a temporary test function at the bottom of the file:

```js
function testHostEmail() {
  sendHostEmail({
    name: 'Test Guest',
    phone: '081-111-2222',
    email: 'test@example.com',
    facebook: 'facebook.com/testguest',
    checkIn: '2026-06-15',
    checkOut: '2026-06-17',
    guests: '1-4',
    notes: 'มีเด็กเล็ก 1 คน',
  }, new Date());
}
```

Select `testHostEmail`, click **Run**, check `nsp.gamedev@gmail.com` inbox. Verify:
- Subject starts with `🏡 การจองใหม่:`
- Body shows all booking fields
- Sheet link present

Delete the test function after verifying.

- [ ] **Step 4: Commit**

```bash
git add google-apps-script.js
git commit -m "feat: add sendHostEmail — notify host by email on every new booking"
```

---

## Task 5: GAS — sendHostLine

**Files:**
- Modify: `google-apps-script.js` — add function after `sendHostEmail`; wire into `doPost`

- [ ] **Step 1: Get your LINE Notify token (one-time setup)**

1. Open [notify-bot.line.me](https://notify-bot.line.me) in a browser
2. Click **Login** → log in with your LINE account
3. Click your profile name (top right) → **My page**
4. Scroll to **Generate token**, click it
5. Token name: `F2H Bookings`
6. Select: **1-on-1 chat with LINE Notify** (sends to your own LINE chat)
7. Click **Generate token**
8. Copy the token immediately (it's shown only once)
9. In `google-apps-script.js`, find:
   ```js
   const LINE_NOTIFY_TOKEN = ''; // paste your token from notify-bot.line.me
   ```
   Replace the empty string with your token:
   ```js
   const LINE_NOTIFY_TOKEN = 'your_actual_token_here';
   ```

- [ ] **Step 2: Add sendHostLine function**

In `google-apps-script.js`, add this function immediately after the closing `}` of `sendHostEmail`:

```js
// ──────────────────────────────────────────────
// ส่ง LINE Notify ให้เจ้าของบ้านทุกครั้งที่มีการจองใหม่
// ข้ามถ้า LINE_NOTIFY_TOKEN ว่างไว้
// ──────────────────────────────────────────────
function sendHostLine(data) {
  if (!LINE_NOTIFY_TOKEN) return;
  try {
    const message = '\n🏡 จองใหม่! ' + (data.name || 'Guest') +
                    ' · ' + data.checkIn + '→' + data.checkOut +
                    ' · ' + data.guests + ' คน · 📞 ' + (data.phone || '-');
    UrlFetchApp.fetch('https://notify-api.line.me/api/notify', {
      method:  'post',
      headers: { 'Authorization': 'Bearer ' + LINE_NOTIFY_TOKEN },
      payload: { message: message },
    });
    Logger.log('LINE Notify sent');
  } catch (err) {
    Logger.log('sendHostLine error: ' + err.message);
  }
}
```

- [ ] **Step 3: Wire sendHostLine into doPost**

In `google-apps-script.js`, find the line you added in Task 4:
```js
    sendHostEmail(data, ts);
```
Add one line after it:
```js
    sendHostLine(data);
```

The full notification block in `doPost` should now read:
```js
    blockOnCalendar(ts, data.checkIn || '', data.checkOut || '', data.name || 'Guest');
    sendGuestConfirmation(data);
    sendHostEmail(data, ts);
    sendHostLine(data);

    return jsonResponse({ ok: true });
```

- [ ] **Step 4: Test in GAS Editor**

Add a temporary test function at the bottom of the file:

```js
function testHostLine() {
  sendHostLine({
    name: 'Test Guest',
    phone: '081-111-2222',
    checkIn: '2026-06-15',
    checkOut: '2026-06-17',
    guests: '1-4',
  });
}
```

Select `testHostLine`, click **Run**. Check your LINE app — you should receive a message from **LINE Notify** within a few seconds. Verify the message matches:
```
🏡 จองใหม่! Test Guest · 2026-06-15→2026-06-17 · 1-4 คน · 📞 081-111-2222
```

Delete the test function after verifying.

- [ ] **Step 5: Commit**

```bash
git add google-apps-script.js
git commit -m "feat: add sendHostLine — LINE Notify push to host on every new booking"
```

---

## Task 6: Deploy and End-to-End Verification

**Files:** No code changes — GAS redeploy + live test only.

- [ ] **Step 1: Redeploy GAS as a new version**

In the Apps Script editor:
1. Click **Deploy** → **Manage deployments**
2. Click the pencil ✏️ icon on your current Web app deployment
3. Under **Version**, select **New version**
4. Click **Deploy**
5. Copy the new deployment URL (ends in `/exec`)
6. In `index.html`, find:
   ```js
   GAS_URL: 'https://script.google.com/macros/s/AKfycbwxv2wXHYYk-BLNzW3-bSEtV8nF9ZmsPUMqyfr7v0hEXbiuga7STWJntnldJwDSCT2QBA/exec',
   ```
   Replace the URL with the new one.
7. Commit:
   ```bash
   git add index.html
   git commit -m "config: update GAS_URL to new deployment with notification support"
   ```

- [ ] **Step 2: End-to-end test**

Open `index.html` in a browser. Submit a test booking:
- Name: `ทดสอบ ระบบ`
- Phone: `081-431-5092`
- **Email: nsp.gamedev@gmail.com** (your own email, to receive both host + guest emails)
- Check-in: any future date
- Check-out: the day after

Verify all 5 outcomes within 30 seconds:
1. ✅ On-screen: form hides and success message `ส่งคำขอจองแล้ว!` shows
2. ✅ Google Sheet: new row appears with email in column K
3. ✅ Google Calendar: new `[F2H-Direct]` event created for the dates
4. ✅ Inbox `nsp.gamedev@gmail.com`: guest confirmation email arrives (subject: `ยืนยันคำขอจอง — Farm to Table House`)
5. ✅ Inbox `nsp.gamedev@gmail.com`: host notification email arrives (subject: `🏡 การจองใหม่: ทดสอบ ระบบ · ...`)
6. ✅ LINE app: LINE Notify message arrives with booking summary

- [ ] **Step 3: Test with no email (optional field verification)**

Submit another test booking leaving the email field blank. Verify:
- Booking saves to Sheet ✅
- Host email arrives ✅
- LINE Notify arrives ✅
- No guest confirmation email is sent (check GAS Execution log: should show no `sendGuestConfirmation` line) ✅

- [ ] **Step 4: Clean up the test rows**

Open the Google Sheet, delete the two test booking rows. Open Google Calendar, delete the two test `[F2H-Direct]` events.
