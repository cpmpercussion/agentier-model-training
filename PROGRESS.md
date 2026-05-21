# Agentier model training — progress

_Last updated: 2026-05-22_

> IMPSY's `dataset` and `train` commands now accept `-O/--destination`, so
> outputs can be written directly into this repo — no post-hoc moves needed.

## Goal
Collect IMPSY interaction logs from the Agentier instrument (impsypi.local) and
train an MDRNN model on them.

## Done so far

1. **Logs copied** from `pi@impsypi.local:impsy/logs/` into `./logs/`
   via rsync. 7 log files, ~8.4 MB, 60,007 rows total.
   - The 1-line file `2026-01-17T00-38-24-9d-mdrnn.log` (failed start) was deleted.
   - All rows are `interface` source (live human input) — no `rnn` rows, so no
     source filtering was needed.
   - Note: impsypi.local SSH host key had changed (Pi reflashed); user fixed
     known_hosts and set up key login.

2. **Dataset built** with IMPSY, saved into this repo:
   ```
   poetry -C /Users/charles/src/impsy run python -m impsy dataset \
     -D 9 \
     -S /Users/charles/src/agentier-model-training/logs \
     -O /Users/charles/src/agentier-model-training/datasets
   ```
   Output: `./datasets/training-dataset-9d.npz`
   (60,000 interactions, 7 performances, ~7.4 hrs of playing, dimension 9, 471 KB).

   The suspiciously round 60,000 is coincidence + structure: the 7 logs hold
   60,007 raw rows, and `transform_log_to_sequence_example` drops the first
   row of each log (no prior timestamp to compute `dt` against), so
   60,007 − 7 = 60,000 exactly.

   To rebuild from the logs in this repo, re-run the command above — `-O`
   writes the npz straight here, overwriting the existing file.

3. **Initial training run** completed with the default `s` (small) model:

   ```
   mkdir -p /Users/charles/src/agentier-model-training/models
   poetry -C /Users/charles/src/impsy run python -m impsy train \
     -D 9 -M s \
     -S /Users/charles/src/agentier-model-training/datasets/training-dataset-9d.npz \
     -O /Users/charles/src/agentier-model-training/models \
     > /Users/charles/src/agentier-model-training/training.log 2>&1
   ```

   - Model size: **`s` (small)** — good fit for Raspberry Pi deployment.
   - 2 LSTM layers × 64 units, 5 mixtures, 58,143 params.
   - Training shape: `(59650, 50, 9)` (sequence length 50, dim 9).
   - Wallclock ~26 s/epoch, ~16 min total.

   **Result: early-stopped at epoch 36/100.**
   - Best `val_loss` = **13.00 at epoch 26** (next best: 13.04 @ ep19, 14.27 @ ep23).
   - Train loss kept dropping (−7.6 → −9.8) while val plateaued/diverged — classic
     overfit, so the default `--patience 10` triggered after 10 epochs without improvement.

   Artifacts written to `./models/`:
   - `musicMDRNN-dim9-layers2-units64-mixtures5-scale10-ckpt.keras` (727K) — **best checkpoint** (use this for deployment)
   - `musicMDRNN-dim9-layers2-units64-mixtures5-scale10.keras` (268K) — final-epoch model
   - `musicMDRNN-dim9-layers2-units64-mixtures5-scale10.tflite` (249K) — Pi-deployable

## Next step — long training run

The default run hit a val_loss wall around epoch 26 with `--patience 10`.
To see if it can break past 13.00, try a longer run with much more patience
(and consider raising `-N` so the cap isn't a constraint):

```
poetry -C /Users/charles/src/impsy run python -m impsy train \
  -D 9 -M s \
  -S /Users/charles/src/agentier-model-training/datasets/training-dataset-9d.npz \
  -O /Users/charles/src/agentier-model-training/models \
  -P 50 -N 500 \
  > /Users/charles/src/agentier-model-training/training-long.log 2>&1
```

- `-P 50` raises patience from 10 → 50 epochs.
- `-N 500` raises the epoch cap from 100 → 500.
- If the wall really is overfit (not under-training), bigger patience won't
  help on its own — also worth trying a larger model (`-M m` or `-M l`) or
  a different batch size (`-B`).
- The artifact filenames are the same shape as before, so this run will
  **overwrite** the existing `./models/musicMDRNN-…` files. Move/rename
  the current ones first if you want to keep them for comparison.

## Environment notes
- IMPSY repo: `/Users/charles/src/impsy` (branch `main`, v1.0.1).
- Dependencies installed via `poetry install` (poetry env already created).
- Run impsy as `poetry run python -m impsy ...` (the `impsy` console script
  is not on PATH).

## Deployment
- For Pi use: copy `./models/musicMDRNN-dim9-layers2-units64-mixtures5-scale10.tflite`
  (built from the best checkpoint) to the impsypi.
- For local sanity check: `impsy test-mdrnn` against the `-ckpt.keras` file.
