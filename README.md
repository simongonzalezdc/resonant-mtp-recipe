# Multi-token prediction on a mini-PC: the measured +33-39% decode recipe

Notes for people serving local models on their own hardware: one llama.cpp
flag turns on the multi-token-prediction head already shipped inside the
Qwen3.8-27B GGUF everyone downloaded, and the paired measurements put decode
speed up `+33%` to `+39%` on the two founding 24 GB NVIDIA cards. The living
community table now spans `+33%` to `+145%` across 53 configurations on both GPU
vendors, with the largest relative wins on the bandwidth-poor mini-PC
parts. No new files, no conversion, no custom build.

These notes adapt the canonical community repo
[sudoingX/qwen38-mtp](https://github.com/sudoingX/qwen38-mtp) (Apache-2.0) —
the recipe, the founding paired A/B (opened hours after the model's
August 14, 2026 release), and the living table its contributors grew —
and add the bench
numbers from our own Strix Halo mini-PC
([KyaniteLabs/qwen38-27b-strix-halo](https://github.com/KyaniteLabs/qwen38-27b-strix-halo),
MIT). Every number keeps the validity label its source published with.
Nothing here re-measures or re-brands anything; if a number here ever
disagrees with its source, the source wins — flag an issue here or there.

Labels · What MTP is · The numbers · The recipe · Tuning the two knobs ·
On our bench: Strix Halo · What breaks · The honest gap · Reproduce it ·
Provenance, credits, license

---

## Read the labels first

The two sources publish at different evidence strengths, and every number
below keeps the class it was published with:

- **PAIRED** — same card, same GGUF, same config on both sides, one arm
  changed; medians of 3 runs x 3 prompts through a streaming client against
  a live server, warmup discarded, thinking off. The canonical repo's
  founding method, and the only class quoted as a headline here.
- **COMMUNITY** — a single-rig row contributed to the canonical living
  table by PR, config footnoted in the canonical README. Real measurements,
  per-rig methods; absolutes are not cross-comparable between rows.
- **SCREEN** — single-run screening arms. They show shape, never ranking.
- **PILOT** — `n=1` paired cells on our bench. A map, not a law.
- **ARTIFACT** — the number measures the bench, not the work. Carry the
  label or do not quote the number.

Arithmetic on measured inputs is labeled **ARITHMETIC** where it appears.

`Provenance: sudoingX/qwen38-mtp README, "The numbers" (method paragraph) + community-table footnotes; KyaniteLabs/qwen38-27b-strix-halo METHODOLOGY.md (evidence-class discipline).`

## What MTP is, in one paragraph

Qwen trained extra next-token heads — multi-token-prediction (MTP, `nextn`
layers) — into Qwen3.8, so the model can draft several tokens ahead of
itself. The quantizers kept those heads: unsloth's GGUFs carry the
`blk.*.nextn.*` tensors, and llama.cpp loads them but, without one flag,
ignores them. With the flag, the server drafts a couple of tokens with the
built-in head while the main model verifies them in one pass — a verified
draft costs a fraction of a full forward pass, and llama.cpp has shipped
this (`draft-mtp` speculative decoding) since PR #22673 in July 2026. The
model released August 14, 2026 with the head already inside the weights;
the flag that connects it cost nothing. The canonical repo's title says it
straight — "the flag was free the whole time" — and the numbers hold up.

`Provenance: sudoingX/qwen38-mtp README ("How it works", "The flag"); llama.cpp PR #22673 (July 2026).`

## The numbers

The claim in the title is the founding pair, and it verifies by simple
arithmetic: `41.3/31.0 = 1.332` and `50.9/36.7 = 1.387`. Same card, same
GGUF, same config both sides; live server, streaming client, every
generated token clocked, warmup discarded, medians of 3 runs x 3 prompts,
thinking off, 131K context resident, q4_0 KV cache, unsloth Q4_K_M. Serve
measurements, not llama-bench numbers.

| Card | Baseline | With the flag | Gain | Acceptance |
|---|---|---|---|---|
| RTX 3090 24 GB | `31.0` tok/s | `41.3` tok/s | **`+33%`** | `0.76-0.80` |
| RTX 5090 mobile 24 GB | `36.7` tok/s | `50.9` tok/s | **`+39%`** | `0.76-0.82` |

`Provenance: sudoingX/qwen38-mtp README, "The numbers" founding table (measured the night of the drop). VALIDITY: PAIRED.`

The founding pair is the floor, not the range. The canonical living table
(ten rows, nine contributors, NVIDIA and AMD) recomputes as:

| Card | Baseline | With flag | n-max | Gain | Acceptance | Contributor |
|---|---|---|---|---|---|---|
| RTX 3090 24 GB | `31.0` | `41.3` | 2 | `+33%` | `0.78` | @sudoingX |
| RTX 5090 mobile 24 GB | `36.7` | `50.9` | 2 | `+39%` | `0.79` | @sudoingX |
| RTX 4090 24 GB | `47.7` | `76.3` | 2 | `+60%` | `0.56` | @Spadav_ |
| RTX A6000 48 GB (Ada) | `26.7` | `52.5` | 2 | `+97%` | `0.54-0.98` | @lingster |
| RX 7900 XTX 24 GB | `30.7` | `43.9` | 2 | `+43%` | `0.60-0.95` | @Jqianggu |
| 2x RTX 3090 + 3090 Ti (TP) | `49.1` | `81.1` | 2 | `+65%` | `0.52-0.96` | @guilhermedemelocabral |
| RTX 4090 24 GB (UD-Q4_K_XL) | `36.1` | `74.8` | 2 | `+107%` | `0.56-0.94` | @rkvhtd |
| 2x RX 9070 16 GB (Vulkan) | `22.1` | `41.6` | 2 | `+88%` | `0.73` | @tomertec |
| AMD Radeon AI PRO R9700 32 GB | `27.0` | `43.3` | 2 | `+60%` | `0.60-0.94` | @ajnytebot |
| Ryzen AI Max+ 395 / Radeon 8060S | `11.5` | `23.7` | 2 | `+106%` | `0.52-0.94` | @shiwuxiu |

Gain is **ARITHMETIC** on the published medians (the canonical table ships
the tok/s columns; the percentages here are computed, `with/baseline`,
rounded to whole numbers). Every config footnote — quants, context, KV
type, VRAM, build, method per row — is in the canonical README.

`Provenance: sudoingX/qwen38-mtp README, "Community numbers" table + row footnotes (astrixed notes; per-row methods differ and are spelled out there). VALIDITY: COMMUNITY per row; Gain column ARITHMETIC.`

The mini-PC rows are the reason this flag matters to small-box owners: the
two bandwidth-poor parts (2x RX 9070, Ryzen AI Max+ 395 iGPU) roughly
double, because their *baseline* is unusually bad at batch-1 decode — the
flagged ceiling is the same absolute band as the desktop cards. The
canonical reading: if your card is bandwidth-poor at batch 1, the flag is
worth more to you than to a 3090 owner.

`Provenance: sudoingX/qwen38-mtp README, RX 9070 section ("the delta is much larger and the ceiling is the same"); Ryzen AI Max+ 395 section (+106% vs 11.5 tok/s spec-off). VALIDITY: COMMUNITY.`

## The recipe

One flag turns it on; the KV-cache flags make room for it. The canonical
launch command on a 24 GB card:

```bash
llama-server -m Qwen3.8-27B-Q4_K_M.gguf \
  -c 131072 -ngl 999 -fa 1 \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --spec-type draft-mtp --spec-draft-n-max 2 --parallel 1
```

- The spec flags are the recipe: `--spec-type draft-mtp
  --spec-draft-n-max 2 --parallel 1`.
- The KV flags matter on their own: without them, context creation fails
  past roughly 90K next to ~17 GB of weights; with them the full 262K
  window fits a 24 GB card at `22.2 GB` (drop `-c` to 262144 and remove
  the spec flags if you want maximum window instead of maximum speed).
- Weights: [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
  (the tensors that make this work are inside them); official model:
  [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) (Apache-2.0).

`Provenance: sudoingX/qwen38-mtp README, "The flag" + "Full launch command" + KV-cache paragraph; serve_mtp.sh in the canonical repo is this exact config.`

Canonical caveats, carried verbatim in meaning: `--parallel 1` is required
for now (single slot); prompt processing takes a small hit from
device-to-host embedding transfers; these are day-one llama.cpp speeds
through the `qwen3_5` code path and the floor should rise with upstream
work; your absolute numbers will differ with hardware, drivers, and
thermals — the deltas are the durable part.

`Provenance: sudoingX/qwen38-mtp README, "Caveats".`

## Tuning the two knobs

The recipe default (`n-max 2`) is right for mixed work, and two measured
sweeps show how to move it. Knob one is `--spec-draft-n-max`; knob two is
`--spec-draft-p-min`, a confidence gate (~`0.60`) that stops the head
drafting when it is unsure — which makes deeper drafts nearly free instead
of harmful. Acceptance decays as you draft deeper, and prose pays first.

On the 5090 mobile (PAIRED method), deeper loses:

| n-max | Overall median | Code prompts | Prose prompts | Acceptance |
|---|---|---|---|---|
| **2** | **`50.9`** | `56.4` | `42.5` | `0.76-0.82` |
| 3 | `48.3` | `59.4` | `37.9` | `0.68` |
| 4 | `47.3` | `60.2` | `33.4` | `0.65` |

`Provenance: sudoingX/qwen38-mtp README, "Tuning n-max". VALIDITY: PAIRED.`

On the A6000 48 GB, deeper wins up to 4 — the card absorbs the verification
cost before acceptance decay eats the win:

| n-max | Overall | Code (py) | Prose (mmap) | Code (bash) | Acceptance |
|---|---|---|---|---|---|
| 2 | `52.5` | `57.0` | `43.1` | `52.5` | `0.54-0.98` |
| 3 | `60.7` | `67.6` | `44.1` | `60.7` | `0.42-0.91` |
| **4** | **`64.1`** | `77.1` | `41.6` | `64.1` | `0.32-0.93` |
| 5 | `62.8` | `80.5` | `40.8` | `62.8` | `0.29-0.84` |
| 6 | `58.6` | `84.3` | `37.4` | `58.6` | `0.23-0.84` |

`Provenance: sudoingX/qwen38-mtp README, "A6000 48GB: n-max sweep". VALIDITY: COMMUNITY (single rig, probe medians + server-log acceptance).`

The gate (`--spec-draft-p-min`) makes depth affordable. On 2x RX 9070
16 GB at the full 262K window, ungated `n-max 4` was *worse* than
`n-max 2`; gated at `0.60` it wins, drafting ~`3.3` tokens per round at
`0.86` acceptance instead of exactly `2.0` at `0.73`:

| config | ~0 ctx | 16K | 65K | Acceptance |
|---|---|---|---|---|
| spec off | `22.1` | `21.0` | — | — |
| n-max 2, no gating | `41.6` | `37.2` | `28.0` | `0.73` |
| **n-max 4, p-min 0.60** | **`42.9`** | **`38.5`** | `27.4` | `0.86` |
| n-max 4, p-min 0.75 | `38.3` | `36.6` | **`28.9`** | `0.91` |
| n-max 4, p-min 0.80 | `40.7` | `35.5` | — | `0.90` |
| n-max 8, any p-min | `30.0-36.7` | `31.8-33.9` | — | `0.74-0.84` |

`Provenance: sudoingX/qwen38-mtp README, "2x RX 9070 16GB: n-max sweep, and the second knob nobody is turning" + its method note (local greedy streaming harness, max 1200 tokens — not probe.py; n-max 2 and p-min 0.60 arms medians of 3; spec-off, p-min 0.75/0.80 and n-max 8 arms single screening runs). VALIDITY: the two headline arms PAIRED-class medians; the rest SCREEN.`

That same section carries the method lesson every tuner should keep: an
`n=1` screen crowned `p-min 0.75`; medians of 3 flipped the winner to
`0.60`. The noise band was around `+/-2` tok/s — wide enough to invert a
ranking. Do reps before you deploy a config.

`Provenance: sudoingX/qwen38-mtp README, RX 9070 section ("N=1 crowned the wrong arm"). VALIDITY: method statement, not a measurement.`

The three rules the community table found, in the canonical wording:
the `n-max` sweet spot is card-dependent (24 GB cards peak at 2, the
48 GB A6000 at 4); `--spec-draft-p-min` is the second knob; the gain
scales with generation length — under ~400 output tokens the spec overhead
can dominate, while a 4096-token run on the same 4090 setup measured
`+60%`.

`Provenance: sudoingX/qwen38-mtp README, "The three rules the community found" + RTX 4090 row footnote (4096-token curl method). VALIDITY: rules are summaries of the COMMUNITY rows cited there; +60% is COMMUNITY, single rig.`

## On our bench: Strix Halo

Our own box is a Ryzen AI Max+ 395 mini-PC (Radeon 8060S iGPU, 96 GB
unified LPDDR5X with 64 GB GTT visible to the GPU, ROCm/HIP, llama.cpp
b10435-era build, unsloth UD-Q4_K_XL)
— same chip family as the community table's Ryzen row, different backend
and deeper spec stack. Two things carry over from the canonical recipe
(the head ships in the weights; `draft-mtp` is the foundation), and two
are different here (the win is measured as wall-time and time-per-task,
and an ngram drafter stacks on top of the MTP head for free).

Even before the ngram stack, deeper drafts scaled on this chip. The
MTP-only ladder from its first night (count-to-30 bench, cold, measured
2026-08-15):

| Config | Cold tok/s | Context |
|---|---|---|
| MTP n=2 (day-0 default) | `22` | 16k |
| MTP n=6 | `32.7` | 32k |
| MTP n=9 | `55.7` | 32k |

`Provenance: KyaniteLabs/qwen38-27b-strix-halo README, "The numbers (measured 2026-08-15, single box, reproducible via components/bench/)". VALIDITY: DIRECTIONAL (single box, single bench, n not stated per row).`

The production champion stacks `ngram-mod` on top of the MTP head
(`--spec-type draft-mtp,ngram-mod --spec-draft-n-max 12
--spec-ngram-mod-n-min 24`): ngram drafts are prompt-derived and cost zero
bandwidth, and they stack losslessly on top of MTP heads because the main
model verifies everything either way.

| Spec state | Mean wall per 200-token answer |
|---|---|
| draft-mtp + ngram-mod, n-max 12 (= serving) | **`15.1 s`** |
| spec off | `17.8 s` |
| ngram only | `17.7 s` |
| mtp only | `15.1 s` |

MTP does the work on prose (mtp-only ties the full stack at `15.1 s`);
ngram adds nothing there but is free when it misses, and it owns the
repetition-heavy regime: the five-arm sweep measured the c30 count-to-30
task at `1.5 s` median with the stack vs `7.5 s` spec-off — spec decoding
is worth `5x` on repetition-heavy tasks, and the n-max 12 cap cost nothing
(`1.5 s` capped vs `1.5 s` uncapped).

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/config-27b-2026-08-21/ (4-way paired walls) + results/spec-sweep-2026-08-19/ (five-arm sweep: uncapped 1.5s/10.8, capped-n12 1.5s/11.0, mtp solo 1.9s/10.7, ngram solo 3.4s/11.3, spec off 7.5s/11.1). VALIDITY: PILOT (n=1 per cell, paired rows).`

The honest band structure on this box, which the canonical mini-PC row
agrees with in shape: cold one-shot `59.7` tok/s; warm back-to-back
`148-163` is an **ARTIFACT** (ngram speculation replays the bench's own
repetition — in production it appears only on genuinely repetitive
output, `72-133` tok/s); real conversational traffic is `11-24` tok/s
(creative long-form ~`11-14`, structured/code `29-40`). That is
memory-bandwidth physics for a 27B dense model on LPDDR5X; the flag moves
the wall-clock of real answers (`15.1 s` vs `17.8 s` per 200 tokens, and
`7.9-14.3 s` per correct task, median `11.3`, on the time-per-task
battery), it does not change the physics.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo README ("SPEED SHEET", "Read the warm numbers honestly", quant table) + re-baseline update 2026-08-16 (tpt band). VALIDITY: band structure CLEAN as labeled bands; individual rates DIRECTIONAL; tpt DIRECTIONAL (n=3, thermal-dependent band).`

## What breaks

Every hazard below was measured, not imagined. This is the part to read
before redeploying a rig.

- **Warm numbers are an artifact class.** Any `148-163`-shaped warm
  number under ngram speculation is repetition-assisted and only exists on
  genuinely repetitive output. The label is part of the number.
  `Provenance: KyaniteLabs/qwen38-27b-strix-halo README ("Read the warm numbers honestly"; ARTIFACT class labeled since 08-16).`
- **Short generations can net negative.** Under ~400 output tokens the
  spec overhead can dominate; the gain scales with generation length
  (`+60%` at 4096 tokens on the same setup that gains less on short
  prompts). `Provenance: sudoingX/qwen38-mtp README (community rule 3 + 4090 footnote).`
- **Acceptance decays with depth, and prose pays first.** Every sweep in
  the canonical repo shows acceptance falling monotonically with `n-max`
  (`0.76 -> 0.65` on the 5090; prose falling while code rises on the
  A6000; prose-only acceptance `29.4%` at n-max 4 on the R9700).
  `Provenance: sudoingX/qwen38-mtp README (all four sweeps + R9700 section). VALIDITY: PAIRED/COMMUNITY per sweep.`
- **`n=1` crowns the wrong arm.** The single-run screen inverted under
  medians of 3. `Provenance: sudoingX/qwen38-mtp README (RX 9070 section).`
- **A second trained drafter loses on unified memory.** On our box, a
  DSpark 1.36B BF16 drafter (acceptance `0.91`) still netted ~`32` tok/s
  vs the champion's `59.6` cold — its forward pass competes with the 27B
  verifier for the same memory bus; a cross-gen DFlash drafter hit
  acceptance `0.063` on creative text. The built-in MTP head wins because
  it is already in the weights, and ngram drafts win because they are
  free. `Provenance: KyaniteLabs/qwen38-27b-strix-halo docs/findings.md ("Neural drafters lose on unified memory", 4-arm A/B, mirror-validated). VALIDITY: DIRECTIONAL (single box, mirrored arms).`
- **Quantizing a *separate* draft model made it slower than no draft.**
  On our second lane (LFM2.5-2.6B + DSpark), the Q8_0 draft ran `21%`
  below baseline with essentially the same acceptance (~`0.80`) —
  dequantizing the small
  network every drafting step cost more than the memory saved. The
  built-in MTP head avoids this entire class: there is no separate draft
  to quantize. `Provenance: blog "The One-Line Bug That Crashed Our Fast Lane" (kyanitelabs.tech/blog/the-one-line-bug). VALIDITY: PILOT (n=3 arms, one prose prompt).`
- **Speculative decoding is not universally positive.** On the same box
  with a 35B MoE resident, the trained draft head accepted ~`33%` and made
  generation ~`9%` slower; verdict OFF, measured twice. Memory bandwidth
  went to the two resident models. `Provenance: blog "Two models, one $1,400 mini-PC" (kyanitelabs.tech/blog/two-models-one-mini-pc-paired-numbers), "What did NOT work". VALIDITY: DIRECTIONAL (measured twice, paired).`
- **Single-slot only, for now.** `--parallel 1` is required in the
  canonical recipe, and prompt processing takes a small unquantified hit.
  `Provenance: sudoingX/qwen38-mtp README ("Caveats").`
- **Build era matters.** Spec-decode behavior moved between llama.cpp
  builds during our arc; the community rows span b10360-b10437. Pin your
  build and keep the A/B paired within one build.
  `Provenance: KyaniteLabs/qwen38-27b-strix-halo METHODOLOGY.md ("Build-era matters"); sudoingX/qwen38-mtp row footnotes (b10360/b10426/b10433/b10437).`
- **Backend changes the answer.** On our box, Vulkan roughly halved
  decode under the full spec stack vs ROCm (`48.8-50.2` vs `94.7-98.5`
  warm c30) — check the backend before copying a config across machines.
  `Provenance: KyaniteLabs/qwen38-27b-strix-halo docs/findings.md ("Build/backend traps on gfx1151"). VALIDITY: DIRECTIONAL.`

## The honest gap

- **What "+33-39%" covers — carried prominently, because the claim line
  is narrower than the table.** The title's number verifies exactly for
  the two founding 24 GB NVIDIA rows (PAIRED, the only headline class).
  It is *not* the whole range: the living table recomputes to `+33%` to
  `+107%`, and the two mini-PC-class rows roughly double. The canonical
  repo's own headline uses the full range; its one-line description uses
  the founding pair. Both agree with the table; quote the label with the
  number.
- **A stale count in the canonical intro.** The canonical README's
  opening line says "ten machines, eight contributors" while its own
  table carries ten rows from nine distinct handles; these notes count
  from the table and flag the intro line rather than smooth it.
- **The canonical repo ships no raw logs.** The receipts are the README
  tables, the per-row method footnotes, and `probe.py` itself. The
  founding rows are `3x3` medians; most community rows are single-rig
  PRs with varying methods (three rows declare stock `probe.py`, one a
  different local harness, one `probe.py` plus a 4096-token curl). n is
  small everywhere; no cross-machine replication of any single cell
  exists.
- **Output identity under MTP was not A/B'd in the canonical repo.** The
  mechanism is target-verified drafting, and a companion rig measured
  byte-identical greedy output for a *neural* drafter (different
  mechanism, different model), but no canonical MTP row publishes an
  output-equality check. We do not claim it here.
- **Prompt-processing cost is acknowledged, not quantified**, in the
  canonical caveats. The canonical repo publishes no TTFT or power
  numbers (our bench has first-token and thermal notes, but no power
  draw either). Decode tok/s is not task latency — on our box the
  quant-ladder verdict closed on time-per-task precisely because decode
  tok/s lies about finishing work.
- **Prose pays for code's lunch.** Every deep-draft win is code-side;
  prose-only arms fall from the first step of depth in every sweep. A
  prose-heavy deployment should run `n-max 2` and stop.
- **One bench, one quant family on our side.** Our Strix Halo numbers are
  single-box, unsloth dynamic quants, one build era; the warm band is
  labeled ARTIFACT and the walls are PILOT (`n=1` per cell). The
  canonical methodology says to treat absolute values as this-box
  measurements and compare only within-condition.

`Provenance: this section is the discrepancy and uncertainty inventory for the two sources cited throughout; each bullet names its source. VALIDITY: statements about what is absent.`

## Reproduce it

The canonical repo is the instrument. The whole method is one paired run:
serve baseline, measure; add the flag, measure again; the delta is the
number.

```sh
git clone https://github.com/sudoingX/qwen38-mtp
cd qwen38-mtp

# the measured 24GB config, verbatim (edit the model path)
bash serve_mtp.sh /path/to/Qwen3.8-27B-Q4_K_M.gguf

# the streaming probe behind every founding number:
# 3 prompts x 3 runs, warmup discarded, per-prompt medians
python3 probe.py                  # defaults to http://127.0.0.1:8080
python3 probe.py http://127.0.0.1:8090
```

Run `probe.py` once against a baseline serve and once with the flag, same
everything otherwise. That pairing is the whole method — the canonical
table's row footnotes ask contributors for exactly it: the A/B, the
config footnote, both columns.

For independent instrumentation on your box, the same measurement kit this
doc's sibling gifts ship runs as a stand-alone add-on:
[resonant-stack-bench](https://github.com/simongonzalezdc/resonant-stack-bench)
(MIT) — prefill/decode tok/s with true-token prompt sizing and an
ambient-noise A/B check, plain JSON files out, no dashboards:

```sh
git clone https://github.com/simongonzalezdc/resonant-stack-bench
cd resonant-stack-bench
python3 server.py     # http://127.0.0.1:4889

curl -s -X POST http://127.0.0.1:4889/ -H 'Content-Type: application/json' \
  -d '{"method":"stackbench.run","params":{"endpoint_url":"http://127.0.0.1:8080","prompt_tokens":[8000],"rounds":3}}'
curl -s -X POST http://127.0.0.1:4889/ -d '{"method":"stackbench.results"}'
```

Run it baseline, add `--spec-type draft-mtp --spec-draft-n-max 2
--parallel 1`, run it again. Post your row to the canonical community
table by PR — that table is the living record, and rows inside or outside
the published bands are both wanted.

`Provenance: sudoingX/qwen38-mtp README ("Measure it yourself", "Community numbers — ran the A/B on your card? Open a PR and add a row") + serve_mtp.sh + probe.py; resonant-stack-bench README (usage).`

## Provenance, credits, license

Everything measured here stands on the canonical community repo
[sudoingX/qwen38-mtp](https://github.com/sudoingX/qwen38-mtp) (Apache-2.0)
— recipe, founding A/B, `probe.py`, `serve_mtp.sh`, and the living table
built by its contributors: @sudoingX (founding rows), @Spadav_,
@lingster, @Jqianggu, @guilhermedemelocabral, @rkvhtd, @tomertec,
@ajnytebot, @shiwuxiu. The stack stands on llama.cpp (MIT) — the
`draft-mtp` implementation (PR #22673) and the engine every row runs on —
and on the unsloth GGUFs that keep the MTP tensors, and Qwen3.8-27B
(Qwen team, Apache-2.0).

The Strix Halo bench numbers are our own published work:
[KyaniteLabs/qwen38-27b-strix-halo](https://github.com/KyaniteLabs/qwen38-27b-strix-halo)
(MIT — README, `docs/findings.md`, `results/`), with the two companion
articles ["The One-Line Bug That Crashed Our Fast
Lane"](https://kyanitelabs.tech/blog/the-one-line-bug) and ["Two models,
one $1,400 mini-PC"](https://kyanitelabs.tech/blog/two-models-one-mini-pc-paired-numbers)
cited in the hazard stamps above.

These notes are an adaptation written for people building on local
hardware: the canonical repo is linked throughout and wins any
disagreement; the adapted portions carry Apache-2.0 attribution in the
NOTICE file; the writeup itself is MIT (see `LICENSE`). The numbers,
labels, and caveats are the sources'; the arrangement errors, if any, are
ours — flag an issue in either place.
