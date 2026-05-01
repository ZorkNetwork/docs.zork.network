# Zorkcoin Brand & Product Philosophy Guide
*A reference document for AI agents, developers, designers, and contributors*

---

## 1. Core Identity

**Tagline:**  
> Explore Your Money.

**Brand Essence:**  
Zorkcoin is a simple, honest, and understandable digital currency that allows users to explore, learn, and confidently use their money.

**Positioning Statement:**  
Zorkcoin is a straightforward, privacy-enabled digital currency designed for people who want to understand how their money works—not just use it blindly.

---

## 2. Core Brand Attributes

Zorkcoin should consistently express the following qualities:

- **Open** → Transparent, approachable, non-exclusive  
- **Honest** → No hype, no hidden complexity, no misleading claims  
- **Exploring** → Encourages curiosity and learning  
- **Rewarding** → Understanding leads to confidence and satisfaction  
- **Fresh** → Clean, modern, uncluttered  
- **Useful** → Practical, reliable, everyday utility  

---

## 3. Strategic Direction

### Primary Identity
> **A trusted financial tool first**

Zorkcoin must feel:
- Reliable
- Predictable
- Calm
- Understandable

### Secondary Identity
> **Exploration is optional, never required**

Zorkcoin includes an **optional discovery layer** inspired by classic text-based systems (e.g., early computing environments and interactive systems), but:

- It must never interfere with core functionality
- It must never confuse first-time users
- It must never feel gimmicky or game-like

---

## 4. Target Audience

Primary audience:
- Crypto newcomers
- Older or more cautious users (35+)
- Curious, thoughtful individuals
- Users skeptical of crypto complexity

User mindset:
> “I want to understand what I’m doing before I trust it.”

---

## 5. Product Philosophy

### Guiding Principle
> **Clarity over cleverness. Always.**

If a feature:
- Adds confusion
- Requires explanation to function
- Slows down core actions

…it should be simplified or removed.

---

## 6. UX Design Principles

### 6.1 Familiar Structure
The wallet should remain structurally similar to established UIs derived from:
- Bitcoin Core
- Litecoin Core

Avoid unnecessary structural changes.

---

### 6.2 Layered Understanding

Do not remove technical concepts. Instead:

- Present simple explanations first
- Allow deeper inspection optionally

Example:
- Show “Private transfer”  
- Allow expansion to reveal technical details (e.g., MWEB)

---

### 6.3 Explain Everything

Every user action should be understandable.

#### Required Feature:
**“Explain This Transaction”**

Must clearly describe:
- What happened
- Where funds went (as clearly as possible)
- What the fee was
- Whether privacy features were used
- Transaction status

All explanations must be in plain English.

---

### 6.4 No Jargon in Primary UI

Avoid exposing:
- Acronyms (e.g., UTXO, MWEB) without explanation
- Internal terminology
- Developer-centric language

---

### 6.5 Predictability

The system must behave in ways users expect:

- No unexplained outputs (e.g., change addresses must be explained)
- No surprising balance changes
- No hidden mechanics

---

## 7. Language & Tone Guidelines

### Tone Characteristics

- Clear
- Calm
- Direct
- Respectful
- Non-technical by default

---

### Avoid

- Hype language
- Superiority claims
- Meme culture
- Overly playful or gamified phrasing
- Fear-based warnings

---

### Preferred Language Patterns

| Instead of | Use |
|-----------|------|
| Execute transaction | Send funds |
| Broadcast to network | Share with the network |
| Invalid address | This address doesn’t look correct |
| Confidential transaction | Private transfer |

---

### Privacy Messaging

Position privacy as:
> Optional, practical, and user-controlled

Avoid:
- “Anonymity” framing
- Extreme or ideological messaging

---

## 8. Visual Design Principles

### Style Direction
- Clean and modern
- Light, structured layouts
- Minimalist but not sterile
- Inspired by early computing clarity (not retro aesthetics)

### Typography
- Modern sans-serif (e.g., Exo 2 or similar)
- High readability
- No novelty fonts

### Avoid
- Pixel art
- Arcade/game aesthetics
- Neon-heavy palettes
- Overly stylized “retro” visuals

---

## 9. Interaction Model

### Primary Actions (must remain simple)
- Send
- Receive
- View Transactions

These must:
- Be immediately visible
- Require no explanation to begin using

---

### Secondary Actions (optional exploration)
- Guided onboarding
- Learning modules
- CLI / advanced interaction
- Educational walkthroughs

These should live in:
> “Explore” or equivalent secondary area

---

## 10. Feature Framing

### Privacy (MWEB or equivalent)
- “Private transfer”
- “Optional privacy”
- “You choose when to use it”

---

### Fees
- Always visible before sending
- Explained as:
  > A small cost to process the transaction

---

### Confirmations
- Explained simply:
  > The network is verifying this transaction

---

### Addresses
- Must be explained once clearly:
  > A destination for receiving funds

---

## 11. What Not To Do

- Do not rename core actions (Send, Receive)
- Do not gamify financial interactions
- Do not obscure technical reality
- Do not overwhelm users with information
- Do not assume prior crypto knowledge
- Do not prioritize novelty over clarity

---

## 12. Brand Experience Summary

Zorkcoin should feel like:

> A cryptocurrency that respects the user’s intelligence by making things understandable.

Not:
- A game
- A speculative tool
- A developer-first platform

---

## 13. Final Guiding Statement

> Zorkcoin is the cryptocurrency for people who want to understand what they’re doing.

Every decision—technical, design, or marketing—should support this.

---

## 14. Implementation Heuristic

Before shipping any feature, ask:

1. Is this clear to a first-time user?
2. Does this increase or reduce trust?
3. Can this be explained in one sentence?
4. Does this align with “Explore Your Money”?

If any answer is “no,” refine the implementation.

---

End of document.