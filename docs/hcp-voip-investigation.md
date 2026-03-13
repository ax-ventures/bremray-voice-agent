# HouseCall Pro VoIP Investigation

**Date:** 2026-03-13
**Investigator:** Claude Code
**Question:** Does HCP's built-in VoIP (used on 734-216-9437) support forwarding calls to an external Twilio number or SIP trunk, enabling our AI agent to intercept inbound calls?

---

## Summary / Bottom Line

**HCP does NOT support SIP trunking and exposes no telephony media stream API.** However, HCP Voice *does* support forwarding calls to an external phone number via its Call Flow system. This means the recommended integration path is:

> Change the existing HCP call forward destination from the Wisconsin call center to our Twilio number. Twilio answers every call and runs the AI agent 24/7. HCP receives the resulting job/lead via REST API after the call ends.

This is viable, well-documented, and requires a **single change** to the existing HCP call flow — swap the forward destination.

### Confirmed Answers (2026-03-13)

| Question | Answer |
|---|---|
| Is a call forward already configured in HCP Voice? | **Yes** — currently forwarding to Wisconsin call center. Change destination to Twilio number. |
| Is Bremray on the HCP MAX plan? | **Yes** — CRM webhooks available if needed. |
| Should the agent handle all calls or only after-hours? | **All calls, 24/7.** After-hours callers must be told callbacks won't happen until next business morning (Mon–Fri 8 AM–5 PM EST). |

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
           └─ ALL calls (24/7) → Forward to Twilio number
                                        │
                                    Twilio webhook → /inbound-call
                                        │
                                    Our Node.js server
                                        │
                                    Gemini Live API (WebSocket)
                                        │
                                    AI agent handles call
                                    (business hours vs. after-hours
                                     behavior determined in agent logic,
                                     not in HCP call flow)
                                        │
                                    HCP REST API → create job/lead
                                    Twilio SMS → notify Brent & Rayno
                                    Gmail SMTP → email bremrayllc@gmail.com
```

**No changes are needed to HCP's existing number.** Bremray keeps `(734) 216-9437` as their business number. The AI agent only needs a Twilio number to receive the forwarded calls (never given to customers).

### Call Flow Configuration in HCP

Since the agent handles all calls, the HCP call flow is simple:

1. Log into HCP → Settings → Communications → Voice tab
2. Open the Call Flow for the main number
3. **Change the existing forward destination** (currently Wisconsin call center) to the Twilio number
4. Set routing to **24/7** (no hours widget needed — agent handles time-awareness in code)
5. Enable **caller ID forwarding** so the Twilio webhook receives the original caller's number

### Environment Variable Impact

The spec already anticipated this. The `.env` variable `TWILIO_PHONE_NUMBER=+17342169437` should be corrected — `(734) 216-9437` is the **HCP number** (never changes). The Twilio number is a separate number we provision for receiving forwarded calls:

```env
TWILIO_PHONE_NUMBER=+1XXXXXXXXXX   # Our Twilio number (receives HCP forwards)
HCP_BUSINESS_NUMBER=+17342169437   # HCP's number — for reference only, not a Twilio number
```

---

---

## Sources

- [Voice FAQs | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/6666356-voice-faqs)
- [Voice Settings Overview | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/6750234-voice-settings-overview)
- [HCP Assist Forwarding | Housecall Pro Help Center](https://help.housecallpro.com/en/articles/9731385-hcp-assist-forwarding)
- [VoIP Phone System for Field Service Businesses | Housecall Pro](https://www.housecallpro.com/features/voice-solutions/)
- [Webhooks | Housecall Pro Public API](https://docs.housecallpro.com/docs/housecall-public-api/46e9e1be07621-webhooks)
