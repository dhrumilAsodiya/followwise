# DESIGN.md: UI/UX design system for FollowWise

Status: PROPOSED. Values below are starting defaults. Confirm or change them at Phase 0, step 4, then keep this file as the single source for UI decisions. Do not introduce new colors, fonts, or components without updating this file.

## 1. Principles
1. **Calm and trustworthy.** These screens hold money and documents. Plain, quiet, no decoration for its own sake.
2. **Scannable in 5 seconds.** The owner should see who is late and what is due without clicking.
3. **Mobile first.** Landlords and tenants use phones. Every screen works at 360 px wide.
4. **Status is never color alone.** Every status has a text label and an icon as well as a color.
5. **Polite by default.** Reminders and portal wording are short, friendly, and never threatening.
6. **Sell the vertical.** Screens say "Tenant", "Unit", "Client", never "Party" or "Record". Labels come from the App Definition.

## 2. Stack (proposed)
Tailwind CSS plus shadcn/ui components (accessible, free, copied into the repo). Icons: lucide-react. Charts: keep to simple bars and numbers in v1. Confirm in Phase 0, step 4.

## 3. Design tokens
### Color
| Token | Value | Use |
|---|---|---|
| `primary` | `#2F4BC0` | Main buttons, links, active nav |
| `primary-foreground` | `#FFFFFF` | Text on primary |
| `background` | `#FFFFFF` | Page |
| `surface` | `#F6F7F9` | Cards, table headers |
| `border` | `#E2E5EA` | Dividers |
| `text` | `#1B1F27` | Body |
| `text-muted` | `#5B6472` | Secondary text |
| `success` | `#1E7A46` | Paid, approved, complete |
| `warning` | `#9A6200` | Due soon, expiring |
| `danger` | `#B42318` | Overdue, rejected, destructive actions |
| `info` | `#2F4BC0` | Pending, received |
Contrast target: WCAG AA (4.5:1 for text). Dark mode is not in v1.

### Typography
Inter (via `next/font`), system fallback. Base 16 px. Scale: 12, 14, 16, 18, 20, 24, 30. Headings semibold, body regular. Numbers in tables use tabular figures.

### Spacing, shape, motion
4 px grid (4, 8, 12, 16, 24, 32, 48). Radius 8 px for cards and inputs, 999 px for badges. One soft shadow level for dialogs only. Motion limited to 150 ms fades; respect reduced-motion settings.

### Breakpoints
Mobile under 640 px, tablet 640 to 1024 px, desktop over 1024 px.

## 4. Layout
- **Owner app:** desktop has a left sidebar (app switcher at top, then Dashboard, People, Charges or Requests, Documents, Settings). Mobile has a bottom bar with the same top four items and a menu.
- **Portal (tenant or client):** single column, no sidebar, big buttons. Shows only their own items, in order of what is due first.
- **App switcher:** one account can hold several apps; the current app name is always visible.
- **Content width:** max 1120 px on desktop.

## 5. Core components (build once, reuse everywhere)
| Component | Rules |
|---|---|
| Status badge | Color + icon + label. Values: Paid, Due soon, Overdue, Pending, Received, Approved, Rejected, Expiring, Expired. |
| Data table | On mobile it becomes a card list. Sortable by due date. Empty state with one clear action. |
| Form field | Visible label (no placeholder-only labels), helper text, inline error text, required marker. |
| Money input and display | Show `$1,234.56`. Store integer cents plus currency code. |
| Date | Show `Oct 22, 2026`. Use the app time zone. Overdue shows "3 days overdue". |
| Dashboard stat card | One big number, label, optional link to the filtered list. |
| Confirm dialog | Required for delete, reject, and reversing a ledger entry. Say what happens and whether it can be undone. |
| Toast | For saved and sent confirmations. Errors stay on screen until dismissed. |
| Empty state | Short sentence, one primary button, optional sample data. |
| Loading | Skeleton for lists. If the first request is slow, show "Waking up, one moment" (the free host sleeps) and retry automatically. |

## 6. v1 screens
1. Sign up, log in, verify email, reset password.
2. Template picker and a short first-run guide (add first unit or client).
3. Dashboard per app (widgets come from the App Definition, README 7.3).
4. People list and person detail (balance or checklist, history, send reminder now).
5. Record payment (manual) and reverse entry.
6. Document requests, upload, approve or reject with reason.
7. Reminder rules (edit the default timings) and reminder history with delivery status.
8. Settings: app name, time zone, email sender name, plan.
9. Portal: what I owe or must provide, upload, my history.

## 7. Content and tone
- Short sentences. Plain words. No legal threats. No ALL CAPS.
- Reminder email pattern: subject says what and when ("Rent of $1,200 is due Nov 1"); first line is the ask; one button; one line saying who it is from; no marketing.
- Escalation wording: friendly nudge (before due), clear notice (on due and 3 days after), firmer but polite (7 days after). Never mention fees or legal action unless the owner configured a late fee.
- Footer on every email: sent on behalf of the owner's business name, with the owner's reply-to address.
- Error messages say what happened and what to do next.

## 8. Accessibility checklist
- Everything works with a keyboard. Visible focus ring.
- Form fields have labels; errors are announced to screen readers.
- Touch targets at least 44 px.
- Color is never the only signal.
- Images and icons that carry meaning have text alternatives.

## 9. Branding
Wordmark "FollowWise" in Inter semibold, primary color. No logo yet. Template names in marketing: "FollowWise for Landlords", "FollowWise for Bookkeepers" (working names).
