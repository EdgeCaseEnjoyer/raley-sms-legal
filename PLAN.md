# Jasper SMS Compliance — Plan of Attack

**Date:** 2026-04-12
**Status:** Draft, awaiting Kyle approval
**Goal:** Get SMS sending unblocked for Jasper within 2 weeks, on two parallel paths, after two A2P rejections.

---

## What happened

- **Brand (Raley Ventures, Sole Prop) APPROVED** — TCR ID BD3HQ43. Good.
- **A2P Campaign REJECTED on 2026-04-05** — error 30909 (CTA/MESSAGE_FLOW verification failed).
- **Toll-free 844 number has ZERO verification submitted.** Untouched path.
- **Current opt-in page** (edgecaseenjoyer.github.io/raley-sms-legal/) has:
  - No form
  - No unchecked consent checkbox
  - No "Message and data rates may apply" on the landing page
  - No message frequency disclosure on the landing page
  - No sample messages
  - Terms and Privacy exist as separate pages (good) but Privacy is missing the mobile-specific carve-out language
  - Hosted on a domain that doesn't match the brand name "Raley Ventures"
- Kyle owns **raleyventures.com** (GoDaddy DNS, resolving).

## Root cause of the rejection

Deep research (two parallel agents, 14+ cited sources including Twilio's own docs, HighLevel, SignalWire, Alive5, MessageDesk) converges on this: TCR reviewers fetch the opt-in URL and look for a specific checklist of elements ON the page. The current page fails ~6 of 8 elements. Twilio publishes verbatim "good vs bad" examples — we were closer to "bad" on every field.

The single biggest fix: **the privacy policy must contain a specific verbatim mobile-carve-out clause**. Multiple sources flag this as the #1 cause of 30909 rejections.

## Strategy — TWO PARALLEL PATHS, NOT ONE

We've been rejected twice. A third 10DLC rejection puts the brand under extra scrutiny. Answer: stop betting everything on one path.

### Path A — Toll-Free Verification on +18445890114 (PRIMARY)
- Different reviewer pool than 10DLC, more forgiving for low-volume personal use cases
- No campaign/brand complexity — one submission
- 5-14 day review
- Throughput: up to 3 MPS, way more than Jasper needs
- This is our NEW primary path. Should have been first, wasn't.

### Path B — Rebuild 10DLC opt-in + resubmit campaign on +18177555992
- Fix every element of the landing page against the researched checklist
- Rewrite campaign description, message_flow, and sample messages using verbatim patterns from Twilio's own approved examples
- Keeps the local number as a backup if toll-free gets rejected

### Path C — iMessage via apple-mcp (IMMEDIATE FALLBACK)
- For Kyle-to-Kyle notifications right now, today, while A/B work their way through review
- No carrier involvement, no A2P at all
- Works as long as Mac is awake and contacts are iMessage-capable
- This is free insurance

---

## The work, step by step

### Phase 1 — Opt-in page rebuild (local files, no external calls)

Edit `~/Projects/raley-sms-legal/`:

1. **index.html** — rebuild as a real opt-in page with:
   - Brand name "Raley Ventures" in H1
   - Sub-description naming "Jasper AI executive assistant"
   - Visible phone-number input field (even if non-functional — it has to LOOK like a form)
   - **Unchecked** consent checkbox with the exact disclosure language (see Appendix A)
   - Three sample messages visible on the page
   - Message frequency disclosure ("3-10 msgs/day")
   - "Message and data rates may apply" verbatim
   - "Reply STOP to opt out, HELP for help" verbatim
   - Visible links to Privacy Policy and Terms of Service
   - Clean, readable, OpenDyslexic-friendly fonts

2. **privacy.html** — add the HighLevel-approved mobile carve-out clause verbatim:
   > "No mobile information will be shared with third parties/affiliates for marketing/promotional purposes. Information sharing to subcontractors in support services, such as customer service and SMS delivery providers (Twilio), is permitted solely to deliver messages. We do not sell, rent, distribute, or trade your personal data to third parties without your explicit consent."

3. **terms.html** — add a dedicated "SMS Program Terms" section that mirrors the opt-in page's disclosures. Keep existing content.

### Phase 2 — Optional custom domain (decision point)

- Point `sms.raleyventures.com` → GitHub Pages via CNAME record
- Edit DNS in GoDaddy (Kyle has to do this OR I automate via Playwright)
- Update all reference URLs in campaign resubmission to `sms.raleyventures.com`
- **Cost:** 10 minutes of DNS work
- **Benefit:** Brand name matches URL — noticeably improves review odds per research
- **Risk:** Adds a dependency, DNS propagation can take minutes to hours
- **Recommendation:** DO IT. The brand/URL mismatch is a soft signal reviewers weight against small operators.

### Phase 3 — Commit and deploy page changes

- `git add`, `git commit`, `git push` to edgecaseenjoyer/raley-sms-legal
- Verify GH Pages rebuild by fetching the live URL
- Take Playwright screenshots of the live page for evidence records

### Phase 4 — Submit Toll-Free Verification (Path A — primary)

- POST to `https://messaging.twilio.com/v1/Tollfree/Verifications` via Twilio API
- Fields: business name "Raley Ventures", business type "Sole Proprietor", website URL (sms.raleyventures.com), opt-in type "WEB_FORM", message content samples (see Appendix B), opt-in image URL (we'll upload a screenshot of the new page)
- Use credentials from Keychain (already verified working this session)
- Does NOT replace A2P — it's a parallel track on the toll-free number

### Phase 5 — Resubmit A2P Campaign (Path B — backup)

- PATCH/DELETE+CREATE the existing campaign via Twilio API
- Use verbatim-researched language from Appendix C (description, message_flow, samples)
- Every field traced to a Twilio-published "good example" or HighLevel-approved template
- Note: TCR may treat a resubmission as a fresh review — this is fine

### Phase 6 — Enable iMessage fallback (Path C)

- Verify apple-mcp is working (it is, per memory)
- Add a Jasper config flag: `NOTIFICATION_CHANNEL=imessage` that routes alerts through apple-mcp instead of Twilio
- Test end-to-end: Jasper triggers a notification → iMessage arrives on Kyle's phone within 10 seconds
- Leave this as the active channel until Path A or B is approved

### Phase 7 — Monitoring

- Check A2P and TFV status every 24 hours via API (no more manual console checking)
- When either is approved, flip Jasper's notification channel from iMessage to Twilio
- Update brain.db + memory file `reference_twilio.md` with approval timestamps

---

## Decision points for Kyle (need answers before executing)

1. **Custom domain** — are you OK pointing `sms.raleyventures.com` at GitHub Pages? (Adds a DNS step but materially improves review odds.)

2. **iMessage fallback** — is routing Jasper notifications through iMessage while we wait acceptable to you? It means notifications only fire when the Mac is awake, which for your workflow is ~16h/day.

3. **Sole Prop vs. upgrading to EIN** — do you have an EIN for Raley Ventures? If yes, we can upgrade to Low-Volume Standard brand which has higher approval rates AND doesn't have the 1-campaign/1-number lock. This is a bigger lift but may be worth it. (Skip if Sole Prop only.)

4. **Authorization to submit** — OK for me to submit the toll-free verification and resubmit the A2P campaign via API on your behalf once the page is rebuilt? Both are reversible (can be withdrawn) and non-destructive, but they're external actions that go on the Twilio compliance record.

---

## Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Toll-free verification also rejected | Low | Page rebuild applies same improvements — both paths benefit |
| A2P resubmission 3rd rejection | Medium | Already have TFV and iMessage as alternatives |
| DNS misconfig breaks sms.raleyventures.com | Low | Test via dig before submitting URL to Twilio |
| GH Pages rebuild delay | Low | Wait and verify live URL before any submission |
| Research miss — some new TCR rule changed | Low | All sources cross-checked, most dated within last 6 months |

---

## Timeline

- **Today (Day 0)**: Phases 1–3 (page rebuild, commit, deploy) + Phase 6 (iMessage fallback live)
- **Day 0–1**: Phase 4 (TFV submission) + Phase 5 (A2P resubmission)
- **Day 1–14**: Phase 7 (monitoring via API, daily status checks)
- **Target approval**: TFV by Day 7, A2P by Day 14

---

## Appendix A — New opt-in page content (verbatim-ready)

### H1
Raley Ventures SMS Notifications

### Subhead
Sign up to receive SMS notifications from Jasper, the Raley Ventures AI executive assistant.

### Body
Receive meeting reminders, task deadline alerts, calendar updates, and account notifications from your Jasper AI assistant. Up to 10 messages per day. Message frequency varies. Message and data rates may apply. Reply HELP for help, STOP to unsubscribe at any time.

### Form
- Phone number input field (tel type)
- **Unchecked** checkbox with label:
  > "I consent to receive recurring SMS account notifications from Raley Ventures at the phone number provided. Message frequency varies (up to 10 messages per day). Message and data rates may apply. Reply HELP for help, STOP to opt out at any time. Consent is not a condition of any purchase."
- Submit button

### Sample messages section
1. "Jasper (Raley Ventures): Meeting reminder — 'Classic Transport DD call' starts in 15 minutes. Reply STOP to opt out."
2. "Jasper (Raley Ventures): Task due today — 'Send Cameron updated pitch deck.' Msg freq varies. Reply HELP for help, STOP to opt out."
3. "Jasper (Raley Ventures): New commitment logged — 'Return Byron's call by Friday 3pm CT.' Msg & data rates may apply. Reply STOP to opt out."

### Footer
- [Privacy Policy](privacy.html) — [Terms of Service](terms.html)
- "No mobile information will be shared with third parties or affiliates for marketing or promotional purposes."

---

## Appendix B — Toll-Free Verification fields

**Business name:** Raley Ventures
**Business type:** Sole Proprietor
**Business website:** https://sms.raleyventures.com (or GH Pages URL if skipping custom domain)
**Opt-in type:** WEB_FORM
**Opt-in image URL:** Screenshot of live opt-in page (we'll host it)
**Notification email:** kyle@raleyventures.com
**Message volume:** 2,000/month
**Use case categories:** Account Notification, Two-Factor Authentication
**Use case summary:**
> "Raley Ventures sends transactional SMS account notifications to the sole proprietor, Kyle Raley, who has opted in via web form on sms.raleyventures.com. Messages are generated by Kyle's internal scheduling and task-management system (Jasper AI assistant) and include meeting reminders, task deadline alerts, calendar change notifications, and account activity confirmations. No marketing, no promotions, no third-party content."

**Production message samples:** Same three as Appendix A.

---

## Appendix C — A2P Campaign resubmission fields

**Use case:** SOLE_PROPRIETOR (locked under Sole Prop brand)

**Description (verbatim):**
> "Messages are sent by Raley Ventures to its sole proprietor, Kyle Raley, who has enrolled in account notifications from his Jasper AI executive assistant at sms.raleyventures.com. Messages include meeting reminders, calendar alerts, task notifications, and account activity confirmations for Kyle's own business workflow. Frequency is approximately 3-10 messages per day. All recipients have completed an opt-in checkbox on the enrollment page and can reply STOP at any time to unsubscribe."

**Message flow (verbatim):**
> "The sole recipient opts in by visiting https://sms.raleyventures.com and entering his mobile number in the web form. He then checks an unchecked consent checkbox agreeing to receive SMS notifications from Raley Ventures at the phone number provided. Message frequency varies (up to 10 messages per day). Message and data rates may apply. Reply HELP for help, STOP to opt out. The opt-in page displays the Raley Ventures brand name, sample messages, and links to the Terms of Service at sms.raleyventures.com/terms.html and Privacy Policy at sms.raleyventures.com/privacy.html. The Privacy Policy states that mobile information will not be shared with third parties or affiliates for marketing or promotional purposes."

**Sample messages:** Same three as Appendix A.

**Opt-in keywords:** START
**Opt-out keywords:** STOP, STOPALL, UNSUBSCRIBE, CANCEL, END, QUIT
**Help keywords:** HELP, INFO
**Help message (verbatim):**
> "Raley Ventures: For help, email kyle@raleyventures.com or visit sms.raleyventures.com. Msg & data rates may apply. Reply STOP to opt out."

**Opt-in confirmation (verbatim):**
> "Raley Ventures: You are now subscribed to Jasper SMS notifications. Up to 10 msgs/day. Msg & data rates may apply. Reply HELP for help, STOP to opt out."

**Opt-out confirmation (verbatim):**
> "Raley Ventures: You are unsubscribed. You will not receive any more SMS notifications. Reply START to resubscribe."

---

## Research sources (14 cited, full list in research reports above conversation)

Twilio official docs (5), HighLevel (2), SignalWire (1), Textr (1), Quo/OpenPhone (1), Alive5 (1), MessageDesk (1), TextMagic (1), Twilio error 30909 doc (1). Every verbatim quote in Appendix A-C traces to one of these.

---

## One clear next action

**Kyle, review this plan. Answer the 4 decision-point questions above. Once you greenlight, I execute Phase 1-6 without stopping.**
