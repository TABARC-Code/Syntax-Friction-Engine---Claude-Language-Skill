# Project Description

Author: TABARC-Code

## so the short version

syntax-friction-engine exists because telling a model to "write clearly" changes the surface of a draft and leaves the habits underneath. The habits are specific and countable: the word "that" dropped from relative clauses, nouns piled three deep, verbs that arrive a line late. So there's a deterministic counter, a rewriting procedure that uses it, and a meaning guard that stops the rewrite winning by deleting things.

That was the design. The testing showed me that the counter is the least trustworthy part. That's an awkward thing to learn about the part that I built first. Most of what follows is the record of that.

## The problem

Clark, Martin and Mahowald counted six constructions in model output (arXiv:2610.29162, October 2026, not peer reviewed). They were reduced object relatives, long dependencies, word pairs of low frequency, stacked nouns, inanimate subjects with person-only verbs, and nouns used as verbs. Their explanation is that training on maths, code, and reasoning rewards short answers. The function words get squeezed out, and those are the small words that help a human parse a sentence.

They compared model text with round-trip-translated paraphrases of itself, which holds the content roughly constant. 

The practical problem is narrower than the paper. Someone has text that reads machine-made and wants it to read like a person wrote it, without any number, name, hedge, or condition changing. 

> The paper names what to look for. 
However It gives no advice on how to edit. Do I used theur work as the base. 

## The approach

There are four moving parts, each there because an alternative failed or was unmeasurable.

`scripts/friction_score.py` is the counter. It parses text with spaCy and counts five of the six features, plus long sentences and long paragraphs, which aren't from the paper. `SKILL.md` holds the rewriting procedure. It edits for the reader first, uses the counter second, and finishes with a hand-read for what the parser can't see.

`compare` is the meaning guard. It checks numbers, entity text, content recall, and length growth between the before and after. The last part is a feedback ledger plus a test suite, and together they make failures permanent. A false flag becomes a lexicon entry or a fixture, a miss becomes a known-miss fixture, and a regression gets written down.

## Architecture

```
SKILL.md                     
what an agent does, in order, and why
scripts/friction_score.py markdown cleaning > spaCy parse > per-sentence detectors > counts, compare, guard
scripts/build_ngrams.py corpus > unigram and bigram counts (optional, for the sixth feature)
scripts/selftest.py tests/fixtures.json > pass or fail per case
references/lexicon.json word lists and thresholds the detectors read
references/feedback-ledger.md human-readable log; not read by any code
tests/fixtures.json the answer key: positive, negative, and known-miss cases
reader-test blind rating page, and an organiser folder with the key
```

The counter does its own markdown cleaning before parsing. Fenced code is dropped, and inline code becomes a placeholder. URLs and tables go. List items become their own paragraphs, and short unpunctuated lines (headings and labels) are skipped, since they aren't running prose. Words inside double quotes are skipped for the lexicon features, because "the framework wants to" quoted as an example isn't the author doing it.

Detectors work sentence by sentence on the dependency parse. A reduced relative is a relative clause with its own subject and no relativiser. Adverbial heads ("every time a word reaches") and clauses that cross a semicolon don't count.

Late verbs are a subject-to-verb gap of seven or more words, skipping gaps that contain a colon, a semicolon, or a lone dash. Noun stacks are three or more nouns in a row, ignoring named entities and prefix hyphens like "sub-skill". The two lexicon features read `references/lexicon.json`.

## Control flow

```
text > score (hits, long sentences) > readability pass with a keep list
     > targeted fixes for flagged lines > hand-read > compare (guard)
     > pass: report            fail: revert that sentence, smaller edit, repeat (max three rounds)
```

The order is deliberate. Version 1.0 ran the counter first and edited only until it was happy. I'll come back to how that went.

## Design decisions

### A deterministic counter, not prompt rules alone

So you can't check a prompt that says "avoid noun stacks", so every claim about improvement would rest on opinion. The counter gives a before and after. The price turned out to be Goodhart's law: the rewriting loop was rewarded for satisfying the counter, and did. Prompt-only was rejected for being unmeasurable, and counter-only after testing.

### Readability first, counter second, hand-read last

Version 1.0 edited only flagged sentences. Blind judges rated its output exactly level with the untouched originals (3.16 each), while a plain "shorter sentences, ordinary words" prompt scored 4.74. Sentence length, paragraph shape, and dash-delimited lists matter to readers as much as the six features. The "minimum effective edit" idea was kept only for the LIGHT intensity, where the text is a human's and the voice is worth protecting.

### Lexicon lists instead of a classifier

The paper used a classifier for animate verbs on inanimate subjects and for nouns used as verbs. I didn't have one. Word lists are small, editable, and wrong in predictable ways, so they output candidates with confidence labels instead of verdicts. The denominal list was cut back after "stack" produced a false flag. Old verbs like "map" and "tag" went with it, since they aren't novel conversions.

### spaCy medium model as the parser

Small was the first choice; medium parsed reduced relatives better. I tried large: it cleared two known misses and broke two passing fixtures, a straight trade of errors, so it stayed an option behind `SFE_MODEL`.

### A pattern route for bare relatives the parser files as `ccomp`: tried, reverted

The idea was to catch "a timeout the client never saw resolved" by pattern. The parser's errors were too deep (it read "never saw" as an auxiliary), so the route never fired on a real miss. Dead code is worse than a documented miss, so it went, and two fixtures were added as known misses.

### The meaning guard checks numbers, entity text, content recall, and length

Content recall is the share of the original's content words that survive, and it must be at least 0.55. Length can grow by no more than 1.6 times. The thresholds are heuristics.

Entity checking first compared spaCy's named-entity output before and after. That wobbled: "2.3 million" was reported lost while it was still in the text. It now checks by string containment, and the result is informational only.

### The keep list and the "add nothing" rule

Do 'Blind judges' kept catching the same faults in rewrites. A conditional got flattened into a statement, or a hedge was dropped. A count appeared that the source never had ("five of them matter"), or an ordinal list, or an added "especially". Those became explicit rules in the readability pass. The guard cannot check them (see limitations).

### BLUNT as the default for model-written text

It averages 10 to 12 words a sentence, caps sentences at 22, and keeps paragraphs to three sentences. It was added after FULL still rated below the plain prompt. On fresh passages it rated best, with the fewest hand-counted constructions. It also changes the most, so it asks the most of the keep list.

### Known misses stay in the test suite

Five fixtures describe constructions that the parser gets wrong. They're marked `known_miss`, reported by the self-test as known, and flip to "NOW OK" if a future parser fixes one. The alternative was deleting hard cases to keep the number tidy, which would have made 33/33 a more flattering figure and a less honest one.

### The sixth feature is optional

Low-frequency n-grams need a reference corpus matched to the readers. I won't ship one, and a mismatched one would produce plausible noise. `build_ngrams.py` and `--ngram-counts` exist; hits are low confidence and reported beside the main count, never inside it.

## Trust and boundaries

The counter owns the numbers that it prints and nothing else. spaCy and its model own the parse, and therefore every limit that comes with parsing. `references/lexicon.json` owns the word lists, and whoever edits it owns the consequences. The agent following `SKILL.md` owns the actual rewrite, and no code verifies hedges, conditions, or the claims themselves. The ledger is plain text that nothing reads at run time.

Credentials never cross any of these boundaries. The scripts read named files and write only the output file `build_ngrams.py` is told to, make no network calls, and start no subprocesses. I checked by searching the scripts for network calls, code execution, and file writes. The one network step is the model download, which the user runs.

In several test runs the agent couldn't write to the skill folder. Some were told not to, and at least two reported it read-only. Ledger lines had to be reported instead of written. That is a boundary worth knowing about: the feedback loop only closes where something is allowed to write.

---
## Ok I'm dumping my thinking and results from my experiments here. 

Incase you are interested and want yo play. 
---
## Current state

| Item | State | Evidence class |
|---|---|---|
| Counter, compare, guard run and produce the output shown in the README | working | VERIFIED: run during this audit |
| Self-test: 33 scored fixtures pass, five known misses reported | working | VERIFIED: run during this audit |
| Five of six features counted; sixth optional | working, with limits | VERIFIED for the five; the optional path SUPPORTED (tested on a tiny corpus only) |
| Lexicon features | partial | SUPPORTED: candidates, tuned against ledger entries |
| Parser handling of bare relatives and long sentences | not solved | VERIFIED: five known misses |
| BLUNT rated best on ease | experimental | VERIFIED as observed, by model judges, on 14 passages |
| Real-reader ratings | not collected | UNKNOWN: pack built and smoke-tested on fake ratings only |
| "Six times better" | not reached on any count I trust | USER-STATED target; measured figures below |

## What the testing showed

### Method

Each passage went to a blind judge in four shuffled, unlabelled versions. They were the original, a plain-prompt revision, and one or two versions of the skill. The judge rated each from 1 to 5 for ease of reading, and listed any change of fact, number, name, hedge, condition, or added content.

A separate annotator, also blind, counted the constructions by reading, with no scorer, and quoted each span so the count could be audited. Each revision was produced by its own model instance given only that arm's instructions and the input files. The plain prompt is the paper's own remedies written out. Name the audience, use short sentences, keep "that", avoid noun stacks, do a second pass, and prefer plain words.

Caveats about the method itself. Judges and writers come from the same model family, which could share blind spots. Counts are small.

Two instances in one round ran the scorer across folders other than their own, so they glimpsed other arms' score summaries (not the input texts). One batch of runs was cut off by a service interruption and rerun with identical instructions. Several scorer fixes were made after looking at results. The first test's passages are therefore not clean, but the second test's were never tuned on.

### First test: 31 passages

It used 10 sections of a briefing written by a model, 15 paragraphs from skill files in my own library, and six dense synthetic passages. The skill files are mostly model-assisted, though I haven't verified who wrote each.

| Round and version | Blind ease (1 to 5) | Hand-counted constructions per 100 words | Scorer hits per 100 words |
|---|---|---|---|
| Round 1: original | 3.16 | 1.61 | 1.48 |
| Round 1: plain prompt | 4.74 | 1.23 | 1.10 |
| Round 1: skill v1.0 | 3.16 | 1.00 | 0.20 |
| Round 2: original | 2.87 | 1.90 | 1.50 |
| Round 2: plain prompt | 4.68 | 1.38 | 1.10 |
| Round 2: skill v1.0 | 2.87 | 1.00 | 0.22 |
| Round 2: skill v1.1 | 3.65 | 0.78 | 0.15 |
| Round 3: original | 2.68 | 1.90 | 1.50 |
| Round 3: plain prompt | 4.68 | 1.38 | 1.10 |
| Round 3: skill v1.1 | 3.16 | 0.82 | 0.15 |
| Round 3: skill v1.2 | 4.23 | 0.73 | 0.26 |

Judges' absolute scores drift between rounds, so compare within a round, not across them. In round 3, v1.2 beat the originals in 26 passages and lost none. Against the plain prompt it lost 17, drew nine, and won five. Hand-counted constructions were 2.5x fewer than the originals (80% bootstrap interval 2.1 to 3.1) and 1.9x fewer than the plain prompt (1.6 to 2.3).

The finding was the gap between the scorer and the hand count. On the originals and the plain-prompt output the two agreed closely. On the skill's early output the scorer saw 9 hits where the annotator counted 45. The loop had optimised the thing that it was graded on. Which is a polite way of saying it cheated, and I wrote the loop.

The judges' complaints about changed meaning, over the three rounds: the plain prompt had none, three, then five. Skill v1.0 had none, v1.1 one and then none, and v1.2 two.

### Second test: 14 fresh passages, version 1.3

Eight paragraphs from the Python Language Reference (human-written), and six new dense synthetic passages. Nothing was tuned on these.

| Version | Blind ease (1 to 5) | Hand-counted constructions per 100 words | Meaning or added-content complaints |
|---|---|---|---|
| Original text | 2.29 | 2.06 | 0 |
| Plain prompt | 4.07 | 1.49 | 4 |
| Skill v1.2 (FULL) | 4.00 | 0.48 | 0 |
| Skill v1.3 (BLUNT) | 4.79 | 0.42 | 2 |

BLUNT beat the plain prompt in 10 passages, drew two, and lost two. It beat FULL in 11 and the originals in all 14. On the human-written reference paragraphs alone BLUNT scored 4.88, which may flatter it.

Hand-counted constructions: 3.5x fewer than the plain prompt (28 against 8) and 4.6x fewer than the originals (37 against 8). The plain prompt added content in four passages: an invented ordinal list, a "three fixes" count, a softened condition, and a reworded modifier. BLUNT's two complaints were a softer phrase and a reworded page reference.

A second independent run of the six dense passages gave scorer hits of 1 and 3 for BLUNT and 1 and 4 for FULL. The plain prompt gave 20 and 21. The skill arms are noisier than the baseline, which matters when the counts are this small.

### Where "six times" went

Scorer-only figures against the plain prompt ranged from 1.6x on low-density text to 26x on the dense synthetic set. Which end you got depended on the set and the version. They were inflated, because the loop was rewarded for lowering that number.

The independent hand counts give 3.5x against the plain prompt and 4.6x against the originals, on small numbers. The target was missed on every measure that I trust. On blind ease the ceiling is 5, so "six times" can't be met on that measure by anything.

---

## Operational notes

Run the counter with `python3 -I` from anywhere; it needs no configuration. Keep spaCy and its model in step: the parse, and so the counts, change with the model version. Re-run the self-test after upgrading either. If you edit the lexicon, edit it in response to a ledger entry, and add a fixture first.

## Limitations and awkward bits

The meaning guard checks numbers, entity text, content recall, and length. It does not check hedges or conditions. The keep list is an instruction to the agent, not code, so "if" becoming "is" depends on the agent doing as told. The judges caught exactly that fault in the plain prompt's output, and twice in the skill's.

The parser struggles on the hardest sentences, which are the ones that most need help. Long, dense sentences confuse parsers as well as people.

The scorer and the revision loop share a blind spot by construction. The answer is the hand-read, which depends on an agent actually reading carefully.

Thresholds are mine: a three-noun stack, a seven-word subject-to-verb gap, a 25-word sentence, a five-sentence paragraph. A write-up of the paper's opening sentence describes "reader-patience budget" as a three-noun stack, which supports the first. The rest are judgement.

The self-test fixtures borrow example constructions quoted in write-ups of the paper. They're short, and there are four of them. I wrote the rest.

## Future direction

Planned, nothing started: I will get around to. Or you build it. 

- Collect real-reader ratings with `reader-test/` and compare them with the model judges. If they disagree, the readers win and the ledger records it.
- Try a second-opinion pass for bare relatives, or a better parser, against the existing known-miss fixtures.
- Build a proper reference table for the sixth feature from a corpus matched to the intended readers.
- Write the notes for the other end of each link into the neighbouring skills, once I'm sure which of their copies are current.
