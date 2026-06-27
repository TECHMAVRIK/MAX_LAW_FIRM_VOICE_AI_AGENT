# Voice AI Scheduling Agent

An end-to-end voice AI system that handles inbound client calls for a law firm: qualifying callers through natural conversation, checking real-time calendar availability, and booking consultations automatically.

## The Problem

Law firm intake calls follow a predictable pattern: a caller explains their situation, gets asked the same qualifying questions, and (if eligible) needs to find a time that works for both them and an available lawyer. This is repetitive, time-sensitive, and easy to automate without losing the human warmth callers need when they're often calling during a stressful moment.

## What It Does

1. A caller dials in and speaks with **Astha**, a voice AI agent built on Retell AI
2. Astha asks qualifying questions one at a time (situation, urgency, location, dependents involved), adapting based on what the caller volunteers
3. If the caller qualifies, Astha offers real, live appointment slots pulled from the firm's calendar (no hardcoded times)
4. The caller picks a time, Astha confirms their name, and books the consultation automatically
5. If the caller doesn't qualify (e.g., outside the firm's practice area or service region), the call ends gracefully with appropriate guidance

## How It Works (Architecture)

```
Inbound Call (Retell AI)
        |
        v
Webhook (n8n) -- triggered before the agent starts speaking
        |
        v
Set current time, lookback window, calendar event type
        |
        v
HTTP Request -> Cal.com API (live availability for next 7 days)
        |
        +--> LLM Chain #1 (Groq/Llama 3.1) -> formats FULL week availability into natural speech
        |
        +--> LLM Chain #2 (Groq/Llama 4 Scout) -> picks 2 best slots, formats as a short spoken offer
        |
        v
Respond to Webhook -> injects both as dynamic variables into the Retell agent
        |
        v
Astha speaks natural, accurate availability to the caller in real time
```

The key design choice: instead of having the voice agent reason about raw calendar JSON live during a call (slow, error-prone), all the date/time formatting work happens upfront in the automation layer, using small fast LLMs optimized for low latency. The voice agent just speaks the pre-formatted result.

## Tools & Stack

- **Retell AI** — voice agent orchestration, speech-to-text/text-to-speech, conversation logic
- **n8n** — backend automation and orchestration (webhook, API calls, branching logic)
- **Cal.com API** — live calendar availability and booking
- **Groq (Llama 3.1 / Llama 4 Scout)** — fast LLM inference for natural language formatting of scheduling data

## Key Decisions & Tradeoffs

- **Two separate LLM calls instead of one**: one model formats the full week's availability (used if the caller wants more options), the other condenses it to just two slots (used for the first offer). This keeps the first response fast and simple while still supporting a richer fallback.
- **Real-time data, not cached**: availability is pulled fresh from Cal.com on every inbound call, so the agent never offers a slot that's already been booked.
- **Qualification before booking**: the agent filters out non-matching inquiries (wrong practice area, wrong region) before offering any scheduling, respecting both the caller's time and the firm's capacity.

## What I'd Improve Next

- Add error handling for when the Cal.com API call fails or returns no slots, so the call degrades gracefully instead of breaking
- Resolve a timezone inconsistency between the automation backend (configured for IST) and some hardcoded example phrasing (referencing US Eastern time)
- Add logging/analytics on call outcomes (qualified vs. not, booked vs. abandoned) to measure conversion

## Security Note

An earlier version of this project had an API key hardcoded in a workflow config file. It was identified and fixed by moving the credential into n8n's secure credential store and rotating the exposed key — the current version contains no embedded secrets.

---

*Built as a personal project to explore voice AI agent design and product-level automation thinking.*
