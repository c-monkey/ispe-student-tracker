# Low-cost web options with hard spending caps

Research for [#4](https://github.com/c-monkey/ispe-student-tracker/issues/4) (map [#2](https://github.com/c-monkey/ispe-student-tracker/issues/2)). Sources were checked on 2026-11-20. Pricing pages change often, so re-check them before committing to a platform.

## The question

Which low-cost hosts can a newcomer with no cloud account use for the tracker? Which of them offer a real **hard spending cap**, or a free tier that cannot bill above zero? The owner pays for this personally, so abuse or a DoS attack must never be able to produce an unbounded bill.

## Our scale, for comparison

- About 10k Students: ~100 Schools × ~100 Students.
- About 500 Events per year: ~100 Schools × ≤5 Chapter Events, plus ~10 ISPE Events.
- Check-ins arrive in bursts of about 100 per Event, which is about 50k Check-ins per year. On a busy evening that could be a few hundred writes within minutes.
- Read traffic is light: Chapter Leaders vetting Check-ins and the Coordinator exporting to Excel.

Every option below handles this load comfortably within its free tier. The deciding factors are what happens when a limit is hit, whether a payment card is ever attached, and how much auth work the option needs.

## Three kinds of "safety"

These are easy to confuse, so this report keeps them separate:

1. **No billing relationship at all.** No card is on file, and going over a limit errors or pauses the service. A bill is impossible. The worst case is an outage.
2. **A hard cap on a paid account.** A card is on file, but the provider stops the service rather than charge beyond the cap. These caps usually exclude some line items, so read the exclusions.
3. **Budget alerts.** You get an email, but usage and charges continue. **This is not protection.**

---

## Google Apps Script web app (with a Google Sheet as the data store)

- **Cost model:** The quotas page describes quotas only. It has no per-use billing for consumer (gmail.com) accounts. "If you exceed a quota or limitation, your script throws an exception and execution stops." Quotas are per user and reset 24 hours after the first request. Source: https://developers.google.com/apps-script/guides/services/quotas
  - **Cap type: (1) No billing relationship.** There is no card and no way to be charged.
- **Limits relevant to us** (consumer accounts; same source): 6 minutes per execution, 30 simultaneous executions per user and 1,000 per script, 20,000 URL Fetch calls per day, 50,000 Properties reads/writes per day.
  - In an "execute as me" web app, every Student's request runs as the owner, so all traffic counts against the owner's quotas.
  - A burst of ~100 Check-ins is well within these limits. Concurrent appends to one Sheet need `LockService` to serialize writes. This is a design note, not a limit.
- **Data store:** Google Sheets allows "Up to 20 million cells or 100MB" per spreadsheet (https://support.google.com/drive/answer/37603). 50k Check-ins × ~10 columns is about 500k cells per year, so years of history fit.
- **Auth for Chapter Leaders:** Web apps either execute as the owner or "under the identity of the active user using the web app" (https://developers.google.com/apps-script/guides/web).
  - `Session.getActiveUser().getEmail()` "returns a blank string" when the web app is deployed to execute as the developer, unless the user is in the same Workspace domain (https://developers.google.com/apps-script/reference/base/session#getActiveUser()).
  - So the Student form can run as "me" and accept anyone. Chapter Leader views need either a second deployment that runs as the accessing user (so their Google sign-in identifies them), or Sheet-level sharing.
  - Every Chapter Leader would need a Google account.
  - Google also limits how quickly new users can authorize a user-executed web app, and the documented limits depend on whether the publisher is in a Workspace domain (same web-app guide).
- **Abuse exposure:** No cost. Spam could exhaust the daily quota or fill the Sheet with junk Check-ins, which is an availability and data-quality risk only. Vetting by Chapter Leaders already filters junk.
- **Excel export:** Native, because the data already lives in a Sheet and Sheets can download it as .xlsx. A script can also build a dedicated export Sheet on demand.

## Cloudflare Pages / Workers + D1

- **Cost model:** "By default, users have access to the Workers Free plan." Paid usage exists only on the opt-in Workers Paid plan ($5 monthly minimum). Source: https://developers.cloudflare.com/workers/platform/pricing/
  - **Cap type: (1) No billing relationship,** as long as you stay on Free.
  - The pricing pages do not say whether signing up needs a card. Our understanding is that it does not, but this was not verified from a primary source.
- **Over-limit behaviour:**
  - Workers: over the daily request limit, "Cloudflare returns **Error 1027**". Routes can fail open or fail closed, and the limit resets at midnight UTC (https://developers.cloudflare.com/workers/platform/limits/).
  - D1: "When your account hits the daily read and/or write limits, you will not be able to run queries against D1" and the API "will return errors" (https://developers.cloudflare.com/d1/platform/pricing/).
  - Free limits error out; they never bill.
- **Free limits vs our scale:**
  - Workers: 100,000 requests per day and 10 ms CPU per invocation.
  - D1: 5M rows read per day, 100k rows written per day, 5 GB storage, 10 databases.
  - Static assets: "Requests to static assets are free and unlimited." Pages Functions count as Workers. (Sources: the two pricing pages above.)
  - Rows read count scanned rows, so indexes matter: "Defining indexes on your table(s) reduces the number of rows read". Our volumes are orders of magnitude below these limits.
- **Auth for Chapter Leaders:** Neither Workers nor D1 includes user auth, so the options are:
  - Cloudflare Access (Zero Trust). Cloudflare markets the Zero Trust free plan for "teams under 50 users" (https://www.cloudflare.com/plans/sase-zero-trust.md). With ~100 Schools and possibly several Chapters each, the Chapter Leaders could exceed 50 seats. Whether the Zero Trust free plan needs a payment method was **not confirmed**.
  - Our own login built in a Worker, for example per-Chapter passcodes or magic links. Magic links need an email-sending service, which is another account and another possible cost.
- **Abuse exposure:** No cost on Free. A flood could use up the 100k-request daily quota and take the app down until midnight UTC. Turnstile, Cloudflare's CAPTCHA, has a free plan with "Up to 20 widgets" and can protect the Check-in form (https://developers.cloudflare.com/turnstile/plans/).
- **Excel export:** No native export. Options are generating .xlsx in the browser with a JS library such as SheetJS (https://docs.sheetjs.com/) or in a Worker. This is easy but has to be built.

## Supabase (Postgres + Auth)

- **Cost model:** "you will not be charged while using the Free Plan" (https://supabase.com/docs/guides/platform/cost-control).
  - The **Spend Cap** "is available only with the Pro Plan". When it is on, usage over quota is blocked instead of billed.
  - The Spend Cap excludes Compute, Custom Domain, IPv4, PITR, MFA Phone, extra Disk IOPS/throughput, and Log Drains. Pro's compute charge is therefore always billed.
  - **Cap type: Free = (1). Pro = (2), with exclusions.**
  - Neither the pricing page nor the billing docs say whether the Free plan needs a card.
- **Free limits:** "500 MB database size per project", "50,000 monthly active users", "5 GB egress", "500,000" Edge Function invocations, and "Limit of 2 active projects" (https://supabase.com/pricing). Our data is a few MB per year, well within these.
- **Big caveat:** "Free projects are paused after 1 week of inactivity" (same page). Our traffic is event-driven and seasonal, with likely gaps of more than a week between Events and over summer. A paused project means a Student's Check-in fails until someone unpauses it, so this needs a keep-alive or manual unpausing. It is a real operational burden for Lisa.
- **Auth for Chapter Leaders:** Built-in Supabase Auth (email/password, magic link, OAuth). Postgres Row Level Security suits "Chapter Leaders see only their Chapter". 50k MAU is far above our need.
- **Abuse exposure:** On Free there is no bill. Over-quota behaviour on Free is not described in the docs we found (the billing docs only say "Only usage beyond the quota is billed as overage"). The Fair Use Policy allows restricting a project that keeps going over quota. The worst case looks like an outage or restriction, not a bill.
- **Excel export:** Client-side .xlsx generation (SheetJS) from a query, or CSV export from the dashboard. It has to be built.

## Firebase (Hosting + Firestore + Auth) on the Spark plan

- **Cost model:** Spark: "No payment method needed" (https://firebase.google.com/pricing). "You don't need to provide any payment information"; Spark projects have no linked Cloud Billing account, and linking one upgrades the project to Blaze. Over the no-cost quota, "your project's usage of that specific product will be shut off for the remainder of that month" (https://firebase.google.com/docs/projects/billing/firebase-pricing-plans).
  - **Cap type: (1) No billing relationship.**
  - Blaze is pay-as-you-go. Its "spend cap budgets" exist only for AI Logic, App Hosting, Cloud Functions and Extensions, **not Firestore or Hosting**. Its other budgets are alerts-only: "Budget alerts do not pause services." (https://firebase.google.com/docs/projects/billing/avoid-surprise-bills). **Blaze is therefore not safe for Firestore.**
- **Spark limits vs our scale** (pricing page):
  - Firestore: 1 GiB stored, 50K reads/day, 20K writes/day, 20K deletes/day, 10 GiB/month egress.
  - Hosting: 10 GB storage and 360 MB/day transfer.
  - Auth: 50K MAU.
  - Cloud Functions are "Not applicable" on Spark, so all logic must live in the client plus Firestore Security Rules.
  - 50K reads/day is ample for vetting screens if queries are scoped per Chapter. A Coordinator export of all history (~50k docs per year) could use a whole day's read quota on its own after a year or two. This needs care, for example a pre-aggregated export document.
- **Auth for Chapter Leaders:** Firebase Auth (email link, Google, etc.) plus Security Rules can scope reads to a Chapter.
- **Abuse exposure:** No cost on Spark. Spam writes could exhaust the 20K daily writes, and over-quota usage is "shut off for the remainder of that month" per the plans page. For Firestore the pricing page says no-cost limits are enforced daily. Either way the outcome is an outage, not a bill. App Check or a CAPTCHA would reduce the risk.
- **Excel export:** Client-side .xlsx generation (SheetJS). It has to be built, and it uses read quota.

## Azure Static Web Apps

- **Hosting cost model:** On the Free plan, overage bandwidth is "Unavailable" (100 GB/month included), with 10 apps and 500 MB storage (https://learn.microsoft.com/en-us/azure/static-web-apps/quotas). The hosting itself therefore cannot bill beyond zero.
- **The catch is the subscription:**
  - Static Web Apps needs an Azure subscription, and Static Web Apps has no data store, so a database (Cosmos DB, Azure SQL, etc.) would sit in the same subscription.
  - Azure's spending limit applies to the free account and credit-based subscriptions. It "isn't available for subscriptions with commitment plans or with pay-as-you-go pricing" (https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/spending-limit).
  - Upgrading a free account to pay-as-you-go removes the limit (same page). Even with a spending limit, some Marketplace services "can incur separate charges".
  - **Cap type: hosting = (1)-like. The surrounding subscription = (3) once on pay-as-you-go.**
  - Signing up for an Azure account involves a payment method. This is from general knowledge and was not re-verified here.
- **Auth for Chapter Leaders:** The Free plan has only "Preconfigured" providers (Entra ID, GitHub). Custom roles work "via invitations" only, with a limit of 25 (https://learn.microsoft.com/en-us/azure/static-web-apps/plans, quotas page). That is too few for ~100+ Chapter Leaders. Roles assigned through functions need Standard.
- **Abuse exposure:** Hosting is capped. Managed Functions and any database are where exposure comes back once the subscription is pay-as-you-go.
- **Excel export:** It has to be built (client-side or in a Function).

## Netlify (Free, credit-based)

- **Cost model:** "The Free plan includes 300 credits/month with a hard limit and no auto recharge option." "You'll never be charged for the Free plan." (https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/credit-based-pricing-plans/).
  - When credits run out, "all projects owned by that team enter a paused state until the start of the next billing cycle". Paused projects receive no "web requests; form submissions". "The free plan does not allow you to purchase more credits" (https://docs.netlify.com/manage/accounts-and-billing/billing/resume-paused-projects/).
  - **Cap type: (1).** Paid plans add opt-in credit packs and auto-recharge.
- **Credit costs:** 2 credits per 10,000 web requests, 20 credits per GB of bandwidth, 10 credits per GB-hour of Functions compute (credit-based plans page).
  - Normal use is tiny.
  - A request flood is the risk: 300 credits is about 1.5M requests, or 15 GB of bandwidth. Used up mid-month, **the whole team stays paused until the next cycle**, possibly weeks with no Check-ins.
- **Data store / auth:** The Free plan lists "Netlify Database" and "Blob storage" (https://www.netlify.com/pricing/). We did not research their free quotas or auth options (Identity is not mentioned on the pricing page). Auth would likely be built in our own code or come from a third party.
- **Excel export:** It has to be built.

---

## Comparison

| Option | Card needed? | Cap type | Over-limit outcome | Fits our scale on free? | Chapter Leader auth | Abuse worst case | Excel export |
|---|---|---|---|---|---|---|---|
| Apps Script + Sheets | No | (1) no billing | Exception; resets in 24h | Yes, easily | Google accounts; needs a user-executed deployment or Sheet sharing | Quota exhausted for a day; junk rows | **Native** (.xlsx download) |
| Cloudflare Workers + D1 | Not confirmed (believed no) | (1) on Free | Error 1027 / D1 errors; resets 00:00 UTC | Yes, easily | Build our own, or Access (free tier marketed for <50 users) | Down until midnight UTC | Build (SheetJS) |
| Supabase Free | Not confirmed | (1) on Free; Pro cap = (2) with exclusions | Restriction under Fair Use (not fully documented) | Yes, but **pauses after 1 week idle** | Built-in Auth + RLS | Restriction / outage | Build |
| Firebase Spark | No | (1) no billing | Product shut off (daily for Firestore) | Yes; watch the 50K reads/day on exports | Firebase Auth + Rules | Writes exhausted; outage | Build; uses read quota |
| Firebase Blaze | Yes | (3) alerts only for Firestore/Hosting | Keeps billing | n/a | — | **Unbounded bill** | — |
| Azure SWA Free | Yes (Azure account; not re-verified) | Hosting (1); subscription (3) on pay-as-you-go | Hosting: no overage. DB: billed | Hosting yes; DB is separate | Preconfigured providers, ≤25 invited roles | Bill from DB/Functions on pay-as-you-go | Build |
| Netlify Free | Not confirmed | (1) hard credit limit | **Whole team paused until next month** | Yes normally | Build or third party | Weeks of downtime | Build |

## What this means for the platform choice

The decision belongs to [#9](https://github.com/c-monkey/ispe-student-tracker/issues/9); these are the inputs.

- **Options that truly cannot bill:** Apps Script + Sheets, Firebase **Spark**, and Cloudflare Workers/D1 on **Free**. In each, no billing relationship exists and over-limit usage errors out. Netlify Free and Supabase Free also say they never charge. Supabase's Pro Spend Cap and Azure's spending limit are partial caps with exclusions. Firebase Blaze budgets and Azure pay-as-you-go budgets are alerts, **not** caps.
- **The real risk is availability, not cost.** On every type-(1) option, abuse leads to an outage. How long it lasts varies a lot: about a day on Apps Script, Cloudflare and Firestore, but up to a month for Netlify's team-wide pause. Any Firebase Spark product that hits the monthly shut-off rule is also out for the month. A CAPTCHA (Turnstile, App Check) and the existing vetting step reduce the risk.
- **Upgrading is a one-way risk.** On most of these, moving to a paid plan (Blaze, Workers Paid, Supabase Pro, Azure pay-as-you-go) changes the cap type. A "stay on free tier" rule written into the runbook matters for Lisa's later maintenance with Claude.
- **Excel export and Lisa running it alone favour Apps Script + Sheets.** The data is already a spreadsheet, export is native, and Lisa can inspect or fix records directly. The trade-offs are a basic UI and Google-account-based auth for Chapter Leaders.
- **Auth is the main build cost elsewhere.** Supabase and Firebase have the best built-in auth with per-Chapter row/rule scoping. Cloudflare needs auth built from scratch, or Access with a 50-user ceiling. Azure Free's 25 invited roles is too few.
- **Supabase's 1-week pause** conflicts with seasonal, event-driven use unless it is kept alive.
- **Open items worth checking during #9:**
  - Card requirements for the Cloudflare and Supabase sign-ups.
  - Payment-method requirement and seat count for Cloudflare Zero Trust Free.
  - Netlify Database free quotas.
  - Google's actual authorization-rate limits for a consumer-account user-executed web app with ~100+ Chapter Leaders.
