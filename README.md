# NETRA

**Near-miss Enabled Traffic Risk Analytics** — Smart India Hackathon 2026

Road hotspots in India are identified only after crashes accumulate, and crash
records are sparse and district-level. A junction can stay dangerous for years
without generating enough recorded fatalities to trigger intervention.

Crashes are rare. Near-misses are constant. **NETRA measures the common thing.**

It watches recorded traffic footage, works out where every vehicle actually is
in **metres on the road surface** rather than pixels on a screen, and computes
how many seconds separate each pair of vehicles from occupying the same space.
Conflicts below 1.5 s are recorded; below 0.8 s they are severe. Each one leaves
as a ~350-byte record. The video never leaves the camera.

The specification lives in `NETRA-PRD.md` (whole project) and
`NETRA-PRD-edge-pipeline.md` (the edge chain, owners A and B).

---

## What runs today

The edge chain — video in, validated conflict events out — is built and
verified against committed fixtures. **No demo video and no detector are needed
to run it.**

```bash
pip install numpy PyYAML requests pytest      # that is the whole core

python -m edge.run_pipeline --dry-run         # the full chain, on fixtures
python -m pytest                              # 70 tests
```

The dry run projects 567 pixel-space track rows into metres, finds two
encounters, emits one severe and one conflict event, and queues them in the
offline buffer. It touches no network and needs no OpenCV.

### The pipeline, stage by stage

| Stage | Module | In | Out | Owner |
|---|---|---|---|---|
| M2 | `edge.track.run_track` | video | `tracks_px.jsonl` | A |
| M1 | `edge.calibration.project` | `tracks_px.jsonl` | `tracks_m.jsonl` | B |
| M3 | `edge.conflicts.run_conflicts` | `tracks_m.jsonl` | `events.jsonl` | B |
| M7 | `edge.emit.uploader` | `events.jsonl` | `POST /api/events` | B |

Each runs standalone, so a fixture can be dropped in at any seam:

```bash
python -m edge.calibration.project \
    --tracks fixtures/tracks_px.sample.jsonl \
    --calib  fixtures/calibration.json \
    --out    out/tracks_m.jsonl

python -m edge.conflicts.run_conflicts \
    --tracks out/tracks_m.jsonl \
    --calib  fixtures/calibration.json \
    --out    out/events.jsonl --print-events
```

With a demo clip and the detector extras (`pip install ultralytics
opencv-python lap`), the whole chain is one command:

```bash
python -m edge.run_pipeline --video data/junction.mp4 \
    --calib out/calibration.json --emit http://localhost:8000/api/events
```

---

## Two things worth seeing

**Suppression rule 2.** A naive TTC implementation on Indian urban footage
emits hundreds of conflicts a minute and every one is garbage. The largest
single category is motorcycles filtering between cars — genuinely close,
entirely routine, and marked by no human labeller. The same fixture, twice, one
flag apart:

```bash
python -m edge.conflicts.run_conflicts --tracks fixtures/lane_split.jsonl \
    --calib fixtures/calibration.json
# 0 events, 212 pair-frames removed by rule 2

python -m edge.conflicts.run_conflicts --tracks fixtures/lane_split.jsonl \
    --calib fixtures/calibration.json --set suppression.lane_splitting=false
# 96 conflict readings, 2 severe events, all of them nonsense
```

All six suppression rules toggle independently, so what each removes can be
shown rather than asserted.

**Debouncing.** At 25 FPS a two-second encounter produces about fifty conflict
readings. Emitted raw, "we detected 200 conflicts" means "we detected four
conflicts, fifty times each". One encounter produces exactly one event,
carrying the minimum TTC observed. The run summary prints the collapse ratio,
because "how did you count?" is a question worth having an answer to.

---

## Repository layout

```
contracts/     frozen schemas, F, sign-off required   [written]
fixtures/      append-only sample data                [written]
edge/
  common/      config, JSONL, geometry, event         A+B  [done]
  calibration/ M1 homography and projection           B    [done]
  detect/      M2 YOLOv8n                             A    [needs a clip]
  track/       M2 ByteTrack                           A    [needs a clip]
  gate/        M4 motion gate                    P2   A    [needs a clip]
  conflicts/   M3 TTC, PET, suppression, debounce     B    [done]
  emit/        batching, SQLite buffer, upload        B    [done]
  norms/       M5 self-calibrating norms         P1   C    [stub]
bench/         M4 benchmark harness             P2   A    [needs a clip]
server/        M6 M7 M8 ingest, enrich, narrate       D, F [stub]
web/           M11 dashboard                          E    [stub]
eval/          M9 ground truth, M10 baseline          C    [stub]
demo/          script and rehearsal assets            F    [stub]
tests/edge/    70 tests, no detector needed           A+B  [done]
```

**You may only edit files inside your own directories.** A change anywhere else
goes through that owner. `contracts/` needs the integration owner's sign-off.
Every stub directory has a README naming its owner, its contract, and its first
file.

---

## Conventions

Full detail in `CLAUDE.md`. The four that cause silent bugs when broken:

- **SI internally.** Metres, seconds, m/s. km/h only at JSON serialisation.
- **`t` is seconds from video start.** Wall-clock is derived only at emission.
  Never mix them.
- **Bottom-centre of the bounding box** is the ground contact point, never the
  centroid. The homography maps the road surface; a centroid floats above it,
  and the error scales with vehicle height.
- **No tuned constant is hardcoded.** It lives in `edge/config.yaml` or it is a
  bug. Override at the CLI with `--set section.key=value`.

Two rules that are not conventions but constraints:

- **`severity` is derived from `ttc_s`** and cannot be set by hand.
- **There is no blame or fault field, and none may be added.** We describe what
  happened; we do not assign responsibility, because we cannot verify it and
  being wrong harms a real person.

---

## On the hardware claim

We have no Raspberry Pi, no Hailo accelerator and no camera. So `bench/`
measures a **laptop CPU proxy**, and every table it produces carries that label
— hardcoded in the writer, so no run can emit an unlabelled figure. The Hailo
figure is stated as a vendor-rated design target, never as a measurement.

The harness is itself the deliverable. It runs the day a board arrives, and
offering it is a stronger position than a number nobody can check.
