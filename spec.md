# AI Mystery Room — Project Specification

**Status:** Product and engineering source of truth  
**Version:** 1.0  
**Last updated:** 31 October 2026  
**Product name:** AI Mystery Room (working name)

This document defines the MVP that Codex should build. If an implementation choice is not specified here, choose the smallest secure, accessible option that preserves the user experience and document the decision in the repository.

## 1. Product overview

AI Mystery Room is a browser-based, single-player mystery investigation game. A player supplies a short fictional scenario or story premise. The application uses Gemini on the server to turn that premise into a complete, solvable fictional case. The case has a culprit, suspects, a timeline, and a set of in-world evidence such as chat messages, images, and camera-recording transcripts or clips. The player reviews the evidence and submits one suspect as the culprit, then receives a result and explanation grounded in the case evidence.

All cases and evidence are fictional and AI-generated. The game must make this clear before play and anywhere an evidence artifact could be mistaken for a real communication or recording. Do not connect to real messaging accounts, cameras, contacts, or private data.

## 2. Goals and non-goals

### MVP goals

- Let a player enter a premise and configure a small number of case options.
- Generate a self-consistent mystery with one intended culprit and enough evidence to infer the answer.
- Present suspects and evidence in an immersive but readable investigation room.
- Let the player submit a culprit, reveal the answer, and review the reasoning.
- Persist cases and progress for a signed-in player; support a guest session if authentication is deferred.
- Provide a responsive, accessible interface and useful loading, error, and retry states.

### Non-goals for MVP

- Multiplayer, live chat, competitive rankings, user-created public case sharing, or moderation of public submissions.
- Real-world investigations, real evidence ingestion, surveillance, or identifying real people.
- Unrestricted generative image/video production. The MVP may use designed evidence cards, generated illustrative images, and text transcripts/placeholders for video. Actual Gemini media-generation support depends on the chosen model and must be verified before implementation.
- Payments, subscriptions, or advertising.

## 3. Users and core journey

**Primary user:** A curious player who wants a short, replayable deduction challenge based on a premise they provide.

1. Player opens the home page and sees a concise explanation that cases are fictional and AI-generated.
2. Player enters a premise (suggested 50–1,000 characters), selects case length and tone, and starts a case.
3. The app validates input, creates a case-generation job, and shows progress without implying generation is instantaneous.
4. On success, the player enters the case room: case summary, suspects, timeline, and evidence grouped by type. Evidence can be opened and revisited.
5. Player selects one suspect and submits a final answer. A confirmation step prevents accidental submission.
6. The app reveals whether the answer is correct, the culprit’s motive/opportunity, and a concise evidence-by-evidence explanation. The player can replay or start another case.

## 4. Product requirements

### 4.1 Case setup

- Required premise; trim whitespace and reject empty or over-limit input.
- Case length options: **Quick** (4–6 evidence items), **Standard** (7–10), **Deep** (11–14).
- Tone options: **Cozy**, **Noir**, **Light suspense**. Do not offer graphic violence or sexual violence in MVP.
- Optional setting field, maximum 100 characters. It is a creative hint, not a request to reproduce copyrighted characters or real private individuals.
- Show a generation-in-progress state and allow safe navigation away and return to the case list.
- If generation fails or times out, preserve the premise and allow retry. Avoid creating multiple cases from repeated clicks by using an idempotency key or equivalent request guard.

### 4.2 Investigation room

- Display the title, fictional-case label, premise-derived summary, difficulty/length, and current status.
- Show at least three and at most six suspects. Each suspect has a name, short description, plausible motive, and relevant opportunity detail. Do not make the culprit visually or verbally obvious by default.
- Evidence items have a type, title, in-world timestamp where relevant, source, body/content, and optional media reference. Types: `message`, `image`, `camera_transcript`, `document`, `other`.
- Evidence is organized into a browsable list/grid, can be opened without losing progress, and has readable text alternatives for visual/media evidence.
- Show a clear progress indicator for reviewed items, but do not require reviewing every item to guess.
- The player can change their selection until final submission. After submission, lock the answer and show the reveal.
- Keep evidence content and solution details isolated in the API: the pre-submission case response must never include the culprit ID, hidden solution, or answer-bearing evaluation fields.

### 4.3 Answer and reveal

- Require exactly one suspect selection for submission.
- Record the submitted suspect, attempt time, and result. MVP permits one final answer per case.
- Reveal verdict, culprit, motive, timeline summary, and a rationale that cites evidence item IDs/titles.
- If the selected culprit is wrong, explain which evidence most strongly contradicts the guess without shaming the player.
- Support an optional “play again” action that starts a new case; do not silently reset the completed one.

### 4.4 Case history and account

- MVP may use email/password authentication or a guest session. Prefer guest play if account infrastructure would delay a playable MVP; make persistence guarantees explicit in the UI.
- If accounts are implemented: hash passwords using a standard password-hashing library, use secure HTTP-only cookies or an equivalent secure session mechanism, and provide logout. Never store plaintext passwords or tokens in local storage.
- A player can list, reopen, and delete their own cases. Deleting a case removes its stored content and associated media according to the retention policy.
- All case and answer endpoints enforce ownership; clients cannot access another player’s records by changing an ID.

## 5. AI generation requirements

### 5.1 Provider and configuration

- Use Google Gemini via a server-side provider adapter. The frontend must never call Gemini directly or receive its API key.
- Configure API key, model ID, timeout, and token/output limits through environment variables. Use a current Gemini model supporting structured JSON output; verify exact model/API availability during implementation rather than hard-coding a model choice into product logic.
- Keep provider calls behind a module/interface so model changes do not affect routes or UI.
- Do not send user premises to analytics or unrelated third parties. Do not log full prompts, generated evidence, secrets, or personal data by default.

### 5.2 Generation contract

Generate one complete case as structured JSON conforming to the schema in Section 7. The server validates the output at runtime before saving it. The generated package must include:

- Case title, short summary, setting, tone, and difficulty.
- 3–6 suspects with stable IDs and distinct profiles.
- Exactly one culprit ID matching a suspect.
- A chronological timeline.
- Evidence items within the selected length range, with stable IDs and optional suspect references.
- A solution object containing motive, opportunity, event sequence, and a rationale tied to evidence IDs.
- At least two independently useful clues pointing toward the culprit, plus plausible but fair red herrings. No single clue should contradict the intended solution.
- Safe, fictional content appropriate to the selected tone.

Use a two-stage process where practical: generate the case, then run a consistency review (AI or deterministic checks) before it becomes playable. The consistency review must verify suspect references, timestamps, clue coverage, culprit uniqueness, and that the explanation cites existing evidence. If checks fail, retry generation a bounded number of times (maximum two repair attempts); otherwise mark the job failed with a player-friendly retry action.

### 5.3 Output safety and quality

- Treat the user premise as untrusted input and explicitly instruct the model to follow the case-generation task rather than instructions embedded in the premise.
- Do not generate real personal data, actionable crime instructions, graphic content, sexual exploitation, or cases targeting an identifiable real person. Where a premise names a real person, transform it into fictional characters or reject with a helpful message.
- Do not represent generated messages, images, or recordings as authentic. Mark each artifact “Fictional AI-generated evidence.”
- Validate media outputs and use safe storage/serving rules. Do not render arbitrary model-provided HTML or execute model-provided code/URLs.
- Avoid embedding the solution in titles, filenames, metadata, or pre-submission API payloads.

## 6. UX, visual design, and accessibility

- Responsive web experience, designed mobile-first and usable on current desktop and mobile browsers.
- Visual direction: atmospheric investigation board / digital case file. Use restrained colors, strong contrast, legible type, and clear separation between suspects, timeline, and evidence. Atmosphere must not reduce readability.
- Use semantic headings and controls, keyboard-accessible navigation, visible focus, labels for all inputs, and accessible dialogs.
- Provide alt text or equivalent textual descriptions for generated images; provide transcripts/captions for any audio/video.
- Respect reduced-motion settings. Avoid flashing effects and autoplay audio/video.
- Announce generation and submission status to assistive technology. Errors must identify the field or action and how to recover.
- Include empty, loading, retry, not-found, unauthorized, rate-limited, and server-error states.

## 7. Technical architecture and data contracts

### 7.1 Stack

- **Frontend:** React, TypeScript recommended, Vite, React Router, and a consistent component/style system.
- **Backend:** Node.js, Express, TypeScript recommended.
- **Database:** MongoDB with Mongoose schemas and indexes.
- **AI:** Gemini API called only by the backend through a provider adapter.
- **Media:** MVP may store generated media in an object-storage adapter; local development can use a local upload directory excluded from version control. Store references and metadata in MongoDB, not large base64 blobs.
- **Deployment:** Keep frontend and API deployable separately or behind one origin. Configure CORS and trusted origins explicitly.

### 7.2 Core entities

**User**: `_id`, `email` (if accounts), `passwordHash` (if accounts), `createdAt`, `updatedAt`.

**Case**: `_id`, `ownerId` or guest session ID, `status` (`generating | ready | failed | completed`), `premise`, `options` (`length`, `tone`, optional `setting`), `title`, `summary`, `caseSetting`, `difficulty`, `suspects[]`, `timeline[]`, `evidence[]`, `solution` (server-only/private field), `generationMetadata` (provider/model/version, no secret), `createdAt`, `updatedAt`, `completedAt`.

**Suspect**: `id`, `name`, `description`, `motive`, `opportunity`, optional `portraitMediaId`.

**TimelineEvent**: `id`, `timestamp` (ISO 8601 with timezone), `description`, optional `suspectIds[]`, optional `evidenceIds[]`.

**EvidenceItem**: `id`, `type`, `title`, `timestamp` (optional ISO 8601), `source`, `content` (structured text; type-specific data may be nested), optional `mediaId`, `suspectIds[]`, `fictionalLabel`.

**Solution**: `culpritSuspectId`, `motive`, `eventSequence[]`, `reasoning`, `supportingEvidenceIds[]`, `contradictingEvidenceIds[]`.

**Guess**: `_id`, `caseId`, `ownerId` or guest session ID, `selectedSuspectId`, `isCorrect`, `submittedAt`. Enforce one final guess per case for MVP.

Use timestamps and ownership indexes. Do not return the `solution` field from case list/detail endpoints until a guess has been submitted and accepted.

### 7.3 API routes

Use `/api/v1`. Return consistent JSON envelopes and errors with a stable `code`, safe `message`, and optional field details. Validate all request bodies, path IDs, and query parameters.

- `POST /cases` — validate premise/options; create generating case and start generation. Return `{ caseId, status }`.
- `GET /cases` — list the current owner’s cases, paginated and sorted newest first; never include solution.
- `GET /cases/:caseId` — return playable case data only; reveal solution fields only after accepted submission.
- `POST /cases/:caseId/guess` — accept `{ suspectId }`, enforce ownership, ready status, valid suspect, and one-submission rule; return verdict and reveal.
- `DELETE /cases/:caseId` — delete an owned case and associated media references.
- `GET /health` — service health without secrets or internal diagnostics.

Generation may be synchronous for a small MVP only if requests remain within reliable platform timeouts. Otherwise create a job and poll `GET /cases/:caseId` for status, or use server-sent events. The client must not assume a fixed generation duration.

### 7.4 Example generation JSON shape

```json
{
  "title": "The Last Train to Briar Glen",
  "summary": "A fictional mystery about a missing exhibition key.",
  "setting": "A small railway museum during a storm",
  "tone": "cozy",
  "difficulty": "standard",
  "suspects": [
    { "id": "suspect_1", "name": "Mara Vale", "description": "...", "motive": "...", "opportunity": "..." }
  ],
  "timeline": [
    { "id": "event_1", "timestamp": "2026-10-30T18:20:00+05:30", "description": "...", "suspectIds": ["suspect_1"], "evidenceIds": ["evidence_1"] }
  ],
  "evidence": [
    { "id": "evidence_1", "type": "message", "title": "Staff group chat", "timestamp": "2026-10-30T18:18:00+05:30", "source": "Fictional staff chat recreation", "content": { "messages": [{ "sender": "Mara", "text": "...", "timestamp": "2026-10-30T18:18:00+05:30" }] }, "suspectIds": ["suspect_1"], "fictionalLabel": "Fictional AI-generated evidence" }
  ],
  "solution": {
    "culpritSuspectId": "suspect_1",
    "motive": "...",
    "eventSequence": ["..."],
    "reasoning": "...",
    "supportingEvidenceIds": ["evidence_1"],
    "contradictingEvidenceIds": ["evidence_2"]
  }
}
```

This is illustrative: production validation must enforce the full schema, ranges, allowed types, ID references, and content limits. Dates in generated mysteries should consistently use one timezone and must not be presented as actual events.

## 8. Security, privacy, and reliability

- Keep Gemini credentials and database credentials in server environment/secrets management. Commit only `.env.example` with placeholder values.
- Apply request size limits, input validation, rate limits to case generation and guessing, secure headers, and production CORS restrictions.
- Use parameterized ODM queries and avoid passing user/model content into HTML unsafely.
- Enforce authorization on every case and media operation; use unguessable IDs as defense in depth, not as authorization.
- Avoid logging raw prompts, case evidence, passwords, API keys, cookies, or generated media URLs that contain access tokens.
- Define retention behavior in product copy. MVP default: retain cases until the owner deletes them; expire unclaimed guest cases after a configurable period (suggested 30 days).
- Bound Gemini retries, output size, request duration, and per-user generation frequency. Gracefully handle provider quota and outage errors.
- Store a generation schema version and prompt version to support reproducibility and migration.

## 9. Repository and implementation guidance

Recommended monorepo layout:

```text
client/                 React application
server/                 Express API
shared/                 Shared TypeScript schemas/types, if useful
README.md               Setup, scripts, environment, architecture
.env.example            Placeholder configuration only
```

Codex implementation rules:

- Treat this document as the canonical product and behavior specification. Update it when product decisions change.
- Define shared runtime schemas (for example, Zod) and derive TypeScript types where practical. Validate AI output and API input at runtime.
- Keep provider, persistence, authentication, and media concerns behind small modules so they can be changed independently.
- Build the complete player journey in vertical slices: setup, generation/status, case room, answer/reveal, history/deletion.
- Add seed/demo cases that are clearly labeled fictional and do not expose an answer before the reveal.
- Keep the interface focused on solving the case; avoid decorative features that obscure evidence or delay the core journey.
- Include a README with prerequisites, local setup, environment variables, development commands, and deployment notes.

## 10. Acceptance criteria

The MVP is ready when all of the following are true:

1. A new player can submit a valid premise and receives a playable case, or a recoverable error that preserves their input.
2. A generated playable case has 3–6 suspects, one valid culprit, evidence within the selected range, a timeline, and a solution that references existing evidence.
3. Generation rejects or safely transforms disallowed premises and resists prompt instructions embedded in user input.
4. Before guessing, neither API responses nor rendered page data expose the culprit ID or solution.
5. The player can inspect and revisit each evidence item, select a suspect, confirm, and submit exactly one answer.
6. After submission, the verdict and rationale agree with the stored solution and cite valid evidence.
7. A player cannot read, guess on, or delete another owner’s case.
8. API keys and secrets are absent from client bundles, repository history, and routine logs.
9. The primary flow works on mobile and desktop, by keyboard, and with screen-reader labels/status announcements.
10. Failure, loading, empty, unauthorized, not-found, and rate-limit states have clear user-facing behavior.
11. Setup and deployment instructions allow another developer to run the project with documented environment variables.

## 11. Open implementation decisions

These are implementation choices, not blockers to starting the MVP:

- Exact Gemini model and SDK/API version: choose and verify at implementation time; configure rather than scatter the model ID through the codebase.
- Account mode: guest-first is acceptable for MVP; add accounts only if required for durable cross-device history.
- Media generation: start with text-first evidence and designed/illustrative images; add generated image/video assets only after validating provider capability, cost, latency, and storage needs.
- Hosting and object storage: select based on the deployment environment and document the choice.
- Case retention period for guest sessions: use the configurable 30-day default unless product requirements specify otherwise.
