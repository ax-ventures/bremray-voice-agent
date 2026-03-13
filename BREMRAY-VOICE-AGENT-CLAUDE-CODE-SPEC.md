# Bremray Electrical — AI Voice Agent
## Claude Code Implementation Specification

**Version:** 1.0  
**Prepared for:** Claude Code CLI session  
**Owner:** Emille Hall, CEO — Bremray Electrical, Ann Arbor, MI

---

## 1. Project Overview

Build a production-grade AI voice agent to replace Bremray Electrical's third-party answering service (Wisconsin call center). The agent answers inbound calls, collects structured lead data, creates records in HouseCall Pro, and notifies field technicians via SMS and email.

**The problem being solved:** Emille is the only office-side operator. Brent and Rayno are both in the field. Calls are currently forwarded to a Wisconsin call center that gives inaccurate information and sends a generic email to bremrayllc@gmail.com. This agent replaces that workflow entirely with an intelligent, accurate, 24/7 front-of-house system.

---

## 2. Critical Architecture Decision (Claude Code Must Resolve First)

### HouseCall Pro VoIP Integration
Bremray's phone system is **built into HouseCall Pro**. The dedicated business number is `(734) 216-9437`. HCP provides the VoIP layer.

**Claude Code must investigate before building:**
- Does HCP's VoIP support SIP trunking or call forwarding to an external number (e.g., Twilio)?
- Does HCP expose a telephony webhook or media stream API?
- If HCP VoIP cannot be intercepted, the fallback is to forward unanswered/after-hours calls to a Twilio number that runs the agent

**Do not scaffold the telephony layer until this is resolved.** Check HCP's API documentation at https://developer.housecallpro.com and their VoIP/phone settings documentation.

---

## 3. Preferred Technology Stack

### Language
**Node.js** — Emille's preferred language. He is experienced with JavaScript/Node.js, SQL, CLI tools, GitHub, and API integrations on both Mac and Windows.

> **Note for Claude Code:** Gemini's architecture report recommends Python/FastAPI due to the maturity of the Gemini Live API Python SDK. Before proceeding, evaluate the current state of the Gemini Live API Node.js SDK (`@google/generative-ai` or `@google-cloud/vertexai`). If the Node.js SDK fully supports bidirectional audio streaming via WebSockets (required for real-time voice), proceed in Node.js. If not, present the trade-offs to Emille and recommend the best path. Do not assume Python without confirming Node.js viability first.

### Core Stack
- **Runtime:** Node.js 20+ (LTS)
- **Web Framework:** Fastify or Express (Claude Code to recommend based on WebSocket performance)
- **Telephony:** Twilio (Media Streams for real-time audio)
- **AI/Voice:** Google Gemini Multimodal Live API (native audio, not STT+TTS pipeline)
- **Schema Validation:** Zod (Node.js equivalent of Pydantic)
- **CRM:** HouseCall Pro API
- **Notifications:** Twilio SMS + Gmail SMTP (nodemailer)
- **Hosting:** Google Cloud Run or Railway (containerized, always-on)
- **Secrets:** `.env` file locally, Google Secret Manager in production
- **Version Control:** GitHub

---

## 4. Business Context

### Company
- **Name:** Bremray Electrical
- **Location:** Ann Arbor, Michigan
- **Business Number:** (734) 216-9437
- **Email:** bremrayllc@gmail.com
- **Founded:** 1988, serving community since 2004
- **License:** Licensed, bonded, and insured electrical contractor
- **Owner:** Emille Hall (CEO, PMP certified, Journeyman license)

### Team
- **Brent** — Field Electrician
- **Rayno** — Field Electrician (currently also fields calls)
- **Emille** — CEO, handles permits, scheduling inspections, invoices, bills, and company organization

### Business Hours
- **Monday–Friday: 8:00 AM – 5:00 PM EST**
- **Weekends/After hours:** Agent takes messages only, no dispatch

### Service Area (25-mile radius from Ann Arbor)
- Ann Arbor (primary — historic neighborhoods: Burns Park, Old West Side, Water Hill, Kerrytown, Old Fourth Ward; new construction: NE Plymouth Road area)
- Ypsilanti
- Chelsea
- Saline
- Dexter
- Pittsfield Township
- Milan, Manchester, Whitmore Lake (extended)
- Washtenaw County broadly

**Agent must verify caller address is within service area before collecting full lead info.**

---

## 5. Services

### High-Ticket Focus (Primary — agent should emphasize these)
| Service | Notes |
|---|---|
| Whole House Generators | Generac authorized dealer/installer. Automatic standby + portable interlock kits |
| Tesla Powerwalls | Battery storage installation |
| EV Chargers | Tesla Wall Connector, ChargePoint, NEMA 14-50. DTE Energy rebates available |
| SPAN Smart Panels | Recently certified installer. App-controlled circuit management, pairs with solar/EV/storage |
| Tigo Energy Systems | Certified Tigo installer. TS4 MLPE solar optimizers, GO battery storage, EI inverters, Energy Intelligence monitoring |

### General Services (Secondary)
- Electrical panel upgrades (100A → 200A)
- **Hazardous panel replacement:** Federal Pacific Stab-Lok (25-30% failure rate, fire risk, insurance issues) and Zinsco/Sylvania-Zinsco panels — agent must flag these as HIGH PRIORITY leads
- Residential wiring, outlets, switches, GFCI
- Smart home integration, surge protection
- Ceiling fan installation
- Commercial electrical (light commercial)
- **No industrial work**

### Pricing Policy
- **NEVER quote prices** — not even ballpark ranges
- All pricing deferred to callback from Brent or Rayno
- Agent response: *"One of our electricians will give you a call back to go over pricing and set up a time that works for you."*

---

## 6. Call Handling Logic

### During Business Hours (Mon–Fri 8AM–5PM)
1. Greet caller, identify Bremray Electrical
2. Ask how you can help
3. Collect lead information (see Section 7)
4. Create/update HCP record
5. Notify Brent and Rayno via SMS
6. Send confirmation email to bremrayllc@gmail.com

### After Hours / Weekends
1. Greet caller, state business hours
2. Offer to take their information for a next-business-day callback
3. Collect lead information
4. Same HCP + SMS + email workflow

### Emergency Calls (Burning smell, sparks, smoke, hot panel)
Bremray does **NOT** offer emergency service. Agent response:
1. Acknowledge the urgency calmly
2. Advise caller to call **911** if there is smoke or fire
3. Advise caller to call **DTE Energy at 800-477-4747** for utility emergencies
4. Offer to schedule a next-business-day inspection
5. Collect contact info if they want a callback
6. Flag lead as `is_emergency: true` in HCP note

### Out of Service Area
- Politely inform caller Bremray doesn't service their area
- Do not collect lead data
- Suggest they contact a local licensed electrician

---

## 7. Lead Data Collection Schema (Zod)

```javascript
const ElectricalLeadSchema = z.object({
  caller_name: z.string(),                          // First and last name
  callback_number: z.string(),                      // 10-digit, confirm with caller
  property_address: z.string(),                     // Must be within service area
  is_existing_customer: z.boolean(),                // Check against HCP
  service_category: z.enum([
    'EV Charger',
    'Generator',
    'Tesla Powerwall',
    'SPAN Panel',
    'Tigo Solar System',
    'Panel Upgrade',
    'Hazardous Panel Replacement',  // FPE or Zinsco — HIGH PRIORITY
    'Residential Wiring',
    'Commercial Electrical',
    'Smart Home',
    'Other'
  ]),
  issue_description: z.string(),                    // Detailed, include brands/home age if mentioned
  is_emergency: z.boolean(),                        // True if safety hazard keywords detected
  appointment_preference: z.string().optional(),    // Preferred callback/visit time
  how_did_you_hear: z.string().optional(),          // Referral source
  hcp_customer_id: z.string().optional(),           // Populated if existing customer found in HCP
});
```

---

## 8. HouseCall Pro Integration

### API Reference
- Base URL: `https://api.housecallpro.com`
- Docs: https://developer.housecallpro.com
- Auth: API Key (store in `.env` as `HCP_API_KEY`)

### Logic
```
IF caller is existing HCP customer:
  → Create new Job (status: unscheduled) on existing customer record
  → Assign to both Brent and Rayno
  → Add note with full call transcript summary

IF caller is new customer:
  → Create new Lead in HCP
  → Include all collected schema fields in lead notes
  → Do NOT auto-schedule
```

### HCP Notification
- Both Brent and Rayno should receive HCP job/lead notifications via their existing HCP mobile app
- Claude Code must verify HCP supports dual-assignee on jobs/leads

---

## 9. Notifications

### SMS (Twilio)
Send to both Brent and Rayno immediately after call ends:

```
New Bremray Lead — [SERVICE_CATEGORY]
Name: [CALLER_NAME]
Phone: [CALLBACK_NUMBER]
Address: [PROPERTY_ADDRESS]
Issue: [ISSUE_DESCRIPTION]
[EMERGENCY FLAG IF APPLICABLE]
HCP record created. Check app.
```

> **Brent and Rayno's phone numbers:** Emille to provide before build begins — store in `.env` as `TECH_SMS_BRENT` and `TECH_SMS_RAYNO`

### Email (Gmail SMTP)
Send to bremrayllc@gmail.com after every call:
- Subject: `New Lead — [SERVICE_CATEGORY] — [CALLER_NAME]`
- Body: Full structured lead data + call summary
- Use nodemailer with Gmail app password (store in `.env`)

---

## 10. Agent Voice & Personality

### System Prompt Guidelines
- **Name:** "You are the virtual receptionist for Bremray Electrical."
- **Tone:** Professional, warm, competent. Local business feel — not a corporate script.
- **Never:** Quote prices, promise availability, schedule without human confirmation
- **Always:** Confirm service area, collect complete contact info, explain that an electrician will call back
- **On FPE/Zinsco panels:** Express appropriate urgency — these are fire hazards and should be addressed promptly
- **On emergencies:** Calm, clear, direct to 911 or DTE Energy, offer next-day inspection

### Sample Opening
*"Thank you for calling Bremray Electrical. This is the Bremray virtual receptionist. How can I help you today?"*

---

## 11. Environment Variables (.env)

```env
# Twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=+17342169437

# Google Gemini
GEMINI_API_KEY=

# HouseCall Pro
HCP_API_KEY=

# Gmail
GMAIL_USER=bremrayllc@gmail.com
GMAIL_APP_PASSWORD=

# Technician SMS
TECH_SMS_BRENT=
TECH_SMS_RAYNO=
TECH_SMS_EMILLE=

# App
PORT=8000
NODE_ENV=development
```

---

## 12. Repository Structure

```
bremray-voice-agent/
├── CLAUDE.md                    # Claude Code instructions (see Section 13)
├── .env.example
├── .gitignore
├── package.json
├── src/
│   ├── index.js                 # Entry point, Fastify/Express server
│   ├── routes/
│   │   ├── inbound-call.js      # Twilio webhook — TwiML response
│   │   └── media-stream.js      # WebSocket handler — Twilio ↔ Gemini proxy
│   ├── services/
│   │   ├── gemini.js            # Gemini Live API connection + audio handling
│   │   ├── hcp.js               # HouseCall Pro API (customer lookup, job/lead creation)
│   │   ├── notifications.js     # Twilio SMS + nodemailer email
│   │   └── audio.js             # Audio transcoding (G.711 mu-law ↔ PCM)
│   ├── schemas/
│   │   └── lead.js              # Zod schema for ElectricalLead
│   └── config/
│       └── index.js             # Environment variable validation
├── tests/
│   ├── schemas.test.js
│   ├── audio.test.js
│   └── hcp.test.js
└── docs/
    └── hcp-voip-investigation.md  # Claude Code to populate this first
```

---

## 13. CLAUDE.md Contents (Place in repo root)

```markdown
# Bremray Electrical Voice Agent — Claude Code Instructions

## Stack
- Node.js 20+ LTS
- Fastify or Express (evaluate for WebSocket performance)
- Twilio Media Streams (bidirectional audio WebSocket)
- Google Gemini Multimodal Live API (native audio streaming)
- Zod for schema validation
- HouseCall Pro REST API
- nodemailer for Gmail SMTP
- Twilio REST API for SMS

## Audio Specifications
- Twilio inbound: Base64 encoded G.711 mu-law @ 8,000 Hz
- Gemini requires: PCM 16-bit little-endian @ 16,000 Hz
- Gemini outputs: PCM 16-bit little-endian @ 24,000 Hz
- Outbound to Twilio: Must downsample to G.711 mu-law @ 8,000 Hz
- Use node-based DSP library for transcoding (evaluate `@flo-bit/audio-utils` or raw Buffer manipulation)

## Critical First Task
Before writing any code, investigate whether HouseCall Pro's built-in VoIP
(used on number 734-216-9437) supports:
1. SIP trunking to Twilio
2. Call forwarding to external number
3. Overflow/after-hours forwarding
Document findings in docs/hcp-voip-investigation.md before proceeding.

## HCP API
- Base: https://api.housecallpro.com
- Docs: https://developer.housecallpro.com
- Existing customer → new unscheduled job
- New customer → new lead
- Both Brent and Rayno assigned

## Business Rules (Never Violate)
- NEVER quote prices
- NEVER schedule without human confirmation
- ALWAYS verify address is within 25-mile Ann Arbor service radius
- ALWAYS send SMS to both techs after call
- ALWAYS email bremrayllc@gmail.com after call
- Emergency calls (fire, sparks, smoke) → direct to 911 + DTE Energy, offer next-day inspection

## Testing
- Write tests before or alongside implementation
- Use Jest for unit tests
- Test Zod schemas, audio transcoding, and HCP API calls
- Run tests before every commit

## Security
- All secrets in .env — never hardcode
- Validate all inbound Twilio webhooks using Twilio signature verification
- Use HTTPS only in production
```

---

## 14. Claude Code Session Startup Prompt

Paste this to begin the Claude Code session:

```
Read CLAUDE.md. We are building the Bremray Electrical AI voice agent.

Before writing any code, complete these two tasks in order:

1. Investigate HouseCall Pro's VoIP system to determine if their built-in phone 
system (used on 734-216-9437) supports forwarding calls to an external Twilio 
number or SIP trunk. Check https://developer.housecallpro.com and search for 
HCP VoIP documentation. Document your findings in docs/hcp-voip-investigation.md.

2. Evaluate the current state of Google Gemini Multimodal Live API support in 
Node.js. Determine if bidirectional audio streaming via WebSockets is fully 
supported. If yes, confirm we proceed in Node.js. If not, present the trade-offs 
and recommend the best path.

Do not write application code until both investigations are complete and I have 
approved the findings.
```

---

## 15. Known Risks & Open Questions

| Item | Risk | Resolution |
|---|---|---|
| HCP VoIP forwarding | HCP may not support SIP/forwarding to Twilio | Fallback: forward unanswered calls to a separate Twilio number |
| Gemini Live API in Node.js | SDK may not fully support audio streaming | Claude Code to evaluate and recommend |
| Dual SMS assignment in HCP | HCP may not support two assignees on one job | Fallback: assign to Emille, SMS both techs directly via Twilio |
| Audio transcoding in Node.js | Less mature than Python audioop | Use native Buffer manipulation or evaluate npm DSP libraries |
| After-hours call volume | Unknown — may need rate limiting | Implement basic rate limiting from day one |
