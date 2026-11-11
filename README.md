# SihtiSR — put the residue back

Sihti stripped the fine residue off an image. This runs it the other way:
take a small image, enlarge it, and **add back a predicted residue**.

**Live page:** https://anttiluode.github.io/SihtiSR/ (turn on GitHub Pages: Settings → Pages → main / root).
Or just open `index.html` from disk. Open an image or use the webcam.

## How it works (three steps, about 150 lines)

1. **Sort — hand-written.** Every 5×5 low-res patch gets a bucket number from its
   structure tensor: edge angle (24) × edge strength (3) × coherence (3) = 216 buckets.
   No k-means, no training. Just a formula.
2. **Solve — closed form.** Each bucket owns one 25→4 linear map (one low-res pixel
   becomes a 2×2 block). The maps come from ridge regression: one matrix solve per
   bucket. No backprop, no epochs. Training takes about 25 seconds on a CPU.
3. **Lock — hand-written.** The 2×2 block is shifted so it averages back to the input
   pixel exactly. The output never contradicts what was measured.

The whole model is **21,600 numbers** (`model.json`, about 200 kB as text).

## Result

×2, grayscale PSNR, trained on the 200 BSDS500 train photos, scored on the 200
BSDS500 test photos it never saw. Low-res = 2×2 mean of the original.

| method | PSNR (dB) |
|---|---|
| bicubic | 28.56 |
| Lanczos | 28.81 |
| one linear filter for all patches | 29.09 |
| **SihtiSR (sorted)** | **29.45** |
| SihtiSR + lock | 29.45 |

Better than Lanczos on all 200 test images. Gain over bicubic: +0.89 dB.

## What is and is not here

- The sorting adds +0.36 dB over the best single filter. That is the part that matters.
- The lock adds nothing measurable; the maps already average back almost exactly.
  It stays because it is a guarantee, not because it scores.
- This is not new. Sorting patches by gradient angle/strength/coherence and giving
  each bucket a least-squares filter is **RAISR** (Romano, Isidoro, Milanfar, Google, 2016).
  This repo is a small version of that idea, arrived at from the Sihti residue side.
- It is far below modern deep SR networks (roughly 2–3 dB below on comparable tests).
  What it has instead: tiny, readable, trained in seconds, runs in a browser tab.
- The numbers hold for the 2×2-mean shrink it was trained on. A real photo was blurred
  by a lens and compressed by JPEG, so in "upscale" mode the gain is smaller and
  the added detail is a prediction, not a measurement. ×4 is the ×2 step applied twice.
- "Hand-written" here means the sorting rule and the lock. The 21,600 filter numbers are
  solved from data, not written by hand as in Moon.

## Run

```
pip install numpy pillow
python sihtisr.py up photo.jpg bigger.png        # x2
python sihtisr.py up photo.jpg bigger.png 2      # x4
python sihtisr.py train path/to/BSDS500/data/images   # rebuild model.json + results.json
```

Files: `sihtisr.py` (train + apply), `sihti.js` (same apply step for the browser,
checked equal to the Python to 1e-7), `model.json` / `model.js` (the numbers),
`index.html` (the page), `results.json` (the table above).
