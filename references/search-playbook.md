# Search Playbook

Query patterns per lane, and how to handle the sources that block you.

**Contents**
- [The two rules that govern every query](#the-two-rules-that-govern-every-query)
- [Lane 1 — Own voice](#lane-1--own-voice-highest-yield)
- [Lane 2 — Core web & press](#lane-2--core-web--press)
- [Lane 3 — Self-published](#lane-3--self-published)
- [Lane 4 — Records (conditional)](#lane-4--records-conditional)
- [Blocked sources: what to do](#blocked-sources-what-to-do)
- [Name collision: how to break a tie](#name-collision-how-to-break-a-tie)

---

## The two rules that govern every query

**1. Never search the bare name.** `"Sarah Chen"` returns a thousand strangers. Every query pairs
the name with a **fingerprint token** — an employer, a school, a city, a product, a co-founder:

```
"Sarah Chen" Stripe
"Sarah Chen" "University of Michigan"
"Sarah Chen" bakery Oakland
```

This is not a nicety. It is the mechanism that keeps another person's life out of the report.

**2. Search the seam, not the subject.** "Sarah Chen biography" returns nothing, because nobody
wrote one. The productive queries target the *transitions* you mapped:

```
"Sarah Chen" "left Goldman"
"Sarah Chen" "why I quit" OR "why I left"
"Sarah Chen" founder story
"Sarah Chen" "how I got started"
```

---

## Lane 1 — Own voice (highest yield)

Where the life story actually is. A person who has never been written about has still been *asked
about themselves* — and podcast hosts ask exactly the questions you want answered.

```
"<Name>" podcast
"<Name>" interview <employer>
"<Name>" episode
"<Name>" "in conversation with"
"<Name>" keynote OR panel OR talk <industry>
site:youtube.com "<Name>" <employer>
"<Name>" fireside chat
"<Name>" AMA
"<Name>" alumni profile <school>
"<Name>" "tell us about yourself" OR "how did you get into"
```

Then chase the **conference and event circuit** — speaker pages carry a bio the subject wrote and a
talk abstract that reveals what they care about:

```
"<Name>" speaker <conference name>
"<Name>" 2023 OR 2024 summit OR conference speaker
```

**Working the finds:**
- YouTube and most podcast platforms have transcripts. Fetch them. A 45-minute interview is often the
  single richest document in the entire research run — it usually opens with "so tell us how you got
  here", which is literally the question you are trying to answer.
- Show notes frequently contain a bio and the episode's key beats. Cheap to fetch, high value.
- Pull direct quotes verbatim. They are unfalsifiable and they carry the person's voice into the
  report in a way your prose never will.

## Lane 2 — Core web & press

```
"<Name>" <employer>
"<Name>" <employer> announcement OR appointed OR joins OR promoted
"<Name>" <company> funding OR raises OR Series
site:crunchbase.com "<Name>"
"<Name>" <company> "co-founder" OR "founding team"
"<Name>" award OR "40 under 40" OR "power list"
site:<company-domain> "<Name>"          ← their employer's own About/team/leadership page
"<Name>" quoted OR "said" <industry topic>
"<Name>" obituary OR memoriam <family surname>   ← only where it bears on the story; handle with care
```

Company About pages and press releases are **Self-reported** evidence, not Confirmed — the company
is not a neutral witness about its own executive. Trade press covering the same fact independently
upgrades it.

Also worth a pass: **the companies themselves.** If the subject was employee #4 at a startup that
imploded in 2019, the story of that implosion *is* part of their life story, and it is far better
documented than they are. Research the orbit when the person is thin — but attribute carefully:
the company's story is not automatically theirs.

## Lane 3 — Self-published

Their own writing is the closest thing to a primary source you'll get.

```
site:medium.com "<Name>"
site:substack.com "<Name>"
site:github.com "<Name>"                    ← the README and bio, plus commit history dates
"<Name>" blog OR newsletter <employer>
site:x.com "<Name>"  /  site:twitter.com "<Name>"
"<Name>" "my first job" OR "growing up" OR "when I was"
```

`"<Name>" "growing up"` and similar first-person phrases are startlingly effective. People narrate
their origins in throwaway posts.

**Personal sites** are gold and easy to miss. Try `<firstname><lastname>.com`, `.io`, `.dev`,
`.me`. Many carry an unguarded "About" written years ago that says more than any press coverage.

## Lane 4 — Records (conditional)

Run **only** if the anchors say academic, researcher, scientist, clinician or inventor. For an
operator or marketer this lane returns nothing and costs you ten minutes.

But when it applies, it isn't a supporting lane — it *is* the biography. A researcher's papers, in
order, are their intellectual autobiography: the field they entered, the lab they trained in, who
they published with, when they switched problems, and who they became.

```
site:scholar.google.com "<Name>"
"<Name>" patent
site:pubmed.ncbi.nlm.nih.gov "<Name>"
"<Name>" thesis OR dissertation <university>
"<Name>" ORCID
"<Name>" "et al" <field>
```

Read the co-author list and the affiliation history. Both are timeline evidence, and both are
fingerprint-checkable.

---

## Blocked sources: what to do

Blocks are normal. Route around them, and **record every source you could not reach** — the report's
provenance section must show its blind spots.

| Source | Reality | Route around it |
|---|---|---|
| **LinkedIn `/in/` profile** | `www` 301s to a country subdomain, which often *does* serve a partial public profile. HTTP 999 = hard bot-block. | Follow the redirect to `my.` / `pk.` / `ae.` / `uk.linkedin.com` and fetch again. On 999, stop and ask the user to paste — retrying and trying other subdomains just returns 999. **Treat any fetched profile as a fragment, never an anchor list** (see SKILL.md, "the partial-fetch trap"). |
| **LinkedIn `/posts/`** | **Not blocked the way `/in/` is** — these fetch fine even when the profile returns 999. | `site:linkedin.com/posts "<Name>"` via WebSearch, then WebFetch each hit. This is own-voice material — their announcements, their exits, their reflections — and it has salvaged an entire run where the profile was walled. Always try it when `/in/` fails. |
| **X / Twitter** | Login wall since 2023 | Search-engine indexes, `nitter`-style mirrors, quoted screenshots in articles, and `site:x.com` in WebSearch (results still surface even when fetch fails) |
| **Instagram** | Login wall on most content | Public-profile snippets via search; press coverage that embeds posts. Usually not worth much effort. |
| **Facebook** | Mostly walled | Skip unless a specific public page is known |
| **Medium / Substack** | Sometimes metered | Usually fetchable; try the RSS feed if the page fights you |
| **Academic PDFs** | Often fetchable | Try the arXiv or repository mirror if the publisher blocks |

Do not attempt to bypass a login, paywall, CAPTCHA, or `robots.txt`. A 403 is an answer. Write it
down as one.

---

## Name collision: how to break a tie

You will hit this constantly. The tools, in order of strength:

1. **Co-occurring fingerprint tokens.** Does the article mention the employer, the school, the city,
   the co-founder? One match is suggestive; two is strong.
2. **Timeline coherence.** A 2012 article about a "20-year veteran" cannot be your subject if their
   first degree was in 2014.
3. **Photograph.** If the source carries a picture and you have the profile photo, compare.
4. **Field.** A Sarah Chen who is a paediatric oncologist is not the Sarah Chen who runs growth at a
   fintech — unless a seam says she retrained, which is exactly the kind of thing you should check
   rather than assume.
5. **Handle continuity.** The same `@handle` across GitHub, X and a personal site is strong linkage.

If, after all that, two plausible candidates remain **and the difference is material to the story —
stop and ask the user.** Present both with their distinguishing details. The user usually knows
instantly. Guessing here doesn't produce a slightly-wrong report; it produces a confident report
about a stranger, and nobody downstream will ever catch it.
