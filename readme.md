# syntax-friction-engine

Author: TABARC-Code
Version: 1.3.1
Shelf mark: 401.900 (Dewey-style, psycholinguistics applied to written style)

A skill and a small Python counter for technical prose that reads as if a machine wrote it. The counter finds sentence shapes that slow a reader down: a dropped "that", stacks of nouns, and a verb that turns up a line late. It also finds things that "want" and "bite", and nouns used as verbs. The skill rewrites for the reader and checks that no number, name, hedge, or condition got lost on the way.

It actually started from one preprint (Clark, Martin and Mahowald, arXiv:2610.29162, October 2026, not peer reviewed). I read it through secondary write-ups, not the PDF, so my account of it is secondhand. The authors didn't test real readers either. Counting features isn't measuring comprehension, and nothing here pretends otherwise..

## What it does

There are three modes. REPAIR takes your text, rewrites it for the reader, and reports what changed. DRAFT briefs itself first, writes, and then does one repair pass. AUDIT scores the text and lists the problem lines, and edits nothing.

There are also three intensities. BLUNT is the default for model-written text, or when you ask for plain. It aims for 10 to 12 words a sentence, paragraphs of three sentences at most, and lists unpacked into sentences. FULL is gentler, for text of unknown origin. LIGHT is for your own writing: it splits the long sentences and leaves everything else word for word.

Every rewrite starts with a keep list: numbers, names, quantity words ("many", "most"), hedges ("may", "likely"), and conditions ("if", "unless"). They're ticked off afterwards. A claim has to leave as strong as it arrived.

## Status

| Part | State | Evidence |
|---|---|---|
| Counter for five of the paper's six features | implemented | self-test: 33 scored fixtures pass |
| Long-sentence and long-paragraph counts | implemented | not from the paper; informational only |
| `compare` with the meaning guard | implemented | exercised in every test round |
| Animate-verb and noun-as-verb detection | implemented, lexicon-based | candidates only; no trained classifier |
| Sixth feature (unusual word pairs) | experimental | needs a reference table you build; tested on a tiny corpus only |
| BLUNT intensity | experimental | rated best by model judges on 14 passages; no real readers yet |
| Reader test pack | implemented, unused | parses and unblinds fake ratings; no real ratings collected |
| Bare relatives the parser misfiles | not handled | five known misses are in the test suite, listed rather than hidden |

## Requirements

- Python 3 (tested on 3.13)
- spaCy and an English model (tested with spaCy 3.8 and `en_core_web_md` 3.8.0)
- Nothing else. No network access at run time.

## Install

```
pip install spacy
python -m spacy download en_core_web_md
```

To use it as a skill, copy this folder into whichever skills directory your agent tool reads. Where that is depends on the tool; I haven't verified the steps for every one. Without an agent, the scripts work on their own.

## First run

Save two sentences as `draft.md`:

```
The framework wants a bigger cache. The fix we shipped bites when the cache is cold.
```

```
python3 -I scripts/friction_score.py score draft.md --lines
```

You should see this:

```
== draft.md
words 16  sentences 2  avg sentence 8.0  longest 10  over 25 words: 0  paragraphs over 5 sentences: 0
friction per 100 words: 12.5   (medium/high confidence only: 12.5)
  reduced_object_relative              1  (6.25/100w)
  late_verb                            0  (0.0/100w)
  noun_stack                           0  (0.0/100w)
  inanimate_subject_animate_verb       1  (6.25/100w)
  denominal_verb                       0  (0.0/100w)
  low_frequency_ngram                unscored
  (overt 'that/which' relatives seen: 0)
- [inanimate_subject_animate_verb | medium] 'framework wants'  (agentive verb 'want' on a non-person subject)
    The framework wants a bigger cache.
- [reduced_object_relative | medium] 'The fix we shipped'  (no 'that' before 'we shipped')
    The fix we shipped bites when the cache is cold.
```

Now a revised version, saved as `after.md`:

```
The framework needs a bigger cache. The fix that we shipped causes slow requests when the cache is cold.
```

```
python3 -I scripts/friction_score.py compare draft.md after.md
```

```
friction/100w  12.5 -> 0.0   ratio all hits removed
words 16 -> 19  (x1.19)   content recall 0.778
avg sentence 8.0 -> 9.5
numbers lost: none   entities lost: none
meaning guard: PASS
```

Then run `python3 -I scripts/selftest.py`. It should end with `33/33 scored fixtures pass; 5 known misses listed, not hidden`. If it doesn't, something is wrong with your spaCy model before it's wrong with the code.

The `-I` is deliberate. It stops a stray file in your working directory shadowing a Python module.

## Usage

```
python3 -I scripts/friction_score.py score FILE [FILE ...] [--lines] [--json] [--gate N] [--ngram-counts TABLE.json]
python3 -I scripts/friction_score.py compare BEFORE AFTER [--json]
python3 -I scripts/build_ngrams.py YOUR_DOCS_FOLDER -o counts.json
python3 -I scripts/selftest.py
```

`--gate N` exits 1 if friction per 100 words exceeds N. `compare` exits 1 if the meaning guard fails. Exit 2 means a setup problem. Use `-` as a filename to read from standard input.

With an agent, describe the job in plain words. "This reads like a machine wrote it" works. So does "make this README plainer", or "write the release notes so they don't sound generated". Ask for "blunt" or "plain" to get BLUNT, or say the text is yours to get LIGHT.

The sixth feature needs a table. Build it from text that a careful human wrote for the same readers, such as your own docs or a manual, then pass it with `--ngram-counts`. If a word pair never appears in that table although both words are common, it's listed as a low-confidence candidate. They're reported beside the main count, not inside it. A small or off-topic corpus gives false flags, and `build_ngrams.py` prints a warning below 200,000 words.

## Configuration

| Where | What | Default |
|---|---|---|
| `references/lexicon.json` > `thresholds.noun_stack_min` | nouns in a row that count as a stack | 3 |
| `references/lexicon.json` > `thresholds.late_verb_gap` | words between subject and verb | 7 |
| `references/lexicon.json` > `suppress` | verbs, noun-verbs, or phrases you've confirmed are false flags | empty |
| `references/lexicon.json` > `animate_verbs`, `animate_nouns`, `denominal_verbs` | the word lists behind the two lexicon-based features | see file |
| `scripts/friction_score.py` > `LONG_SENTENCE`, `LONG_PARAGRAPH` | 25 words, 5 sentences | constants, not config |
| `SFE_MODEL` environment variable | spaCy model to load instead of `en_core_web_md` | unset |

After editing the lexicon or the scorer, run the self-test. The fixtures are the answer key, and a counter with no outside standard drifts.

## Security

The scripts read the files that you name and write nothing. The one exception is `build_ngrams.py`, which writes only the output file that you give it with `-o`. They make no network calls and start no subprocesses. The only network step is the spaCy model download, which you run yourself.

When the skill runs inside an agent, the agent reads your text and may run these scripts. It may also append lines to `references/feedback-ledger.md` if the folder is writable. That ledger is plain text and ends up wherever the folder does, so don't let anyone paste anything confidential into it. In the test runs the agents often couldn't write to the skill folder, so the log lines went into the report instead.

Nothing here makes a document safe, accurate, or anonymous. It's just a style tool.

## Troubleshooting

**"spaCy is not installed", exit 2.** The import failed. Run `pip install spacy` in the same Python that runs the script, then the model download.

**"No English spaCy model found", exit 2.** Neither `en_core_web_md` nor `en_core_web_sm` loads. Run `python -m spacy download en_core_web_md`. A mistyped `SFE_MODEL` prints a warning and falls back to the default, so check stderr if results look odd.

**Score is zero but the text is still hard going.** Expected, sometimes. The parser misfiles some bare relatives ("a timeout the client never saw resolved") and loses the subject on very long sentences. A clean score means no countable hits. The skill ends with a hand-read for exactly this.

**`compare` reports a guard failure.** Read the line after FAIL. It names the cause: a number lost, content recall under 0.55, or length grown past 1.6 times. Put the offending sentence back and make a smaller edit.

**Self-test fails after you edited the lexicon.** Your change flagged, or stopped flagging, something a fixture protects. The failing fixture's name shows which.

**Different results on another spaCy model.** Parsers disagree. Swapping the large English model in cleared two known misses and broke two passing fixtures, which is why medium is the default.

## How well it works

All of this comes from model judges and a hand-counting model, not real readers. The full method, tables, and caveats are in [description.md](description.md). The short version, on 14 passages nothing was tuned on:

| Version | Blind ease (1 to 5) | Hand-counted constructions per 100 words |
|---|---|---|
| Original text | 2.29 | 2.06 |
| Plain prompt ("short sentences, ordinary words") | 4.07 | 1.49 |
| This skill, BLUNT | 4.79 | 0.42 |

Against the plain prompt, BLUNT won on ease in 10 passages, drew two, and lost two. It left 3.5x fewer hand-counted constructions than the plain prompt, and 4.6x fewer than the originals. The counts behind those ratios are small.

The counter on its own flattered the skill badly in early versions. The rewriting loop was rewarded for lowering the counter's number, and it lowered it. That's why the skill ends with a hand-read, and why the scorer's figures are never the headline.

`reader-test/` is the fix for the model-judge problem. It's an offline page that shows four shuffled versions of each passage and exports ratings, plus an organiser script that unblinds them. If real readers disagree with the judges, the readers win.

## Known limitations

- No real readers have rated anything yet.
- The parser is the weak link. All five known misses are parser failures.
- The meaning guard checks numbers, entity text, how many of the original's content words survive, and length. It doesn't check hedges or conditions. That's the keep list's job, and the keep list is an instruction to the agent, not code.
- Two of the six features use word lists where the paper used a trained classifier. Expect misses and some false flags.
- My thresholds are heuristics of my own: three-noun stacks, seven-word gaps, and 25-word sentences. The paper doesn't give them.
- BLUNT changes the most text, so it leans hardest on the keep list. Two small meaning complaints turned up in 14 passages.
- It's not an AI detector and can't say who wrote a text.
- Don't (just dont,' run it on fiction, verse, dialogue, quotations, or contract wording.

## Possible future work

Planned only in the sense that I want it, not promised. Real-reader ratings with the pack come first. After that, a better parser or a second-opinion pass for bare relatives. Last, a proper reference table for the sixth feature.

## Layout

```
SKILL.md 
The instructions an agent follows
README.md, description.md these docs
links-for-other-skills.md paste-in notes for the other end of each link scripts/friction_score.py the counter, compare, and meaning guard scripts/selftest.py runs tests/fixtures.json scripts/build_ngrams.py builds a reference word-pair table (optional)

references/lexicon.json editable word lists and thresholds references/feedback-ledger.md log of false flags, accepted fixes, misses, regressionstests/fixtures.json              
38 cases: 33 scored, five known misses
reader-test/index.html
Blind rating page for readers
reader-test/organiser/answer key and unblinding script (keep away from readers)
```

## Links to other skills

It calls humanizer afterwards for vocabulary tells. When humanizer or overwatch-prose-audit finds syntax problems rather than word problems, it hands the text back. For a restricted vocabulary it offers technical-clarity-engine. Dash checks pass to em-dash-calibration, and manual structure stays with professional-technical-writing.

`links-for-other-skills.md` has the note for the other side of each link. I haven't edited those skills.

## Licence

MIT Licence 

## Credits

The paper, spaCy, and the write-ups of the paper.i may have grabbed a few quotes 
