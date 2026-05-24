# Prompt Design — Bloom Aesthetics Clinic AI Agent

## System Prompt

The full system prompt is embedded in `main.py` under the `SYSTEM_PROMPT` constant. It is passed to the model on every API call as the `system` parameter.

---

## Design Decisions

### 1. SOP Embedded Directly in Prompt

The entire `sop.json` is serialised as JSON and injected into the system prompt at runtime. This ensures:
- The model sees the exact, authoritative data source on every turn
- No retrieval step needed for a small SOP (reduces latency and failure points)
- Easy to swap: update `sop.json` → prompt updates automatically

**Trade-off**: For larger SOPs (>10k tokens), this approach becomes costly. Would switch to RAG (vector search over SOP chunks) at that point.

---

### 2. Hallucination Prevention

Three explicit mechanisms:

| Mechanism | Implementation |
|---|---|
| Hard boundary instruction | "You ONLY answer using the information below. Never invent facts, prices, or policies not listed here." |
| Graceful gap handling | Explicit fallback phrase defined: *"I don't have that information, but I'll connect you with our team."* |
| Escalation on uncertainty | After 2 unanswered questions, escalation is triggered — model cannot keep guessing |

The model is never told to "do its best" on unknown topics. It is explicitly told to escalate instead.

---

### 3. Confidence-Based Escalation

Escalation is **output-format based**, not threshold-based. The model is instructed to append:

```
[ESCALATE: <reason>]
```

to its message whenever an escalation trigger is met. The Python code parses this tag with regex, logs it to `escalation_log.jsonl`, and strips it from the displayed message.

**Why output-format over threshold?**
- No access to logprobs in standard API calls
- Output-format approach is deterministic and auditable
- Reason is human-readable and logged with full context

**Escalation triggers defined:**
1. Anger / frustration / complaint language
2. Medical questions (side effects, suitability, contraindications)
3. Pricing negotiation attempts
4. >2 questions unanswerable from SOP
5. Explicit request to speak to a human

---

### 4. Tone & Persona

**Target persona**: Friendly, premium front-desk receptionist at a boutique aesthetics clinic.

Design choices:
- Warm but professional — not robotic, not overly casual
- Short responses (2-4 sentences) to match WhatsApp/chat norms
- Uses customer's name when shared (builds rapport)
- No medical jargon beyond SOP terms
- Reassuring language for first-time customers who may be nervous about treatments

**Why this tone?**
SMB aesthetics clinics compete on trust and personal feel. A cold, corporate tone would undermine the brand. A too-casual tone would reduce perceived expertise.

---

### 5. Stage Management

The 4 stages (FAQ → Qualification → Escalation → Summary) are described in the system prompt, not enforced by separate API calls. The model is trusted to transition naturally, which:
- Keeps the conversation feeling organic
- Reduces latency (single model call per turn)
- Allows escalation to interrupt any stage

For a production system, explicit state machine enforcement (checking stage in Python, routing to different prompts) would be more reliable.

---

### 6. Conversation History

Full message history is passed on every API call (`messages=state.history`). This gives the model:
- Context for lead qualification (avoid re-asking answered questions)
- Sentiment trend across the conversation
- Accurate summary generation at session end

---

## Known Limitations

- **No memory across sessions**: Each run starts fresh. Production would persist to a DB.
- **Single-turn escalation**: Once escalated, the bot continues responding. Production would hand off to a human queue.
- **SOP size**: Prompt-stuffing works for small SOPs. Needs RAG for real business SOPs.
- **Language**: English only. SMB customers may write in other languages.
