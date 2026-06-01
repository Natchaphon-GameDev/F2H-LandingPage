# Farm to Table House — Landing Page

**Single-page homestay website for a vacation rental @ Khao Yai (ปากช่อง · วังไทร)**

## Project structure

```
index.html              ← MAIN FILE (1814 lines) — all HTML/CSS/JS in one file
google-apps-script.js   ← Backend: deploy to Google Apps Script
preview.html            ← Preview/demo page
images/                 ← 01.jpeg – 32.jpeg + media-baanlaesuan.jpg
```

No build tool, no framework, no npm. Edit `index.html` directly.

## Key config (index.html ~line 1015)

```js
const CONFIG = {
    FACEBOOK_URL: 'https://www.facebook.com/farm2tablefamily/',
    GAS_URL: 'https://script.google.com/macros/s/AKfycbzue1e4KbdMEhYsFcstO0aaKlU529fvXJrCPCabvPGDAgpepZG8w40GvKAtkpbxl9xo0A/exec',
};

const PRICES = {
    weekday: '5,500',   // Mon–Thu  (฿/night)
    weekend: '6,500',   // Fri–Sun + holidays
};
```

To change prices: edit `PRICES` object — it auto-updates the DOM.
To change GAS_URL: re-deploy Google Apps Script → new URL → paste here.

## CSS design tokens (index.html ~line 29)

```css
--green:       #2d6a4f   /* primary brand green */
--green-mid:   #40916c
--green-light: #74c69d
--gold:        #f4c542   /* star color */
--dark:        #1d1d1f
--gray:        #6e6e73
--off-white:   #f5f5f7
--radius:      20px
```

Fonts: `Nunito` (hero editorial, weight 200), `Prompt` (Thai in hero, weight 200), system font everywhere else.

## Multilingual (TH / EN / ZH)

All text uses `data-i18n="key"` attributes. Translation strings live in `const T = { th:{}, en:{}, zh:{} }` (~line 1033). `setLang(lang)` swaps all strings at runtime.

## Page sections (top → bottom)

1. **Nav** — fixed, transparent over hero, white on scroll
2. **Hero** — full-screen image (images/01.jpeg), editorial type, CTA → #pricing
3. **Stats bar** — 10 guests / 3 beds / 3 baths / 4.92★
4. **Pricing** — weekday/weekend price cards → #booking
5. **Booking + Calendar** — interactive calendar + direct booking form + Airbnb button
6. **Highlights** — 3 feature cards
7. **Gallery** — 5-image grid + "See all" modal (32 images)
8. **About** — dark section with feature list
9. **Amenities** — 4-col grid + "Show all" modal with categories
10. **Media/Press** — บ้านและสวน feature + YouTube cards
11. **Rules** — 3-col grid house rules
12. **Location** — Google Maps embed
13. **Contact** — phone + Facebook + channel links (Airbnb, FB)
14. **Footer CTA** — final booking CTA
15. **Footer**

## Booking system

```
User submits form (POST JSON) → doPost() → Google Sheet + blockOnCalendar() immediately
Availability calendar → GET GAS_URL?action=availability → merges Sheet + Airbnb iCal
Status changed in Sheet → onSheetStatusEdit fires → syncDirectToCalendar() immediately
syncAirbnbToSheet (every 6h) → also calls syncDirectToCalendar at the end
```

**Google Sheet**: `Booking` tab — columns: A:Timestamp B:Name C:Phone D:Check-in E:Check-out F:Guests G:Notes H:Facebook I:Source J:Status

**Airbnb listing**: https://www.airbnb.com/rooms/1177471385484136834

**Deployed site**: https://natchaphon-gamedev.github.io/F2H-LandingPage/

**Latest GAS deployment**: `AKfycbxL59EB7Qykb-OsgU-BvJxcjKZ429zqK2UBNuJBlIFCVKap2TFBFD_8AIo2MAU9daaP0g`

## Google Calendar (Airbnb blocking)

Direct bookings block Airbnb via Google Calendar iCal import.

- **Calendar ID**: `65d66736a7caceaca276e2f91e3fdf7d5531f64f7fe5ec940098ffd77eb4b9b2@group.calendar.google.com`
- **GCAL_ID** in `google-apps-script.js` CONFIG section
- Events tagged `[F2H-Direct] YYYY-MM-DDTHH:MM:SS` — **never change EVENT_PREFIX**
- Blocking logic: `Confirmed` = block · `Cancelled` = unblock · `Pending` = nothing · `Airbnb` = skip

## GAS functions reference

| Function | How triggered | What it does |
|----------|--------------|--------------|
| `doPost` | HTTP POST from site | Save booking + block calendar immediately |
| `doGet?action=availability` | HTTP GET from site | Return booked dates for calendar UI |
| `syncAirbnbToSheet` | Every 6h trigger | Pull Airbnb iCal → Sheet + calls syncDirectToCalendar |
| `syncDirectToCalendar` | Every 1h trigger | Reconcile Sheet → Google Calendar |
| `onSheetStatusEdit` | Sheet onEdit trigger | Fires on col J edit → syncDirectToCalendar immediately |
| `installSheetEditTrigger` | Run once manually | Install sheet onEdit trigger programmatically |
| `blockOnCalendar` | Called by doPost | Create single calendar event immediately |
| `setupSheet` | Run once manually | Create Sheet headers + formatting |

## Google Apps Script setup (first time)

1. script.google.com → New Project → paste `google-apps-script.js`
2. `SHEET_ID`, `AIRBNB_ICAL`, `GCAL_ID` already set in file
3. Deploy → Web app (Execute as: Me, Anyone can access) → copy URL → paste into `CONFIG.GAS_URL`
4. Run `setupSheet()` once from editor
5. Run `installSheetEditTrigger()` once from editor
6. Add Time triggers:
   - `syncAirbnbToSheet` → every 6 hours
   - `syncDirectToCalendar` → every 1 hour
7. Airbnb → Calendar → Import calendar → paste the Google Calendar public iCal URL

## Common edit tasks

| Task | Where |
|------|-------|
| Change prices | `PRICES` object ~line 1023 |
| Change phone number | search `tel:` in index.html |
| Add/remove image from gallery | `gallery-grid` section + `gallery-photo-grid` in modal |
| Edit Thai/English/Chinese text | `const T` object ~line 1033 |
| Update Airbnb link | search `airbnb.com/rooms` |
| Update Facebook link | `CONFIG.FACEBOOK_URL` ~line 1016 |
| Change GAS endpoint | `CONFIG.GAS_URL` ~line 1017 |
