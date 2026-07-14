# linkedin-life-story

A [Claude Code](https://claude.com/claude-code) skill. Give it a LinkedIn URL — or just a name and
where someone works — and it returns a **cited life story**: a long-form narrative of who they are
and how they got here, plus a timeline, a confidence table, and every source it used.

```
> https://www.linkedin.com/in/someone/ — who is this guy? give me his life story
```

It is built around one uncomfortable fact: **"write me this person's life story" is a request that
invites fabrication.** Point a language model at a real human with a thin public record and ask for
a narrative, and it will produce one — fluent, plausible, and partly invented. The invented version
is indistinguishable from the real one, which is what makes it dangerous. Everything in this skill
exists to stop that.

---

## What it actually does differently

Anyone can search a name. The value here is that the skill **refuses to believe the record.** In
testing against a no-skill baseline, on the same three real people:

- The baseline stated flatly that its subject had been **arrested at a protest**, graded the claim
  *High confidence*, and embellished it with detail no source contains. It had taken the claim from
  an encyclopedia page; the article that page cited reports the arrests and **does not name him**.
  The skill chased the citation, found he wasn't in it, and marked the claim **Unverified** —
  reporting only what independent monitors actually confirmed. A person's arrest record is not a
  detail to get wrong.
- Where the record went silent on a subject's school years, the baseline filled it — inventing an
  exam result and a warm paragraph about what that school must have been like for a boy from his
  background. The skill wrote, of the same period: *"What actually happened to him at school is not
  something I can tell you. It is the single largest hole in this report."*
- A search snippet supplied a subject's **parents' occupations** — intimate, story-shaped, and
  impossible for a reader to disprove. It belonged to a different man. The skill found it appeared on
  no page it could open, and killed it.
- On a third subject, the claims that failed checking mostly traced back to **the subject's own
  bio** — the awards, titles and figures a person writes about themselves and everyone downstream
  repeats. Self-description is not evidence, and the skill tiers it as *Self-reported* no matter how
  many blogs have echoed it.
- It also cut **"he was 26"** from its own opening summary — an innocuous, specific,
  confident-looking number that turned out to be arithmetic off a birth year sourced to an uncited
  encyclopedia infobox. Not a wild invention; just a small certain-sounding fact resting on nothing.

None of that comes from searching harder. It comes from a verification stage that assumes the record
is lying.

## The three ideas it's built on

**The profile is the index, not the story.** A résumé is a document engineered to remove exactly the
friction you're looking for. LinkedIn gives you dated rows; the story lives in the **seams between
them** — the 14-month gap, the sideways move, the leap from banking to bread. The skill maps each
seam as an explicit question and sends researchers to answer *that*, because an agent hunting a
specific question finds things an agent "gathering background" never will.

**Hunt the subject narrating themselves.** Nobody wrote a biography of a VP of Engineering at a
logistics firm — but that VP told their origin story on a niche podcast, and the host opened with
"so how did you get here?" That's the highest-yield source and the one a naive run misses entirely,
because it searched for "[Name] biography" and got a funeral notice. Podcasts, conference bios,
alumni profiles, and `"<Name>" "growing up"` as a literal query.

**Two hard gates before any fact is written.** An *identity fingerprint* (employers, schools, dates,
cities, handles) — nothing enters the report unless it ties to one, because "Sarah Chen, product
manager" is thousands of people. And a *confidence tier* — Confirmed / Probable / Self-reported /
Inferred / Unverified — with anything clearing neither gate **deleted rather than hedged**.
"Reportedly" and "sources suggest" are the vocabulary of a report quietly making things up.

If the person genuinely has no footprint, it says so and refuses to pad. **The absence is the
finding.**

## Install

```bash
git clone https://github.com/andreteow/linkedin-life-story.git \
  ~/.claude/skills/linkedin-life-story
```

That's it — Claude Code picks up skills in `~/.claude/skills/` automatically. For a single project
instead, clone into `.claude/skills/` in the project root.

Requires Claude Code with `WebSearch` and `WebFetch`. No API keys, no scrapers, no paid data
sources.

## Using it

Just ask. It triggers on the intent, not the format:

```
research this person: linkedin.com/in/someone
who is Ravi Menon? i keep seeing his name in AI policy circles
dossier on our incoming board member, Aisyah Rahman — ex-Grab, ex-Sea, based in KL
dig into this founder before my call thursday
```

**LinkedIn will usually block the fetch.** That's expected and handled: `www.linkedin.com` redirects
to a country subdomain (`my.`, `pk.`, `uk.`) that often *does* serve a partial profile, and
`linkedin.com/posts/…` URLs fetch fine even when `/in/` returns HTTP 999. When all of that fails, the
skill asks you to paste the profile text — which is the best input anyway.

⚠️ **A partial fetch is more dangerous than no fetch.** It looks complete and isn't. In testing, a
fetched profile showed only student-era roles ending eight years earlier and no current position; the
subject had in fact co-founded a national campaign that amended his country's constitution. The skill
treats any fetched profile as a fragment and sanity-checks the person against the open web before
locking anything in.

Reports are written to `~/research/people/<name>.md`.

## Results

Measured against a no-skill baseline on three real subjects, graded by an independent agent:

| | With skill | Baseline |
|---|---|---|
| Assertion pass rate | **89%** | 43% |
| Fabricated/unsupported claims | none found | several, incl. an unsupported arrest and an invented exam result |

Skill triggering, measured across 20 queries (10 that should fire, 10 near-misses that shouldn't):
**100% recall, 100% precision.**

It is **slow and expensive** — roughly 15 minutes and ~105k tokens per person, about 3× a naive run.
The verification pass is where both the cost and the value live. If you want a quick summary of
someone, don't use this. If you're going to *rely* on what you're told, do.

## Limits and ethics

This points at real, living people who did not ask to be researched. Public sources only. It will
not use data brokers, people-search sites, leaked datasets, or scraped contact databases; it will not
hunt for personal email, phone, or home address; it will not bypass a login, paywall or robots.txt;
and it never contacts, connects with, or messages the subject. Scope is the public professional record
plus the personal colour the subject made public themselves — not their health, finances, family
members who never chose to be public, religion, or politics. Controversies are in scope, handled like
a journalist would: cited, with the outcome, and with the subject's response or a note that they
didn't give one.

It is a research tool, not a background check, and it should not be used as one.

## Evals

`evals/` holds the test suite. The assertions are deliberately **method-based** — citation
discipline, confidence tiering, identity gating, honest gaps — rather than facts about specific
people, so they transfer to whoever you point them at. `evals/evals.json` covers the three modes the
skill is built for: a well-documented subject, a moderately-documented one, and a common name with no
unique fingerprint (where the only correct output is a refusal to guess).

## Layout

```
SKILL.md                        the skill: engine, workflow, verification, rails
references/search-playbook.md   query patterns per lane; how to route around blocked sources
references/report-template.md   the output shape
evals/                          method-based test suite + triggering evals
```

## License

MIT.
