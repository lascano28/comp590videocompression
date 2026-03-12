# Assignment 1: Lossless Video Compression

## Overview

This program compresses grayscale video frames losslessly using arithmetic coding with adaptive probability models. Each pixel is encoded one at a time in row-major order.

## Compression Strategy

### Temporal Prediction (Residual Coding)

Rather than encoding raw pixel values directly, the encoder exploits **temporal redundancy** between consecutive frames. For each pixel at position `(r, c)`, the encoder computes a **temporal residual**:

```
pixel_difference = (current_frame[r][c] - prior_frame[r][c] + 256) % 256
```

This residual is what gets encoded. In typical video, most pixels change very little between frames, so residuals cluster heavily near 0. Arithmetic coding can then assign very short codes to small residuals, achieving good compression.

The prior frame is initialized to uniform mid-gray (y = 128) before the first frame is processed.

### Context-Adaptive Arithmetic Coding (256 Contexts)

The key innovation over a naive single-context approach is using **256 adaptive arithmetic coding contexts**, one for each possible prior-frame pixel value (0–255).

The intuition: the distribution of temporal residuals is not uniform across all pixels — it depends strongly on what the pixel's prior value was:
- A pixel at intensity 0 (black) can only stay the same or brighten → residuals are asymmetrically biased toward 0 and small positive values.
- A pixel at intensity 255 (white) can only stay the same or darken → residuals are asymmetrically biased toward 0 and values near 255 (i.e., small negative changes in modular arithmetic).
- A pixel at intensity 128 (mid-gray) can change in either direction, but still tends to be stable.

By maintaining a separate probability model per prior-pixel value, each context quickly learns the characteristic residual distribution for that intensity level. The arithmetic coder then assigns shorter codes when the residual matches the context's expected distribution.

**Context selection:**
```rust
let ctx = prior_frame[pixel_index] as usize;  // 0..=255
enc.encode(&pixel_difference, &pdfs[ctx], &mut bw);
pdfs[ctx].incr_count(&pixel_difference);
```

Because `ctx` is derived entirely from the prior frame (which both encoder and decoder possess identically), no side information needs to be transmitted — the decoder can reproduce the exact same context selection.

### Decoder Mirroring

The decoder mirrors the encoder exactly:
1. Compute `ctx = prior_frame[pixel_index]`
2. Decode the residual using `dec_pdfs[ctx]`
3. Update `dec_pdfs[ctx]`
4. Reconstruct the pixel: `(prior_frame[pixel_index] + decoded_residual) % 256`

### Compression Results

#### Strategy comparison on `bourne.mp4` (first 10 frames)

| Approach | Contexts | Compression Ratio |
|---|---|---|
| Single global context (baseline) | 1 | 2.36 |
| **Prior pixel value (this approach)** | **256** | **2.74** |

Compression ratio = original bits / compressed bits (higher is better).

#### Results across different videos (10 frames each)

To reproduce: `cargo run -p assgn1 -- -in data/<video> -count 10`

| Video | Content Type | Compression Ratio |
|---|---|---|
| bourne.mp4 | Action film (high motion) | 2.74 |
| news.mp4 | News broadcast (low motion) | 2.91 |
| rick_morty.mp4 | Animation (flat colors, low detail) | 9.31 |

Compression ratio = original bits / compressed bits (higher is better).

The results align with intuition: animation compresses dramatically better (9.31×) because flat-colored cartoon frames change very little between frames, producing residuals tightly clustered at 0. The news broadcast (2.91×) compresses slightly better than the action film (2.74×) due to less camera movement and fewer scene cuts. High-motion video has larger, more spread-out residuals that are harder to compress.

## Building and Running

```
cargo run -p assgn1                        # compress 10 frames, print ratio
cargo run -p assgn1 -- -check_decode      # verify lossless decode
cargo run -p assgn1 -- -verbose           # per-frame bit counts
cargo run -p assgn1 -- -count 20          # compress 20 frames
```

### All Options

| Option | Default | Description |
|---|---|---|
| `-verbose` / `-no_verbose` | no_verbose | Print per-frame info |
| `-report` / `-no_report` | report | Print final compression ratio |
| `-check_decode` / `-no_check_decode` | no_check_decode | Verify lossless decode |
| `-skip_count N` | 0 | Skip first N frames |
| `-count N` | 10 | Compress N frames |
| `-in path` | data/bourne.mp4 | Input video file |
| `-out path` | data/out.dat | Output compressed file |
