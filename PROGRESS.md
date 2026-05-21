# Agentier model training — progress

_Last updated: 2026-05-21_

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
     -D 9 -S /Users/charles/src/agentier-model-training/logs
   ```
   Output: `./datasets/training-dataset-9d.npz`
   (60,000 interactions, 7 performances, ~7.4 hrs of playing, dimension 9, 471 KB).

   Note: `-C` changes poetry's cwd, so the npz initially lands in
   `/Users/charles/src/impsy/datasets/` and needs to be moved here.

## Next step — NOT YET RUN

Training was about to start (user paused to switch Claude account). Run it with:

```
poetry -C /Users/charles/src/impsy run python -m impsy train \
  -D 9 -M s \
  -S /Users/charles/src/agentier-model-training/datasets/training-dataset-9d.npz \
  > /Users/charles/src/agentier-model-training/training.log 2>&1
```

- Model size chosen: **`s` (small)** — good fit for Raspberry Pi deployment.
- Early stopping is on by default.
- Watch progress in `./training.log`.

## Environment notes
- IMPSY repo: `/Users/charles/src/impsy` (branch `main`, v1.0.1).
- Dependencies installed via `poetry install` (poetry env already created).
- Run impsy as `poetry run python -m impsy ...` (the `impsy` console script
  is not on PATH).

## After training
- Trained model lands in IMPSY's models directory — copy it back to the
  impsypi for use, or test locally with `impsy test-mdrnn`.
