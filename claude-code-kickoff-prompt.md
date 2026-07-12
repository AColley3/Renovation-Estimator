# Project Kickoff: Home Renovation Assistant (working name: "HouseSmart" — suggest alternatives if you like)

Read this entire brief before doing anything. Your first job is NOT to write code — it is to ask me clarifying questions and then produce a plan.md. Details in "Your First Tasks" at the bottom.

## Who you're working with

I am not a developer. This changes how you must work:

- Explain every decision in plain language. No unexplained jargon.
- When I need to do something (create an account, add an API key, click something in a dashboard), stop and give me exact step-by-step instructions, then wait.
- Give me exact copy-paste terminal commands. Never say "just run the migration" — show me the command.
- I cannot debug. If something breaks, you diagnose and fix it. Build defensively: good error handling, clear error messages.
- Every phase must end with something I can open in a browser and test myself. You will give me a short "how to test this" checklist at the end of each phase.

## The product

A web app that acts as a renovation and maintenance assistant for a house it deeply knows.

**The core insight — never lose this:** Anyone can paste photos into an AI chatbot and ask about skirting boards. The value of THIS product is the persistent context layer: a structured profile of the user's house (rooms, dimensions, materials, photos, floor plans), a catalogue of tools and products they already own, and a history of past projects. Every AI interaction is grounded in that context, so answers are specific to THIS house, and the profile improves with every completed job. We are building a structured memory + workflow system, with AI as the engine — not a thin chat wrapper.

**Example interactions it must eventually handle:**
- "I want to replace the skirting boards in the spare bedroom, they're damaged" → it already knows the room, its dimensions, the flooring type, and which tools I own.
- "I'm having drainage issues down the side of the house, it drops toward the back and water is pooling" + uploaded photos → it locates this on the property, assesses from the photos, and runs the project workflow.

## V1 scope

**IN scope:**
1. **House profile**: create a house; add rooms/outdoor areas; upload floor plans and photos tagged to rooms/areas; AI-assisted extraction of structured details (dimensions, flooring, wall materials, fixtures, known issues) which the user confirms or edits — AI output is always a draft the human approves.
2. **Inventory**: catalogue of tools and products the user owns. Quick manual add, plus "photo of my shelf/garage" AI-assisted recognition producing a draft list the user confirms.
3. **Project workflow** (the heart of the app — full spec below): describe a job in chat with optional photos → triage → research → measurements → materials & tools list → cost estimate → step-by-step guide with hold points → completion → house profile update.
4. **Project dashboard**: list of projects with status (planning / in progress / on hold / complete).
5. **Multi-user, invite-only**: me plus a handful of friends, each with their own house profile, fully isolated from each other. No public signup.
6. **Mobile-first responsive design**: photos will mostly be taken and uploaded from a phone while standing in the room. Desktop is secondary.

**OUT of scope for v1 — do not build, put in a backlog section of plan.md instead:**
- Payments, subscriptions, billing of any kind
- Live retailer price scraping or retailer API integrations (use search links + estimate ranges instead)
- Native mobile apps (responsive web only)
- YouTube API integration (plain links to relevant searches/videos are fine)
- Public signup, password reset flows beyond what the auth provider gives us for free
- Notifications/emails
- Anything else not listed as IN scope — ask me before adding features

## Tech stack — locked unless you hit a genuine blocker

- **Next.js (App Router) + TypeScript**, Tailwind CSS + shadcn/ui
- **Supabase**: Postgres database, Auth (invite-only), Storage for photos/floor plans, Row Level Security so users can only ever see their own data
- **Anthropic API** for all AI features (I will provide a key as an environment variable; never hardcode or commit it)
  - Default model: Claude Sonnet (current version) with vision for photo analysis and the web search tool for research
  - Use Claude Haiku (current version) for cheap lightweight tasks (e.g. tagging, short classifications)
- **Vercel** for hosting and deployment
- Secrets only ever in environment variables. Add `.env*` to .gitignore in the very first commit.

If you believe something here is wrong for the job, say so and explain in plain language — but don't silently deviate.

## Data model — starting point (refine in plan.md)

users, houses, rooms/areas (indoor and outdoor), photos (belongs to a room/area, with caption + AI-extracted observations), floor_plans, inventory_items (name, category, condition, photo, notes), projects (title, description, status, triage_result), project_steps (ordered, with is_hold_point flag and completion state), materials_list_items (linked to project; name, quantity, unit, est. cost range, already_owned flag, purchase search link), house_updates (log of profile changes made when projects complete).

## The project workflow — full spec

This pipeline is the product. Each stage's output is saved and visible on the project page, not trapped in a chat transcript.

1. **Intake**: user describes the job conversationally, optionally attaching photos. The app assembles context: house profile, relevant room(s), relevant photos, inventory.
2. **Triage (mandatory, before anything else)**: classify the job as one of:
   - 🟢 **DIY-safe** — proceed to full workflow
   - 🟡 **DIY with caution** — proceed, but state the specific risks and what makes this harder than it looks
   - 🔴 **Licensed trade required** — do NOT produce step-by-step instructions. Instead produce: why it requires a trade (legal/safety reason), what to ask tradespeople when getting quotes, red flags in quotes, and a rough expected cost range for the region.
   - Australian rules the triage must respect: virtually all electrical work is illegal for unlicensed people; most plumbing and gasfitting requires a licensed plumber/gasfitter; structural work, significant drainage/stormwater work, and some fencing/retaining walls may require council approval — flag when approval might apply and tell the user to check with their local council.
3. **Research**: use web search to ground the approach in current best practice for the user's region (Australia — materials, terminology, standards). Save a short "approach summary" with source links.
4. **Measurement checklist — hold point**: NEVER estimate quantities or dimensions from photos. Photos have no scale. Instead, generate a specific measurement checklist ("length of each wall at floor level in the spare bedroom; count doorways and note frame widths"). The project pauses here until the user enters measurements.
5. **Materials & tools list**: from measurements, calculate quantities including standard waste factors (state the waste % used). Cross-check the tools/materials required against the user's inventory — only list what's missing under "you need to get". Items they own appear under "you already have".
6. **Cost estimate**: price *ranges* per item (clearly labelled as estimates), a total range, and for each item a search link to Bunnings and one other relevant Australian retailer. No scraping, no claimed live prices.
7. **Step-by-step guide**: numbered steps with: tools needed per step, safety warnings where relevant, **hold points** (explicit "stop and check X before continuing" gates, e.g. "confirm the first board is level before fixing the rest"), and rough time estimates. Include links to recommended video searches for visual steps.
8. **Completion loop**: when the user marks the project complete, prompt for outcome photos and update the house profile (e.g. spare bedroom skirting → new, replaced [date]). Log the change in house_updates.

## Safety & liability guardrails

- Triage cannot be skipped or overridden into producing 🔴-category instructions.
- A clear disclaimer on every generated guide: general information, not professional advice; user is responsible for checking local regulations; when in doubt, consult a licensed professional.
- Guides must call out mandatory safety equipment per step where relevant.
- Never present AI-extracted house details or estimates as verified fact — always user-confirmable drafts and labelled estimates.

## AI cost guardrails (my API key pays for all users)

- Per-user daily and monthly usage caps, enforced server-side, with a friendly "limit reached" message. Make the cap values easy for me to change in one place.
- Log every AI call: user, purpose, model, input/output tokens. Simple admin page where I can see usage per user.
- Resize/compress photos before sending to the API; cap the number of photos included per AI request.
- Use Haiku wherever Sonnet-level intelligence isn't needed.
- All AI calls happen server-side only. The API key must never reach the browser.

## Your First Tasks — in this exact order

1. **Ask me your clarifying questions.** Anything ambiguous in this brief, anything you'd challenge, anything you need decided. Batch them into one round if possible.
2. **Write `plan.md`** — the living build plan. It must contain:
   - A one-paragraph product summary (so future sessions regain context fast)
   - The refined data model
   - Phased milestones. Proposed structure (adjust with justification):
     - **Phase 0 — Walking skeleton**: repo, Next.js scaffold, Supabase project connected, deployed to Vercel, I can log in and see a "hello" dashboard *live on the internet*. Nothing else. This proves the whole pipeline before we invest in features.
     - **Phase 1 — House profile**: houses, rooms, photo upload + tagging, floor plan upload, AI-assisted detail extraction with user confirmation.
     - **Phase 2 — Inventory**: manual add + AI photo-recognition draft lists.
     - **Phase 3 — Project intake + triage**: chat intake with house context, triage classification, project dashboard.
     - **Phase 4 — Full workflow**: research, measurement hold point, materials/tools list vs inventory, cost estimate ranges + links, step-by-step guide with hold points, completion loop.
     - **Phase 5 — Friends**: invite-only multi-user, RLS verified, usage caps + admin usage page.
     - **Phase 6 — Polish**: mobile UX pass, empty states, error states, onboarding flow.
   - For every phase: goal, task checklist, definition of done, and a "How Nate tests this" checklist written for a non-developer.
   - A **Backlog** section for all out-of-scope ideas.
   - A **Decisions log** (date, decision, why).
3. **Write `CLAUDE.md`** with project conventions: stack, commands, working rules from this brief, and anything a fresh session needs to continue work seamlessly.
4. **Stop and wait for my approval of plan.md before writing any application code.**

## Working rules — permanent, non-negotiable

- Work one phase at a time. Never start the next phase without my explicit go-ahead after I've tested the current one.
- Keep plan.md updated as you go: check off tasks, log decisions, add discovered work.
- Small commits with clear plain-language messages. The deployed app must stay in a working state — never leave main broken.
- If you're uncertain between two approaches, pick the simpler/boring one and note the tradeoff in the decisions log.
- If I ask for something mid-phase that belongs later, add it to the backlog rather than derailing the phase — and tell me you did.
- At the end of every working session, give me: what changed, how to test it, and what's next.
