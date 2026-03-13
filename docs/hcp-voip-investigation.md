# HouseCall Pro VoIP Investigation

**Date:** 2026-03-13
**Investigator:** Claude Code
**Question:** Does HCP's built-in VoIP (used on 734-216-9437) support forwarding calls to an external Twilio number or SIP trunk, enabling our AI agent to intercept inbound calls?

---

## Summary / Bottom Line

**HCP does NOT support SIP trunking and exposes no telephony media stream API.** However, HCP Voice *does* support forwarding calls to an external phone number via its Call Flow system. This means the recommended integration path is:

> Configure HCP to forward unanswered or after-hours calls to a Twilio number we control. Twilio answers and runs the AI agent. HCP receives the resulting job/lead via REST API after the call.

This is viable, well-documented, and requires zero changes to Bremray's existing HCP setup beyond a one-time call-flow configuration.

---

## Findings

### 1. HCP VoIP Infrastructure

HouseCall Pro's Voice product is built on **Twilio's infrastructure** (confirmed in HCP help documentation). This has two implications:

- Call quality and reliability are Twilio-grade.
- HCP does **not** expose Twilio's programmable media stream features to customers — it's a walled-garden product built on top of Twilio, not direct Twilio access.

### 2. SIP Trunking

**Not supported.** No SIP trunking option exists in HCP Voice settings or documentation. HCP has no mechanism to bridge a SIP trunk to an external provider.

### 3. Telephony Webhook / Media Stream API

**Not available.** The HCP developer API (`https://docs.housecallpro.com`) provides webhooks and REST endpoints for business objects (jobs, customers, estimates, employees). There are **no telephony webhooks** — no inbound call events, no call audio streams, no TwiML-equivalent hooks.

> The HCP API is a CRM/field-service API, not a telephony API.

### 4. Call Forwarding to External Numbers ✅

**This is supported.** HCP Voice includes a **Forward Call Widget** within its Call Flow builder that allows forwarding calls to either an HCP employee or **an external phone number**. Confirmed from official help docs:

> *"The forward widget lets you forward your calls to an employee or to an external number."*

The Call Hours Widget supports three routing modes:
- **24/7** — always forward
- **Business hours** — route based on company profile hours
- **Custom hours** — configure open/closed windows with separate routing per window

This means Emille can configure HCP to:
- During business hours: ring HCP normally (ring the team's devices)
- After hours / no answer: forward to a Twilio number we control

### 5. HCP Assist (Native AI Answering Service)

HCP launched **CSR AI** in January 2025 — their own 24/7 AI-powered answering and booking service. This is a competing/overlapping feature, but it is a paid HCP add-on with no developer access and it does not integrate with our Gemini-based agent.

---

## Recommended Architecture

```
Caller → (734) 216-9437 [HCP number]
           │
           ├─ During business hours, ring answered → HCP handles normally
           │
           └─ After hours / no answer (rollover) → Forward to Twilio number
                                                        │
                                                    Twilio webhook
                                                        │
                                                    Our Node.js server
                                                        │
                                                    Gemini Live API
                                                        │
                                                    AI agent handles call
                                                        │
                                                    HCP REST API → create job/lead
                                                    Twilio SMS → notify Brent & Rayno
                                                    Gmail SMTP → email bremrayllc@gmail.com
```

**No changes are needed to HCP's existing number.** Bremray keeps `(734) 216-9437` as their business number. The AI agent only needs a Twilio number to receive the forwarded calls (can be any number — it is never given to customers).

### Call Flow Configuration in HCP

1. Log into HCP → Settings → Communications → Voice tab
2. Open the Call Flow for the main number
3. Add a **Call Hours Widget** configured for business hours (Mon–Fri 8 AM–5 PM EST)
4. For the "closed" path: add a **Forward Call Widget** pointing to the Twilio number
5. Set forward transfer timeout to ~20 seconds (per HCP Assist recommendation)
6. Enable **caller ID forwarding** so the Twilio webhook receives the original caller's number

### Environment Variable Impact

The spec already anticipated this. The `.env` variable `TWILIO_PHONE_NUMBER=+17342169437` should be corrected — `(734) 216-9437` is the **HCP number** (never changes). The Twilio number is a separate number we provision for receiving forwarded calls:

```env
TWILIO_PHONE_NUMBER=+1XXXXXXXXXX   # Our Twilio number (receives HCP forwards)
HCP_BUSINESS_NUMBER=+17342169437   # HCP's number — for reference only, not a Twilio number
```

---

## Open Questions for Emille

1. **Does Bremray already forward unanswered calls somewhere?** (Currently to the Wisconsin call center) — If so, the HCP call flow is already partially set up and just needs the destination changed to our Twilio number.
2. **Is the existing HCP Voice subscription the MAX plan?** Webhooks on the CRM side require MAX. Call forwarding does not.
3. **Do you want the AI agent to answer all calls, or only after-hours/overflow?** During business hours, Rayno currently fields some calls. The call flow can be configured either way.

---

## Sources

- [Voice FAQs | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/6666356-voice-faqs)
- [Voice Settings Overview | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/6750234-voice-settings-overview)
- [HCP Assist Forwarding | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/9731385-hcp-assist-forwarding)
- [VoIP Phone System for Field Service Businesses | Housecall Pro](https://www.housecallpro.com/features/voice-solutions/)
- [Webhooks | Housecall Pro Public API](https://docs.housecallpro.com/docs/housecall-public-api/46e9e1be07621-webhooks)
