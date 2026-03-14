# autoresearch

Autonomous experiment loop to minimize `val_bpb` in `train.py`.

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar12`). The branch `autoresearch/<tag>` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from current master.
3. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data prep, tokenizer, dataloader, evaluation. Do not modify.
   - `train.py` — the file you modify. Model architecture, optimizer, training loop.
4. **Verify data exists**: Check that `~/.cache/autoresearch/` contains data shards and a tokenizer. If not, tell the human to run `modal run modal_run.py::prepare`.
5. **Initialize results.tsv**: Create `results.tsv` with just the header row. The baseline will be recorded after the first run.
6. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, kick off the experimentation.

## Experimentation

Each experiment runs on a single GPU. The training script runs for a **fixed time budget of 5 minutes** (wall clock training time, excluding startup/compilation). You launch it as:

```bash
set +e
modal run modal_run.py::train > run.log 2>&1
code=$?
echo "exit_code: $code" >> run.log
set -e
```

**What you CAN do:**
- Modify `train.py` — this is the only file you edit. Everything is fair game: model architecture, optimizer, hyperparameters, training loop, batch size, model size, etc.

**What you CANNOT do:**
- Modify `prepare.py`. It is read-only.
- Install new packages or add dependencies.
- Modify the evaluation harness. The `evaluate_bpb` function in `prepare.py` is the ground truth metric.

**The goal is simple: get the lowest val_bpb.** Everything is fair game. The only constraint is that the code runs without crashing and finishes within the time budget.

**VRAM** is a soft constraint. Some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Removing something and getting equal or better results is a great outcome. A 0.001 val_bpb improvement from deleting code? Definitely keep. A 0.001 improvement that adds 20 lines of hacky code? Probably not worth it.

**The first run**: Always establish the baseline first by running the training script as is.

## Logging results

Log every experiment to `results.tsv` (tab-separated). Do not commit this file — leave it untracked.

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	LR 0.04->0.08; overshot, bounded [0.04, 0.08]
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

Use 0.000000 / 0.0 for crashes. **In descriptions, record bounds learned** — when a discard narrows a parameter's useful range, note it. This prevents re-running dead ends.

## The experiment loop

LOOP FOREVER:

1. Review `results.tsv` and `notes.md`. Be intentional about what you're testing and why.
2. Hack `train.py` with the experimental idea.
3. Run the experiment (see run command above).
4. Read results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`
5. If empty, the run crashed. Run `tail -n 50 run.log` for the traceback. Fix if trivial, otherwise skip.
6. Record results in `results.tsv`.
7. If val_bpb improved: git commit with message `exp N: <what and why> (keep)`.
8. If equal or worse: `git reset` back to where you started. The discard lives only in `results.tsv`.

**Timeout**: If a run exceeds 10 minutes, kill it and treat as failure.

**Crashes**: If it's a dumb fix (typo, missing import), fix and re-run. If fundamentally broken, log "crash" and move on.

## Research discipline

Keep `notes.md` (max ~50 lines) as a running scratchpad. Update after each experiment:
- **Current best**: val_bpb and what config got there
- **Bounded parameters**: ranges explored and dead (e.g. "MATRIX_LR: optimal ~0.04 at 768-dim, bounded [0.03, 0.045]")
- **Key learnings**: 5-10 bullets of what works, what doesn't, and why
- **Regime changes**: when you change model size, depth, or batch size, note it — previously tuned hyperparameters may need re-validation at the new operating point

When choosing what to try next, draw from diverse sources: continue what's working, combine near-misses, try structural changes, re-read `train.py` for ignored knobs, or try something radical when stuck. **Throughput beats planning** — don't spend more than a minute deciding.

## NEVER STOP

Do NOT pause to ask the human if you should continue. The human may be asleep and expects autonomous progress. If you run out of ideas, think harder — re-read the code, try combining previous near-misses, try radical architectural changes. The loop runs until the human interrupts you, period.

~12 experiments/hour, ~100 overnight. Maximize throughput.
