---
name: linkedin-life-story
description: Find out who a specific real, living person is and write their life story — a cited narrative of how they got here, plus a timeline, a confidence table and a source list. Trigger on the intent, not the format. The user might paste a linkedin.com/in/ link, or give a name plus an employer, a city or a field, or just name someone they keep hearing about and don't understand — all of these count, and a missing LinkedIn URL is never a reason not to use this. Covers "who is this person", "what's this guy's story", "give me their background", "their career arc / origin story / track record", "profile / dossier / bio on X", and "dig into them before my call, my podcast, my board meeting, my investment". Fans out parallel searches across news, company records, podcasts, talks and their own writing; gates every fact against an identity fingerprint so a same-name stranger's life never leaks in; chases each claim back to the source that supposedly proves it instead of repeating it; and says plainly when someone has no public footprint. Public sources only — never scrapes private data, never contacts the subject, never invents a life. Not for editing the user's own LinkedIn profile, screening a resume or CV file, exporting LinkedIn data, researching a company rather than a person, or writing up a long-dead historical figure.
argument-hint: "<linkedin-url-or-name> [--quick|--deep]"
license: MIT
allowed-tools:
  - WebSearch
  - WebFetch
  - Read
  - Write
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
---

# LinkedIn Life Story

Take one LinkedIn profile. Return a life story a stranger could read and actually *understand the
person* — every claim sourced, every guess labelled as a guess, and the empty spaces left honestly
empty.

## 0. First, locate the bundled files

This skill ships reference docs alongside `SKILL.md`. Resolve their directory **once**, at the
start of the run, and reuse the absolute path it prints:

```bash
dirname "$(find ~/.claude -name SKILL.md -path '*linkedin-life-story*' \
  -not -path '*/marketplaces/*' 2>/dev/null | sort -V | tail -1)"
```

Everything below writes this path as `<SKILL>`. Substitute the real absolute path.

- Read `<SKILL>/references/search-playbook.md` before you fan out. It holds the query patterns per
  lane and the tricks for the sources that block you.
- Read `<SKILL>/references/report-template.md` before you write. It holds the exact output shape.

---

## The engine: the profile is the index, not the story

The instinct when handed a name is to search for it and write up what comes back. For a famous
person that works. For everyone else it returns a LinkedIn page, a Crunchbase stub, three
namesakes and a funeral notice, and you are tempted to fill the silence with plausible prose.
That is how this task fails.

Two facts reshape the whole method:

**Nobody writes a biography of a normal person — but people narrate themselves constantly.** No
journalist has profiled a VP of Engineering at a mid-size logistics firm. But that VP told their
origin story on a niche podcast in 2022, wrote a "why I left banking" post in 2019, gave a
conference talk with a personal opening anecdote, and answered "how did you get started" in an
alumni magazine. The life story is *out there in their own voice*, scattered across low-authority
sources that a search for "[Name] biography" will never surface. **So don't search for a biography.
Hunt for the subject narrating themselves.** This is the single highest-yield move in the skill and
the one a naive run always misses.

**LinkedIn gives you dated rows; the story lives in the seams between them.** The résumé says
`Goldman Sachs, 2015–2019` then `Founder, bakery, 2020–`. The rows are not the story. The *joint*
is the story — and the profile is deliberately silent about it, because a résumé is a document
written to remove exactly the friction you are looking for. Every gap, every sideways move, every
downshift in title, every industry jump is a question the profile refuses to answer. Go and find
the answer.

> **fingerprint the identity → map the anchors and the seams → hunt their own voice → gate every
> find against the fingerprint → narrate only what survives**

---

## Workflow

### 1. Get the profile in front of you

**Follow the redirect.** `www.linkedin.com/in/<slug>` 301s to a **country subdomain** —
`my.linkedin.com`, `pk.linkedin.com`, `ae.linkedin.com`, `uk.linkedin.com` — and *that* host often
serves a public version of the profile where `www` gives you nothing. WebFetch hands the redirect
back to you rather than following it; call it again on the country URL. This works often enough to
always be worth the one extra call.

If you get **HTTP 999**, that's LinkedIn's bot-block on `/in/` profiles. Don't retry and don't try
other subdomains — you'll just get 999 again.

But don't give up on LinkedIn yet, because **`/posts/` URLs are not blocked the way `/in/` is.**
Search for `site:linkedin.com/posts "<Name>"` and fetch the results directly. In testing, this
recovered a subject's own posts — including the one announcing his departure from a company — after
the profile itself returned 999, and it was the single move that salvaged the run. Their posts are
their own voice and self-narration, which is exactly the material this whole skill is hunting for.
A blocked profile is not a blocked person.

Then ask for the paste anyway.

Ask for the paste plainly, without apology, because it is the normal path and not a malfunction:

> LinkedIn blocks automated reads, as it does for everyone. Open the profile, select all (Cmd-A),
> and paste it here — the whole dump, formatting mangled is fine. I'll take it from there.

#### The partial-fetch trap — read this even when the fetch "worked"

A country-subdomain fetch returns a **fragment** of the profile, and the fragment does not announce
itself as one. Expect a truncated About section, missing dates, and — the dangerous part — **an
incomplete and badly-ordered role list.**

This is not a minor degradation. In testing, a fetched profile showed only student-era roles ending
eight years earlier and no current position. The subject had in fact co-founded a national campaign
that amended his country's constitution, and he has a Wikipedia page. None of it was in the fetch. A
run that trusted that fragment would have written a confident, fluent life story about a former
student debater who stopped doing things eight years ago — and nothing in the output would have
looked wrong.

So a fetched profile is a **starting hypothesis, never an anchor list**:

- **Absence in a fetch is not absence in life.** You may never conclude "they have no current role",
  "they did nothing after 2017", or "their career stalled" from a fetch. You have not seen the
  profile; you have seen a piece of it.
- **Before locking your anchors, sanity-check the person against the open web** — one search on
  name + strongest fingerprint token. If they have a Wikipedia page, a Forbes listing or national
  press, you need to know that in minute one, not discover it in the write-up.
- **When the fetch and the world disagree, the world wins**, and the disagreement itself is worth a
  line in the report.
- **Prefer the paste whenever you can get it.** It carries the full About text, complete role
  descriptions, education dates, volunteering and recommendations. If a fetch looks thin or the
  sanity-check turns up someone more significant than the fragment suggests, go back and ask for the
  paste rather than proceeding on a fragment.

**The About section, in the subject's own words, is the most valuable paragraph you will be given** —
it tells you how they narrate themselves, which is the frame the whole report tests against. A
truncated About is a real loss; say so if that's all you got.

If the user gives you only a name and a company, that's workable. Say you're proceeding without the
profile and that the identity gate will be looser as a result.

### 2. Build the identity fingerprint — before you search for anything

This is a safety gate, and skipping it is how a report ends up containing another human being's
life. Common names are the norm, not the exception.

From the profile, extract and write down:

- Full name and any former names, spelling variants, or non-Latin renderings
- Every employer, with dates
- Every school, with dates and degrees
- Current and past locations
- Distinctive tokens: an unusual title, a named product, a company handle, a co-founder's name
- Approximate age band (infer from first degree, and mark it inferred)

This is your fingerprint. **A search result may only enter the report if it can be tied to at least
one fingerprint element.** A 2018 article about "Sarah Chen, product manager" is worthless unless it
also says Stripe, or Michigan, or Lagos. Sarah Chen is thousands of people.

### 3. Map the anchors and the seams

List the **anchors**: the dated, checkable rows — jobs, schools, funding rounds, publications,
awards. These are what you'll verify.

Then list the **seams**: the joins between anchors, which are where the story actually is. For each
one, write the question it poses.

```
2015–2019  Goldman Sachs, VP
   ↳ SEAM: left at 28 after a promotion. Why? → search departure, layoffs at desk, any interview
2019–2020  [nothing on the profile]
   ↳ SEAM: 14-month hole. → the loudest thing on this page. Illness, caregiving, travel, a failed
     venture, a firing? Search hard, and if nothing surfaces, say nothing.
2020–      Founder, bakery
   ↳ SEAM: banking → bread is a huge leap. Someone has asked them about this on a podcast. Find it.
```

Carry these questions into the fan-out. A researcher hunting a specific question finds things a
researcher "gathering background" never will.

**One discipline about gaps:** a hole in a résumé is a question, never a conclusion. It is a
sabbatical, a sick parent, a baby, a burnout, a startup that died quietly, a visa. If the answer
isn't public, the honest report says *"the profile shows no role between 2019 and 2020; no public
source explains the interval"* — and then stops. It does not speculate, and it does not imply.

### 4. Fan out — parallel lanes

Read `<SKILL>/references/search-playbook.md` now for the query patterns.

Spawn the lanes as subagents. Each gets the fingerprint, the seam questions, and the instruction to
return only fingerprint-matched findings with URLs. Lanes are blind to each other on purpose — they
surface different material.

| Lane | Hunting for |
|---|---|
| **Own voice** | Podcasts, conference talks, YouTube, webinars, AMAs, interviews. Where they tell their own story. **This is the lane that makes the report, not a supporting act** — see the sequencing note below. |
| **Core web & press** | News, trade press, funding announcements, company About pages, official bios, speaker pages, award listings, Crunchbase. |
| **Self-published** | Their own writing: Substack, Medium, personal site, GitHub, X/Twitter, public Instagram, newsletters, Quora, forum posts. |
| **Records** *(conditional)* | Only if the anchors mark them as an academic, researcher, scientist, clinician or inventor — then Google Scholar, patents and papers *are* their biography and this lane becomes primary. Skip it entirely for an operator or a marketer; it will return nothing and cost you ten minutes. |

#### How to run the lanes — this part has a wrong way that feels right

**Issue every lane as an Agent call in a single message.** They then run concurrently and their
results come back to you together. Do not spawn them one at a time.

**Then do nothing until they return.** No searching, no fetching, no "getting a head start" — and
above all, no `sleep` to pass the time. This instruction exists because runs that ignored it all
made the same two mistakes: they re-fetched pages their own lanes were already fetching (pure waste,
and it is what makes this skill slow), and they began drafting early.

**Do not draft until the own-voice lane has reported.** Drafting early is worse than idling. The
own-voice lane is routinely the last to finish and the most valuable when it lands: it turns your
careful inference into the subject's own sentence. A run that drafts without it produces a report
built on reasoning where it could have been built on a quote, and then has to tear it up. Wait.

X/Twitter and Instagram will often refuse you. That's expected: reach them through search-engine
indexes, public mirrors and quoted screenshots, and when a source is unreachable, record it as
unreachable rather than dropping it silently.

### 5. Verify — assume the record is lying to you

This is where the value of the whole exercise concentrates, and it is the step a confident
researcher skips. **Every trap below was hit in real runs of this skill.** They are not
hypotheticals.

**A search snippet is not evidence.** Open the page. Snippets are where hallucinated citations are
born: the URL is real, the content is not what the summary said it was. A run once had a snippet hand
it the occupations of its subject's parents — intimate, humanising, exactly the texture a life story
wants, and present on no page that could be opened. It was another man's family. Had it gone in, no
reader could ever have caught it.

**Open the citation behind the claim, not just the page making it.** A Wikipedia article said the
subject was arrested at a named vigil on a named date. The article the sentence cited reported 31
arrests and *did not name him*. The claim was not false, exactly — it was unestablished, and every
downstream summary had been repeating it as fact. When a source cites another source for a load-
bearing claim, follow it. Circular sourcing is how a rumour becomes a fact.

**Ask who is actually a witness.** Ten articles about a funding round are not ten sources if all ten
are the same wire release, rewritten. A company's own About page is not a neutral witness about its
own executive. A subject's own bio inflates titles and awards — and this shows up constantly: a
"CMO" title that independent sources render as a much narrower role, a magazine someone is credited
with founding whose own published history names other people. Check the claim against someone with
no stake in it, and if nobody independent exists, say so.

**Separate the man from the organisation.** "We sued the government" is often an organisational
*we*. A movement's collective wins are not personal ones, and a co-founder is not automatically a
plaintiff, an author, or an architect. This distinction is invisible in press coverage and it is
usually the difference between an accurate life story and a flattering one.

**Watch for the same-name mis-merge.** Aggregators and org charts silently blend two careers into
one. A row that fits your subject's story a bit too neatly — the job that conveniently explains how
they met their co-founder — deserves more scrutiny than one that doesn't, not less.

**Interrogate the headline number.** The most-repeated figure in someone's story is reliably the
least-checked: the users they grew, the people they reached, the money they raised, the voters they
enfranchised. It appears in every bio *including their own*, which reads as corroboration and is
actually just one claim echoing. Runs of this skill have deflated six inflated claims about a subject
in a dedicated table and then asserted his headline figure at the highest confidence tier without
blinking — because it was so familiar it never looked like a claim at all. Go find where the number
comes from. It is often a broader total quietly credited to a narrower cause.

When a source and the world disagree, say so in the report. A claim you *disproved* is one of the
most useful things you can hand back, because it's the one the reader would otherwise have repeated.

### 6. Gate, then tier

Every candidate fact clears two checks before it can be written.

**Gate 1 — is this the right human?** Tie it to the fingerprint. If it cannot be tied, it does not
go in.

*If a name collision is live and material — two plausible people, and the difference changes the
story — stop and ask the user.* Show both candidates with their distinguishing details and let them
pick. A wrong attribution is not a small error; it is a different person's life in your subject's
report, and the user has no way to detect it downstream. Interrupting is cheap. Being wrong is not.

**Gate 2 — how solid is it?** Tag each surviving fact:

- **Confirmed** — an independent source you opened and read says it. Two beats one.
- **Probable** — a single credible source, or the subject's own claim corroborated by circumstance.
- **Self-reported** — only the subject says it (their LinkedIn, their bio, their own post). Perfectly
  usable, but the reader must know the subject is the only witness. Résumés are marketing.
- **Inferred** — you reasoned it from other facts. Must be visibly marked as reasoning *in the
  narrative itself*, not just in a table.
- **Unverified** — plausible, uncorroborated, kept only because it's load-bearing. Say so.

Anything that clears neither gate is **deleted, not softened**. "Reportedly", "it seems", "sources
suggest" are the vocabulary of a report quietly making things up.

### 7. Write the story

Read `<SKILL>/references/report-template.md` for the exact shape.

#### Length: the appendix is where research goes, the narrative is where the story goes

**Narrative: 1,500–2,500 words. This is a ceiling, not a quota, and it is routinely violated.**

Every early run of this skill blew through it — one by 77% — and none of them did so because the
story needed the room. They did it because they had *found* the material and couldn't bear to leave
it out. That instinct is sunk cost wearing the costume of thoroughness, and the reader pays for it.

The discipline that actually works is structural: **the appendix already exists to hold everything
you found.** A fact you're proud of but that doesn't move the story goes in the timeline or the
confidence table, where a reader who wants it will find it, and where it costs the narrative
nothing. Nothing is lost by cutting it from the prose. That is what the appendix is *for*.

So before you save, count the narrative. If it's over 2,500 words, you are not finished — you have
a draft. Cut it. The test for each paragraph: *does this change how the reader understands who this
person is?* If it only proves you did the work, it belongs in the appendix.

A tight 1,800-word story with a 60-source appendix is the shape to aim for. A 4,000-word story is a
research dump wearing a narrative's clothes, and it buries the very findings you worked for.

#### The opening summary is where uncited claims hide

The "short version" at the top may run without inline citations — it is a recap, and peppering it
with brackets makes it unreadable. That licence comes with one condition, and it gets broken often
enough to call out: **nothing may appear in the summary that is not established, and cited, below.**

The failure is subtle and it has happened in every early run. Because the summary *feels* like
recap, the discipline that governs the body relaxes — and a fact slips in that exists nowhere else
in the document: a neighbourhood, a date, a quote with no source. It reads as a confident distillation
of research you did, when it's the one sentence you didn't do it for.

There is a second, subtler leak in the same place, and it survives even a careful check. A claim can
be perfectly cited in the body, tiered honestly as **Self-reported**, footnoted with *"no independent
confirmation exists"* — and then appear in the summary as a flat statement of fact. Nothing was
fabricated and no rule was broken, yet the reader of the summary now believes something the report
itself does not. A real run did exactly this: it established that every source for its subject's
university degree traced back to the subject himself, said so plainly in the appendix, and still
opened by stating the degree as fact.

The same run, once told to check, also cut *"he was 26"* from its own summary — an innocuous,
specific, confident-looking number that turned out to be arithmetic off a birth year sourced to an
uncited encyclopedia infobox. That is the shape of the thing: not a wild invention, just a small
certain-sounding fact resting on nothing.

So the rule has two halves. Before saving, re-read the summary against the body, claim by claim:

- **Nothing in the summary may be absent from the body.** Cut it, or source it.
- **Nothing in the summary may be stated more confidently than the body tiers it.** If the body says
  Self-reported, the summary says *"by his own account"*. If the body says Unverified, the summary
  either hedges or stays silent. The summary is allowed to be shorter than the truth; it is never
  allowed to be surer than it.

#### Two things separate a life story from a résumé read aloud

**Find the through-line.** A person is not a list of jobs. There is usually one thread — a
preoccupation, a wound, a bet they keep re-making, a place they keep returning to. Somebody who
went biology → consulting → biotech founder isn't a wanderer; they took a nine-year detour and came
back. Name the thread. If the evidence genuinely doesn't support one, say the record doesn't show a
clear arc — that is itself a finding, and far better than a false arc imposed on a real person.

**Let them talk.** Quote the subject directly wherever you have their words. A single line from a
podcast — *"I only started the bakery because I couldn't sleep"* — does more work than a paragraph
of your inference, and it cannot be accused of being invented, because they said it.

And the rule under everything:

> **Every sentence in the narrative traces to a source or is visibly marked as inference.**

Inline citations `[1]`, `[2]` map to the appendix source list. If you catch yourself writing a
sentence with no number after it and no hedge in front of it, that sentence is fiction. Delete it.
The most seductive failure of this skill is the well-turned paragraph about a childhood you invented
because the arc needed one.

### 8. Save and hand back

Write to `~/research/people/<firstname-lastname>.md`, creating the directory if needed. Print the
absolute path, then give the user the short version in chat: who this person is, the one thing that
surprised you, the biggest hole in the record, and **any claim in circulation that you disproved** —
that last one is often the most useful sentence you will write, because it's the thing they would
otherwise have gone on believing.

---

## When the subject is thin

Most people are. A mid-level manager at a normal company has no press, no podcast, no writing —
their entire public existence is the profile you were handed.

**Do not pad.** The failure mode is a beautifully written 2,000-word story about someone the sources
barely mention, and it is worse than useless: it looks exactly like the good version, so the user
cannot tell it apart. Length must track evidence, always.

Write the honest thin report:

- What the record actually confirms (usually: their employers, their school, their titles)
- What the profile claims but nothing corroborates — flagged **Self-reported**
- What is genuinely absent, and *why* you think it's absent: a private person who doesn't post; a
  name so common that everything is unattributable; a career in a field that generates no public
  record; a footprint in a language or platform you couldn't reach
- The seam questions you could not answer

Say it plainly, up top: **"This person has a minimal public footprint. Here is everything that is
actually verifiable, and here is what I could not establish."** Four hundred honest words beat two
thousand invented ones, and a user who learns *"there is nothing out there on this person"* has
learned something true and useful.

The absence is the finding. Report it as one.

---

## Rails

You are researching a real human being who did not ask to be researched. That the material is
public does not make every use of it fair.

- **Public sources only.** Things the subject or a publisher put into the open web. No data brokers,
  no leaked datasets, no people-search or background-check sites, no scraped contact databases, no
  paywall, login, CAPTCHA or robots bypass. A 403 is an answer.
- **No contact enrichment.** Do not hunt for personal email, phone, or home address. Not the goal
  and not on.
- **Professional record, plus the personal colour they made public themselves.** Their marathon,
  their hometown, the band they post about — fine, they published it. Their health, their finances,
  their family members who never chose to be public, their religion, their sexuality, their politics
  — out, unless the subject has made it publicly and deliberately part of their own story, and even
  then only where it bears on the story you're telling.
- **Controversies are in scope, handled like a journalist would.** A lawsuit, a critical piece, a
  departure under a cloud: include it if credible public sources report it, cite them, name the
  outcome if there was one, and include the subject's response or note that they didn't give one.
  Never launder an allegation into a fact. Never omit one because it's awkward.
- **Never contact, connect, follow, message or comment.** Research is silent. This applies to every
  tool you have, Bash and MCP servers included.
- **Never infer protected traits.** Not from a name, not from a photo, not from a school.
- **If the subject looks like a genuinely private individual** — no public role, no public writing,
  nothing that invites attention — hold a higher bar. Ask the user what they're using this for
  before going deep. Most of the time the answer is fine. Occasionally it won't be, and it costs one
  question to find out.

## Quality bar

- Every factual sentence carries a citation or a visible hedge. No exceptions, and no drift late in
  the document, where discipline always slips and invention creeps in.
- Length tracks evidence, and the narrative stays under 2,500 words. A padded report is a failed
  report even if it reads beautifully — *especially* if it reads beautifully, because then nobody
  can tell it apart from the real thing.
- Their own words beat your prose. Quote them wherever you can.
- The seams get answered, or their unanswerability gets stated.
- Claims you disproved are reported, not silently dropped. They're often the most valuable output.
- What you couldn't reach is named in the report, not hidden.
- A reader should finish it able to say *"now I understand how they got here"* — and able to check
  every load-bearing claim you made.

## The one failure that matters

Everything above serves a single end. This skill is pointed at real, named, living people, and its
output will be believed. A fluent, confident, well-structured report about a life that was partly
invented is indistinguishable — to the reader, and often to you — from a true one. That is the
failure to fear. Not a thin report. Not a slow one. Not one that says *"I could not establish this."*

Prefer the honest gap, every time.
