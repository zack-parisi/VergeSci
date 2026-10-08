# VergeSci

**Making university research accessible to the students who want in.**

[vergesci.com](https://vergesci.com)

> **Note on the source code.** This repository is documentation only. The VergeSci
> platform is a private, closed-source codebase owned by VergeSci Labs and is not
> published here. This README explains what we build, how the matching and
> recommendation systems work, and where the product stands today. Happy to walk
> through the implementation in detail on a call.

---

## What VergeSci does

University research runs on who you happen to know.

A first-year who wants to work in a lab is told to "email professors." So they open
a department directory of several hundred names, read faculty pages written for other
faculty, guess which three sound closest to what they care about, and send cold emails
into a void. Most never hear back. The students who do land positions are
overwhelmingly the ones who already had a connection: a parent in academia, an older
sibling who did it last year, a TA who made an introduction. Research experience is
the single biggest gate on graduate admissions, medical school, and science careers,
and access to it is distributed almost entirely by social capital.

Meanwhile the information that would answer the student's question already exists in
public. Faculty research summaries, publication records, and federal grant abstracts
are all out there. They are just scattered across thousands of department pages,
written in language built for peers, and effectively unsearchable by an
eighteen-year-old who does not yet know the vocabulary of the field.

**VergeSci closes that gap.** We build a per-campus research graph from public data,
then use it to put the right faculty, labs, and open positions in front of each
student based on what they are actually interested in and what they have actually
done, not who they know.

### Who it serves

| | |
|---|---|
| **Undergraduates and master's students** | Find faculty whose current work genuinely matches their interests, see what each lab is working on right now in plain language, and reach out with context instead of a cold email. |
| **Principal investigators** | Post research positions and see ranked, interest-matched students from their own campus, instead of fielding hundreds of untargeted emails each term. |
| **Graduate students and postdocs** | Discover collaborators, labs, and departmental seminars across their institution. |
| **Universities** | A campus-scoped view of research participation, including which students are looking, which fields are oversubscribed, and where undergraduate research access is thinnest. |

Each campus is a closed loop. A student at one university sees that university's
faculty, that university's positions, and that university's students. Every campus
runs on its own isolated database cluster, so no institution's user data is
co-mingled with another's.

---

## Demo

<!-- TODO: replace the placeholders below with a real recording and screenshots.
     See "Adding the demo" at the bottom of this file. -->

**90-second walkthrough:** a student describes what they want to work on, gets ranked
faculty matches with current-work summaries, and contacts a lab.

> 🎥 _Demo video coming shortly._

| | |
|---|---|
| ![Interest-matched faculty feed](docs/demo/01-discover-labs.png) | **Discover Labs.** Ranked faculty for this student, each card showing a plain-language research summary, current work, and salient tags. |
| ![Semantic research search](docs/demo/02-align-interests.png) | **Align Interests.** Natural-language search across faculty, labs, and open positions. "I want to work on protein folding with computational methods" returns real people. |
| ![Ranked opportunities feed](docs/demo/03-opportunities.png) | **Opportunities.** One ranked feed of open research positions, scored against the student's interests, profile, and project history. |
| ![Lab detail and outreach](docs/demo/04-outreach.png) | **Outreach.** A student contacts a PI with shared context already filled in, so the first message is specific rather than generic. |

---

## How recommendations work

Recommendations are built on dense vector retrieval over a per-campus corpus, then
reranked with a cross-encoder and adjusted by observed behavior. Three ingredients
go in: what the student says, what the faculty actually do, and what the student
does on the platform.

### 1. The student side

Every student gets a single embedding built from a weighted text representation of
who they are. The weighting is deliberate, because stated interests and revealed
interests are not equally informative:

- **Tier 1, active research queries (3x weight).** Every semantic search a student
  runs is captured to their profile, capped at the twenty most recent. These are the
  strongest available signal of real intent, so the deduplicated recent queries are
  placed first in the profile text and repeated three times to dominate the
  embedding.
- **Tier 2, engaged-lab research text (2x weight).** The research text of the top
  five labs a student has actually engaged with, by summed interaction weight, is
  folded in at double weight.
- **Tier 3, declared profile (1x weight).** Research interest statement, major and
  minors, intended degree, coursework, relevant skills, prior research experience,
  career aspirations, availability and commitment length, funding status, and parsed
  CV text.

The result is embedded with OpenAI `text-embedding-3-small` into a 1536-dimensional
vector. It is regenerated asynchronously after every search a student runs, so the
profile tracks a student whose interests shift over a term instead of freezing at
whatever they typed during onboarding.

### 2. The faculty side

Every faculty member in a campus corpus gets an embedding built from name, title,
department, areas of research, and an LLM-generated research summary. Faculty who
match active federal awards also get a grant-enriched embedding (see
[research freshness](#the-research-freshness-challenge) below).

### 3. Retrieval, then reranking

1. **Approximate nearest neighbor search.** The student vector is queried against the
   campus faculty index using MongoDB Atlas Vector Search (HNSW), which returns
   candidates ranked by **cosine similarity** between the student embedding and each
   faculty embedding. Open positions and other students are retrieved the same way
   against their own indexes.
2. **Cross-encoder rerank.** The top candidates are reranked with Cohere
   `rerank-v3.5`, scoring the student's actual query text against each candidate's
   research text jointly. A bi-encoder has to compress a whole research program into
   one vector before it ever sees the query; a cross-encoder reads both together and
   is markedly better at the near-miss cases, which is exactly where ranking quality
   is decided.
3. **Behavioral adjustment.** Rerank scores are boosted by the student's accumulated
   interaction signals for each entity.

### 4. Interaction signals

High-signal events are written back to the student's own campus database and feed both
the embedding and the ranking boost. Weights reflect how much each action actually
reveals about preference:

| Signal | Weight |
|---|---|
| Message sent to a lab | 1.00 |
| Contact clicked | 0.80 |
| Semantic research search submitted | 0.70 |
| Lab saved / position saved | 0.50 |
| Labs search submitted | 0.50 |
| Lab card clicked / lab website clicked / papers clicked | 0.30 |
| Position card clicked | 0.30 |
| Connection requested | 0.20 |
| User profile viewed | 0.10 |
| Contact modal opened / abandoned | 0.00 |

The zeros are intentional. Opening a compose box and then closing it says nothing
about whether a student likes that lab, so those events are recorded for funnel
analysis but kept out of ranking entirely. Letting telemetry leak into the
recommender is a quiet way to poison it.

---

## The research freshness challenge

**The problem: published work is a lagging indicator of what a lab actually does.**

Our first-generation faculty embeddings were built from department-page research
blurbs and publication summaries. This worked, and it had a structural flaw we could
not fix by tuning. Papers describe work that was done. Between the experiments and
the published record sit submission, review, revision, and press, and in many fields
that is two to four years. A faculty page can be years out of date on top of that,
because nobody updates their bio.

So a student searching for a lab working on something would get matched on a
professor's greatest hits from 2019 and show up to a meeting about a project that
had already wrapped. Worse, the labs taking on new students are precisely the ones
that just started something new, which is exactly the signal publications cannot see.

**The fix: embed the federal grant record instead.**

Funded research announces itself years before it publishes. An NIH or NSF award
abstract is a researcher describing, in their own words, what they are about to spend
the next three to five years doing, and it is public on the day the award is made.
It is the closest thing to a forward-looking index of a lab's actual agenda.

We built an ingestion pipeline that, for each campus:

1. Pulls every active award from **NIH RePORTER** and the **NSF Awards API** scoped to
   that institution's organization names.
2. Matches awards to indexed faculty by name, with manual review of low-confidence
   matches. This step is less mechanical than it looks. Organization naming is
   genuinely treacherous: Georgia Tech's NSF awards are issued to "Georgia Tech
   Research Corporation" rather than any campus name, a bare "University of Illinois"
   query folds Urbana into Chicago, and Vanderbilt's NIH organization string collapses
   the university and the medical center into one entity.
3. Writes structured grant records per faculty member: project titles, abstract
   summaries, NIH spending categories, preferred terms, co-investigators, NSF funding
   program names, and top publications.
4. Builds a second **grant-enriched embedding** (`vector_v2`) from the base research
   profile concatenated with that grant text. Faculty with no federal funding keep
   their base embedding, so coverage is never a cliff.
5. Generates the plain-language layer students actually read: a short research
   summary, a **current work** paragraph grounded in active awards, and salient topic
   tags.

Across launched campuses, roughly half to three-quarters of indexed faculty match at
least one active NIH or NSF award, so this is a majority of the corpus rather than a
long-tail enhancement.

**Where it stands.** The grant-derived *student-facing* layer is live everywhere: the
current-work paragraphs, short summaries, and salient tags on every lab card are
generated from active award text, and they are the single biggest improvement students
have remarked on. The grant-enriched *retrieval* embedding is built and populated
across campuses but is still being evaluated offline against the base embedding before
it takes over the serving path. Promoting a recommender's retrieval layer without a
real evaluation is how you ship a regression you cannot see, so that one waits for the
benchmark.

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. DATA INGESTION                                                       │
│                                                                         │
│    Public faculty directories ──┐                                       │
│    NIH RePORTER awards ─────────┼──► per-campus faculty corpus          │
│    NSF Awards API ──────────────┤    (name, dept, contact, research,    │
│    PubMed / OpenAlex ───────────┤     grants, publications)             │
│    Departmental seminar feeds ──┘                                       │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│ 2. FACULTY MATCHING & RESOLUTION                                        │
│                                                                         │
│    Name matching across sources · institutional disambiguation ·        │
│    department canonicalization (messy labels → LLM-normalized buckets)  │
│    · deduplication · confidence tiers with manual review                │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│ 3. RESEARCH SUMMARIES & EMBEDDINGS                                      │
│                                                                         │
│    LLM-generated: research summary · current work (from active awards)  │
│                   · salient tags                                        │
│    Embeddings:    base profile vector · grant-enriched vector           │
│                   (text-embedding-3-small, 1536-dim)                    │
│    Indexed:       Atlas Vector Search (HNSW, cosine)                    │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│ 4. RECOMMENDATIONS                                                      │
│                                                                         │
│    student embedding (queries 3x · engaged labs 2x · profile 1x)        │
│         │                                                               │
│         ├─► ANN retrieval, cosine similarity ──► candidate set          │
│         ├─► cross-encoder rerank (Cohere rerank-v3.5)                   │
│         └─► interaction-signal boost ──────────► ranked results         │
│                                                                         │
│    Three surfaces: recommended labs · ranked opportunities ·            │
│                    recommended students (the PI-side reverse match)     │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│ 5. STUDENT EXPERIENCE                                                   │
│                                                                         │
│    Discover Labs · Align Interests (semantic search) · Opportunities ·  │
│    Seminars · Projects & Science Fair · contextual outreach to PIs      │
│                                                                         │
│    Every interaction feeds step 4 back through the signal layer ────────┼──┐
└─────────────────────────────────────────────────────────────────────────┘  │
                                  ▲                                         │
                                  └─────────────────────────────────────────┘
```

**Stack:** Next.js 15 (App Router) · React 19 · TypeScript · Tailwind · MongoDB Atlas
with Vector Search · OpenAI embeddings · Cohere rerank · Python ingestion pipelines ·
Vercel · PostHog.

**Multi-tenancy:** each campus is a separate Atlas cluster reached by its own
subdomain, resolved per request. Student data never crosses institutional boundaries
by default. The one exception is explicitly opt-in: consortium campuses that have
agreed to share, such as the Pittsburgh Research Network linking Pitt and Carnegie
Mellon, where writes stay on the home cluster and reads fan out.

---

## Traction

_All figures as of October 2026. Deliberately separated by confidence, because
"users," "committed," and "reachable" are three very different claims and conflating
them is the oldest trick in the deck._

### Live usage

| Metric | Value |
|---|---|
| **Weekly active users** | **1,000+** |
| Launched campuses accepting signups | **11** |
| Indexed faculty profiles across launched campuses | **~20,400** |
| Faculty matched to active NIH/NSF awards | roughly 50 to 75% of indexed faculty per campus |
| Departmental seminars ingested | several hundred per term, refreshed weekly |

### Launched campuses

Live today, real students creating real accounts on their own institutional email:

- University of Chicago
- Case Western Reserve University
- University of Illinois Urbana-Champaign
- Carnegie Mellon University
- University of Michigan
- University of Virginia
- Georgia Institute of Technology
- University of Pittsburgh
- Vanderbilt University
- Northwestern University
- Emory University

Most recent launches: Vanderbilt (August 2026), Georgia Tech (August 2026),
Carnegie Mellon (September 2026), Northwestern (September 2026), Emory (October 2026).

### Built and provisioned, opening next

Full data pipeline complete and infrastructure provisioned, signups currently routed
to a waitlist pending launch:

- Pennsylvania State University
- Purdue University
- University of Illinois Chicago

Combined, these add roughly **4,400** indexed faculty profiles that are already built.

### Addressable reach

Not users. The enrolled student population of the eleven launched campuses is roughly
**300,000**, which is the ceiling our current infrastructure already covers without
onboarding another institution.

### Other signals

- **Institutional programming.** The VergeSci Showcase, a research project competition
  with published official rules, has run in two cycles across six campuses, including a
  joint cross-institutional fair between Pitt and Carnegie Mellon.
- **Faculty participation.** PIs claim their own lab profiles and post research
  positions directly, which converts a scraped directory entry into a maintained,
  owner-verified listing.
- **Repeatability.** The per-campus pipeline, from scrape through enrichment,
  embedding, and launch, has been run end to end fourteen times. Onboarding a new
  university is now a known process rather than a research project, which is what
  makes the reach above a plan instead of a hope.

---

## Why this matters

Undergraduate research experience is one of the strongest predictors of whether a
student goes on to a career in science, and right now it is allocated largely by
who already knows how the system works. First-generation students, transfer students,
community college students, and students at large public universities where a single
department can hold four hundred faculty are the ones who lose most from a
network-based system, and they are the ones a search engine over public data helps
most.

Every piece of information VergeSci surfaces is already public. We are not creating
access to anything that was previously restricted. We are removing the requirement
that a nineteen-year-old already speak the language of a field in order to find out
who in it might want their help.

---

## Contact

- **Platform:** [vergesci.com](https://vergesci.com)
- **Zack Parisi** · [@zack-parisi](https://github.com/zack-parisi)

---

<details>
<summary><b>Adding the demo (notes to self)</b></summary>

The demo section above references files that do not exist yet. To finish it:

1. Record a 60 to 90 second screen capture of the student flow: onboarding interests →
   Discover Labs → Align Interests search → a lab card → contact. Upload it to YouTube
   or Loom and replace the `🎥 Demo video coming shortly.` line with a thumbnail link:

   ```markdown
   [![Watch the demo](docs/demo/thumbnail.png)](https://youtu.be/VIDEO_ID)
   ```

2. Take four screenshots and commit them to `docs/demo/`:
   - `01-discover-labs.png`
   - `02-align-interests.png`
   - `03-opportunities.png`
   - `04-outreach.png`

   GitHub also accepts images pasted directly into a web edit of this file, which
   uploads them to its own CDN and may be faster than committing binaries.

3. Scrub any real student names, emails, or profile photos before publishing. Use
   the sandbox campus (VergeSci University) for clean demo data.

</details>
