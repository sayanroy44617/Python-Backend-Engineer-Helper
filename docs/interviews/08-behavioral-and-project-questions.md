# Interview Prep: Behavioral / Project Questions

## Why this section is different

Every other page in this handbook reviews technical material you already
know from the corresponding section. Behavioral and project questions
have no equivalent "correct answer" to review — they evaluate how
clearly you can communicate what you actually did, why you made specific
decisions, and what you learned. This page is a framework and a set of
categories to prepare *your own* answers against, not content to
memorize.

## The STAR method

Structure answers to behavioral questions as:

- **Situation** — brief context: what was the project/team/problem?
- **Task** — what was your specific responsibility or goal?
- **Action** — what did *you* specifically do? (Not "we" — be explicit
  about your individual contribution, even within a team effort.)
- **Result** — what happened? Quantify it if at all possible (latency
  reduced by X%, incident resolved in Y minutes, adoption by Z teams).

Keep each answer to roughly 1-2 minutes. A common failure mode is
spending 80% of the time on Situation/Task and rushing Action/Result —
the Action and Result are almost always what the interviewer actually
wants to hear.

## Common question categories

### Ownership and impact

> "Tell me about a project you're proud of."
> "Describe something you built end-to-end."

Prepare 2-3 stories in advance covering different angles (a technical
deep-dive, a cross-team collaboration, an ownership/initiative story).
Reusing the same one or two stories across multiple questions (adapted
slightly) is normal and expected — you don't need a unique story for
every possible question.

### Failure and mistakes

> "Tell me about a time you made a mistake."
> "Describe a project that didn't go well."

Pick a *real* mistake with genuine consequences, not a disguised
humblebrag ("I worked too hard"). The strongest answers focus on what
you learned and changed afterward (a new habit, a new safeguard you now
build in) — the interviewer is evaluating your self-awareness and growth
more than judging the original mistake itself.

### Conflict and disagreement

> "Tell me about a time you disagreed with a teammate/manager."

Show that you engaged with the disagreement directly and professionally
— what was the actual technical/process disagreement, how did you make
your case, and how was it resolved (even if you didn't get your way).
Avoid framing this as "I was right and they eventually agreed" every
time; showing you updated your own view based on new information is
often a stronger signal.

### Ambiguity and prioritization

> "Tell me about a time requirements were unclear."
> "How do you prioritize when everything feels urgent?"

Demonstrate a concrete process: how you sought clarification, what
assumptions you made explicit and validated, and how you decided what
to build first. Vague answers ("I just used my judgment") are weaker
than a specific example with a specific reasoning process.

### Learning and growth

> "How do you stay current technically?"
> "Tell me about something you had to learn quickly."

This handbook itself is a legitimate, honest answer to both — building
a structured, curriculum-driven personal reference (with a deliberate
foundation/intermediate/advanced progression, covering Python through
system design and DevOps) demonstrates exactly the kind of self-directed
technical growth this question is probing for. Be ready to talk
concretely about *why* you structured your learning this way and what
it changed about how you approach a specific technical area.

## Discussing your own projects

- **Lead with the problem, not the tech stack.** "We had a workflow that
  took 3 hours manually" is a stronger opener than "I used FastAPI and
  PostgreSQL."
- **Be ready to go deep on any technical claim you make.** If you say
  "I optimized a slow query," expect (and prepare for) a follow-up:
  *which* query, *what* was slow about it, *how* did you diagnose it
  (see [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md)
  for the vocabulary), and *what specifically* did you change.
- **Own trade-offs explicitly.** "I chose X over Y because of Z" is far
  stronger than presenting your past decisions as if there were no
  alternative — this is the same trade-off-articulation skill tested in
  [System Design](06-system-design.md) and
  [API Design](05-api-design.md) interviews, just applied to your own
  real work instead of a hypothetical.
- **Quantify impact wherever possible** — "reduced page load time from
  2.1s to 400ms" is more convincing than "made it faster."

## Common mistakes

- Rambling without structure — STAR exists specifically to keep an
  answer tight and complete; skipping straight to a long, unstructured
  narrative loses the interviewer partway through.
- Saying "we" for the entire answer, leaving the interviewer unable to
  tell what *you* actually contributed versus the rest of the team.
- Picking a conflict/failure story with no real stakes or resolution,
  which reads as evasive rather than reflective.
- Not preparing at all, assuming behavioral questions are "easy" and
  can be answered off the cuff — vague, rambling answers are one of the
  most common reasons an otherwise technically strong candidate doesn't
  advance.
- Contradicting your resume/project descriptions under follow-up
  questions — only claim depth you can actually defend if probed.

## A short prep checklist

- [ ] 2-3 prepared stories covering ownership/impact, failure, and
      conflict, each structured with STAR.
- [ ] For each story, know the specific numbers/outcomes, not just a
      qualitative description.
- [ ] Practice saying each story out loud in under 2 minutes.
- [ ] Prepare 2-3 thoughtful questions to ask the interviewer at the end
      — about the team's technical challenges, on-call practices, or
      how they approach the topics covered in
      [System Design](06-system-design.md)/[DevOps](07-devops.md) —
      genuine curiosity here reads better than generic questions about
      "company culture."
