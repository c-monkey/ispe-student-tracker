# Spreadsheet-native options: Google Forms + Sheets vs Microsoft Forms + Excel

Resolves [#3](https://github.com/c-monkey/ispe-student-tracker/issues/3) (child of map [#2](https://github.com/c-monkey/ispe-student-tracker/issues/2)). Researched 2026-11-20 against Google and Microsoft first-party docs. Vocabulary follows `GLOSSARY.md`.

## The question

What can Google Forms + Sheets and Microsoft Forms + Excel do on free tiers (personal accounts, and how school / Workspace / M365 accounts differ) for:

1. a per-Event Student Check-in via QR code / phone-friendly URL;
2. a Chapter Leader vetting Check-ins into Attendance;
3. restricting each Chapter Leader to their own Chapter's data, while the Coordinator sees everything;
4. computing per-Student Sponsorship status;
5. exporting status + history as an Excel workbook;

with quotas at ~10k Students and ~500 Events/year, and what free automation exists (Apps Script, Power Automate)?

## Scale assumptions used below

From map #2: ~100 Schools x ~100 Students = ~10k Students; ≤5 Chapter Events per Chapter per year plus ~10 ISPE Events, so roughly 500 Events/year. If each Student checks in to a handful of Events, that is on the order of 10k-50k Check-ins/year, all of which land in a responses table. Peak load is bursty: one Event's Check-ins arrive within minutes.

---

## Google Forms + Google Sheets

### Account tiers

- **Personal (gmail.com)**: free. Forms, Sheets, Drive, Apps Script all available.
- **Google Workspace** (paid, or a school's Workspace for Education): same tools, higher Apps Script quotas, plus domain-restricted forms.
- **Google Workspace for Nonprofits**: no-cost edition for eligible charities (US: IRS 501(c)(3), verified through Goodstack; schools, hospitals and government bodies excluded). Whether ISPE qualifies is not established here. Source: [Google Workspace for Nonprofits edition](https://knowledge.workspace.google.com/admin/getting-started/editions/google-workspace-for-nonprofits-edition), [US eligibility guidelines](https://support.google.com/nonprofits/answer/3215869?hl=en&co=GENIE.CountryCode%3DUS).

### 1. Check-in via QR / phone URL

- A form is shared by its responder link (optionally shortened) and can be opened to "anyone with the link". Source: [Publish & share your form with responders](https://support.google.com/docs/answer/2839588?hl=en).
- **Per-Event URLs**: a pre-filled link (More > Pre-fill form > Get link) lets one form carry a different Event already filled in per URL. Source: same page. Pre-filled values are ordinary editable answers, so a Student could change them; vetting has to catch that.
- **QR code**: Google's Forms help does not document a built-in QR generator (none found). The QR code has to be made from the URL with another tool (a browser's "create QR code" feature, or a QR library in Apps Script). Inference, not a documented Forms feature.
- **Identity**: "Collect email addresses" can be *Responder input* (typed, unverified) or *Verified* (requires a Google Account sign-in). "Limit to 1 response" also requires respondents to sign in. Sources: [View & manage form responses](https://support.google.com/docs/answer/139706), [Publish & share](https://support.google.com/docs/answer/2839588?hl=en). Most Students will not have, or want to use, a Google Account for this, so on a personal account the realistic option is typed identity (name/email/School), which is unverified, consistent with a Check-in being an unvetted claim.
- **Workspace difference**: forms in a work/school domain default to "Restrict to users in [domain]"; that setting must be turned off for outside Students. A school-domain account offers no advantage here, since Students span ~100 Schools.

### 2. Vetting Check-ins into Attendance

- No built-in approval workflow. Options: an "Approved" checkbox column next to responses, or an Apps Script that copies vetted rows into an Attendance sheet. Form responses land in a linked Sheet.
- Apps Script installable triggers (e.g. on form submit) are free on personal accounts. Quotas, [Apps Script quotas](https://developers.google.com/apps-script/guides/services/quotas):

| Quota | Consumer (gmail.com) | Workspace |
|---|---|---|
| Script runtime | 6 min / execution | 6 min / execution |
| Triggers total runtime | 90 min / day | 6 hr / day |
| Triggers | 20 / user / script | 20 / user / script |
| Email recipients | 100 / day | 1,500 / day |
| URL Fetch calls | 20,000 / day | 100,000 / day |
| Simultaneous executions | 30 / user; 1,000 / script | same |
| Custom function runtime | 30 s / execution | 30 s / execution |

  At ~50k Check-ins/year a sub-second on-submit trigger uses far less than 90 min/day; the 30-simultaneous-executions-per-user cap is the one to watch during a burst of Check-ins at a large ISPE Event (inference from the numbers; untested).

### 3. Per-Chapter data restriction

- **Sharing is per file.** Protected ranges/sheets only restrict *editing*: protection "shouldn't be used as a security measure"; "People can print, copy, paste, and import and export copies of a protected spreadsheet", and viewers can make a copy. Source: [Protect, hide, and edit sheets](https://support.google.com/docs/answer/1218656). Hidden tabs are likewise not security.
- So isolating each Chapter means **one spreadsheet per Chapter** (potentially a few hundred files), each shared only with its Chapter Leader, with a Coordinator master file pulling them together (IMPORTRANGE or script). An Apps Script would have to create these files, route Check-ins into them, and re-share them when Chapter Leaders turn over. This is the main maintenance burden.
- **Apps Script web app alternative**: a web app can "execute as me" (the owner) and serve each Chapter Leader only their rows. Source: [Web apps](https://developers.google.com/apps-script/guides/web). But the app needs to know who is viewing: `Session.getActiveUser()` returns a blank email in a web app deployed to "execute as me", unless the viewer is the developer or in the same Workspace domain. Source: [Session reference](https://developers.google.com/apps-script/reference/base/session). On a personal account, identifying Chapter Leaders therefore needs either "execute as user accessing" (each Chapter Leader signs in with a Google Account and authorizes the script, and must then be given access to the underlying data) or a home-made login (secret per-Chapter links/tokens). At that point it is a small custom web app hosted on Apps Script, not "spreadsheet-native".

### 4. Sponsorship status

- Computable with ordinary formulas (COUNTIFS over Attendance by Student and year) or Apps Script. Custom functions are capped at 30 s each, so heavy logic is better done in a triggered script that writes values.
- Size limit: "Up to 20 million cells or 100MB" per spreadsheet. Source: [Files you can store in Google Drive](https://support.google.com/docs/answer/37603). At ~10 columns, that is ~2M rows, decades of headroom at this scale.

### 5. Excel export

- Sheets downloads natively as .xlsx (File > Download > Microsoft Excel), and Apps Script can produce an .xlsx export of a generated "status + history" spreadsheet. Formulas without Excel equivalents and Apps Script logic do not carry over (well-known behaviour; not separately sourced).

---

## Microsoft Forms + Excel

### Account tiers

- **Personal Microsoft account (Outlook.com/Hotmail/Live)**: free Forms and Excel for the web, OneDrive personal storage. Paid Microsoft 365 Personal/Family raises some limits.
- **Work/school (Microsoft 365 Business, Education)**: full Forms, OneDrive for Business, Office Scripts, and Power Automate.
- **Microsoft nonprofit grant**: primary cloud grant is Microsoft 365 Business Basic for eligible nonprofits; licences only for paid staff and unpaid executive leadership, with an 85% active-usage requirement. Source: [Microsoft nonprofit eligibility](https://www.microsoft.com/en-us/nonprofits/eligibility). Chapter Leaders (students/faculty at other institutions) would not be grant-eligible users.

### 1. Check-in via QR / phone URL

- Forms has a built-in QR code: in the sharing dialog, "Select the QR code button, and then ... Download", giving a .png. Source: [Send a form and collect responses](https://support.microsoft.com/en-us/office/send-a-form-and-collect-responses-2eaf3294-0cff-492d-884d-a1dee909e845).
- "Anyone can respond" lets people inside or outside the organization respond without signing in; responses come in without names. "Record name" and "One response per person" are only available when responses are limited to the organization, and "Share with people in my org" is "disabled for consumers". Source: [Choose who can fill out a form or quiz](https://support.microsoft.com/en-us/forms/choose-who-can-fill-out-a-form-or-quiz). Since Students come from many Schools, Check-in forms would be "Anyone can respond" with typed identity under any account type.
- **Pre-filled links** exist (announced May 2024; "Get Pre-filled link"), so one form can serve many Events. Prefilled fields cannot be made read-only or hidden. Source: [Pre-fill Responses in Your Microsoft Forms (Microsoft Forms blog)](https://techcommunity.microsoft.com/blog/microsoftformsblog/pre-fill-responses-in-your-microsoft-forms/4144232).

### Response limits: the deciding constraint on personal accounts

Source: [Form, question, response, and character limits in Microsoft Forms](https://support.microsoft.com/en-us/forms/form-question-response-and-character-limits-in-microsoft-forms).

| Limit | Personal free | Personal paid | Work/school |
|---|---|---|---|
| Forms per account (incl. recycle bin) | 400 | 400 | 400 |
| Questions per form | 200 | 200 | 200 |
| Responses per form | **200** | **1,000** | up to 5,000,000 |

- On a free personal account a single shared Check-in form fills up after 200 Check-ins; even one form per Event would exceed 400 forms in a year at ~500 Events. Staying under the caps means exporting and deleting responses regularly (a manual, error-prone chore).
- On work/school accounts the caps are not a concern at this scale.

### 2. Vetting Check-ins into Attendance

- **Live sync to Excel is a work/school feature**: "Open results in Excel" stores the workbook "in OneDrive for work or school" with "a live data connection to your form". Source: [Check and share your form results](https://support.microsoft.com/en-us/office/check-and-share-your-form-results-02859424-341d-406f-b32a-9a0fbaf357af). For personal accounts no live connection is documented; you get a downloaded snapshot.
- **Power Automate cloud flows require a work or school account**: "Before you can create a cloud flow, sign up by using a Microsoft work or school email address." Source: [Sign up and sign in - Power Automate](https://learn.microsoft.com/en-us/power-automate/sign-up-sign-in). Community answers on Microsoft Q&A report personal accounts lost cloud-flow access in mid-2025 (not first-party policy text). There is a free plan, but only for work/school identities.
- **Office Scripts**: business/education licences need OneDrive for Business; personal/family M365 subscriptions are in *preview* via the Microsoft 365 Insider program; running Office Scripts from Power Automate requires a business licence (1,600 Run-script calls/user/day, 120 s timeout, 5 MB request/response). Source: [Office Scripts platform limits](https://learn.microsoft.com/en-us/office/dev/scripts/testing/platform-limits). A free personal account (no M365 subscription) has no documented Office Scripts access.
- Net: on a free personal Microsoft account, vetting is manual (copy rows from a downloaded snapshot into an Attendance workbook and mark them), with no free automation. On work/school, Forms → live Excel + Power Automate can append Check-ins to per-Chapter tables.

### 3. Per-Chapter data restriction

- Like Google, OneDrive/SharePoint sharing is per file/folder; Excel has no per-sheet read permission (sheet protection limits editing, not viewing). So per-Chapter isolation again means one workbook per Chapter plus a Coordinator master. On work/school, Power Automate could route rows into per-Chapter workbooks; on personal accounts this would all be manual. (Inference from the sharing model; Microsoft's docs do not offer row-level permissions in Excel.)
- Sharing Chapter workbooks with Chapter Leaders from a work/school tenant may be blocked or limited by the tenant's external-sharing policy (a school tenant is usually locked down; out of Lisa's control).

### 4. Sponsorship status

- COUNTIFS-style formulas work in Excel for the web on any account. Excel's grid is 1,048,576 rows per sheet. Source: [Excel specifications and limits](https://support.microsoft.com/office/1672b34d-7043-467e-8e27-269d656771c3). Volume is not a concern.

### 5. Excel export

- Native: the data already is an .xlsx workbook.

---

## Comparison

| Need | Google, personal (free) | Google, Workspace | Microsoft, personal (free) | Microsoft, work/school |
|---|---|---|---|---|
| Check-in URL, phone-friendly, no Student login | Yes (typed identity) | Yes, if domain restriction turned off | Yes, "Anyone can respond" | Yes, "Anyone can respond" |
| Built-in QR code | No (generate externally) | No | Yes | Yes |
| Per-Event link from one form | Pre-filled link | Pre-filled link | Pre-filled link | Pre-filled link |
| Response volume at ~50k/yr | Fine (20M cells per sheet) | Fine | **Blocked: 200 responses/form, 400 forms** | Fine (5M/form) |
| Live responses → spreadsheet | Yes | Yes | No (snapshot only, as documented) | Yes |
| Free automation | Apps Script (90 min/day triggers) | Apps Script (6 h/day) | **None** (no Power Automate cloud flows; Office Scripts only via paid M365 preview) | Power Automate + Office Scripts |
| Per-Chapter isolation | One file per Chapter, or Apps Script web app with its own login scheme | Same; web app can see identities in-domain only | One workbook per Chapter, manual | One workbook per Chapter, automatable |
| Sponsorship status | Formulas / script | Formulas / script | Formulas | Formulas / scripts |
| Excel export | .xlsx download (Apps Script can automate) | Same | Native | Native |
| Unbounded bill risk | None | Per-seat subscription, fixed | None | Per-seat subscription, fixed |

## What this means for the platform choice

For ticket [#9](https://github.com/c-monkey/ispe-student-tracker/issues/9) to weigh; no decision made here.

- **Microsoft on a free personal account is effectively ruled out by its own limits**: 200 responses per form, no live Excel sync, and no free automation. Microsoft is viable only with a work/school tenant (an ISPE M365 tenant, a nonprofit grant, or paid Business licences), which conflicts with "no existing cloud accounts" unless ISPE already has one.
- **Google on a free personal account meets the volume and automation needs**: Forms + Sheets + Apps Script at no cost and with no billing surface. Its weak point is per-Chapter isolation, which spreadsheet sharing cannot do inside one file.
- **Per-Chapter isolation is the hard requirement neither suite does natively.** Both need one file per Chapter (a few hundred files to create, share, and re-share at each yearly Chapter Leader handover) or, on Google, an Apps Script web app. The web app route on a personal account must solve identity itself, since `getActiveUser()` is blank for "execute as me" apps outside a Workspace domain. That moves it toward a custom web app, which overlaps with the other platform tickets.
- **Check-in identity is unverified on every option** (Students come from ~100 different Schools, so domain sign-in is out). That fits the glossary (a Check-in is an unvetted claim), but the form should collect a stable Student key (e.g. email) so Attendance can be matched across Events.
- Things to confirm by hands-on test before relying on them: whether a personal-account Microsoft form gets any live Excel connection; Apps Script behaviour under a burst of ~100+ simultaneous Check-ins; and whether ISPE already has, or qualifies for, a Workspace or M365 nonprofit tenant.
