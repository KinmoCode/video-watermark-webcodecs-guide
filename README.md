# Client-side video watermark removal with WebCodecs

Reference notes for removing a visible corner watermark from an MP4 **entirely in the browser**: no server, no ffmpeg.wasm, no upload. The approach is the one used by [Free Video Watermark Remover](https://freevideowatermarkremover.com), whose exact client-side engine is published at [freevideowatermarkremover.com/engine/](https://freevideowatermarkremover.com/engine/).

The pipeline in one line:

```
File ─► MP4Box demux ─► VideoDecoder ─► ImageData ─► boundary-ring fill (per box) ─► VideoEncoder (H.264) ─► MP4 mux ◄─ AAC samples copied as-is
```

## Contents

| File | What it covers |
|---|---|
| [README.md](README.md) | The pipeline, the boundary-ring fill, temporal smoothing, bitrate choice, limits |
| [presets-aspect-ratios.md](presets-aspect-ratios.md) | Starting boxes for Seedance 2.0 / Dreamina, Sora 2, Higgsfield, PixVerse, Hailuo / MiniMax and TikTok, by 16:9, 9:16 and 1:1 |
| [webcodecs-aac-passthrough.md](webcodecs-aac-passthrough.md) | Copying the original AAC track into the new MP4 without re-encoding it, plus browser support |

## 1. Why do it in the browser

- **Privacy you can check.** The video never leaves the device. Anyone can confirm it in DevTools: the Network tab shows no request near the size of the file, and cleaning still finishes with the tab set to Offline (step by step: [freevideowatermarkremover.com/about#verify](https://freevideowatermarkremover.com/about#verify)).
- **Hardware codecs.** WebCodecs exposes the same decoders and encoders the browser uses for playback, usually hardware accelerated. A 1080p clip decodes and encodes far faster than a WebAssembly ffmpeg build.
- **No server cost per video.** The only server work is whatever your product needs for accounts or quotas.

## 2. The pipeline

### 2.1 Demux

MP4 and MOV are ISO-BMFF containers. [mp4box.js](https://github.com/gpac/mp4box.js) parses them in the page and hands out encoded samples per track:

- the video track's samples, plus the codec string (`avc1.64001f`, `hvc1...`, `vp09...`, `av01...`) and the `avcC`/`hvcC`/`vpcC`/`av1C` box, which becomes the decoder's `description`;
- the audio track's samples, kept as encoded AAC frames (see [webcodecs-aac-passthrough.md](webcodecs-aac-passthrough.md)).

WebM cannot be read by MP4Box. A practical fallback is to play the file once in a hidden `<video>`, grab each presented frame with `requestVideoFrameCallback`, and re-encode without audio.

### 2.2 Decode

```js
const decoder = new VideoDecoder({
  output: (frame) => {
    ctx.drawImage(frame, 0, 0, width, height);   // OffscreenCanvas 2D
    const image = ctx.getImageData(0, 0, width, height);
    const timestamp = frame.timestamp;
    frame.close();                                // always release GPU memory
    queue.push({ image, timestamp });
  },
  error: (e) => fail(e),
});
decoder.configure({ codec, description, codedWidth, codedHeight });
for (const s of videoSamples) {
  decoder.decode(new EncodedVideoChunk({
    type: s.is_sync ? 'key' : 'delta',
    timestamp: (s.cts * 1e6) / s.timescale,
    duration: (s.duration * 1e6) / s.timescale,
    data: s.data,
  }));
}
await decoder.flush();
```

Back-pressure matters: watch `decoder.decodeQueueSize` and the encoder's `encodeQueueSize`, and wait before feeding more samples, or long 4K clips will exhaust memory.

Rotation: phone videos often store a rotation matrix instead of rotated pixels. Boxes are drawn on the *displayed* picture, so convert them into *coded* coordinates before touching pixels, and write the same rotation into the output container.

### 2.3 Rebuild the boxed area: the boundary-ring fill

For each box (padded by 2 px so anti-aliased mark edges are included):

1. **Profiles.** For each of the four sides, average the 2-pixel ring just outside that side, column by column (top and bottom) or row by row (left and right). That gives four 1-D colour profiles `T(x)`, `B(x)`, `L(y)`, `R(y)`.
2. **Coons patch.** Fill every pixel inside with the bilinearly blended Coons surface of the four profiles, with `u = x / (w-1)`, `v = y / (h-1)`:

   ```
   P(u,v) = (1-v)·T(u) + v·B(u) + (1-u)·L(v) + u·R(v)
          − [ (1-u)(1-v)·C00 + u(1-v)·C10 + (1-u)v·C01 + uv·C11 ]
   ```

   where `C00..C11` are the four corner colours. Any linear gradient across the box (sky, a lit wall, a vignette) is reproduced exactly, and the patch is continuous with the pixels around it, so there is no visible seam.
3. **Frame edges.** A side that touches the frame border has no ring. Drop it and let the remaining sides carry the fill (for a bottom-right corner mark: top and left only). This is what makes corner marks work.
4. **Only write inside the box.** Everything else in the frame stays as decoded. For a typical corner box that is 97–99% of the picture.

The cost is tiny: under 1 ms per frame for a 1080p corner box in plain JavaScript on typed arrays. Decode and encode dominate.

### 2.4 Temporal smoothing

Compression noise makes the ring flicker from frame to frame, and a patch built from it would shimmer. Keep each box's profiles from the previous frame; per channel, if the new value differs by less than a threshold (10 on a 0–255 scale works well), average it with the old one; if it differs more, it is real motion, so take the new value. Static backgrounds go quiet; camera pans still follow.

### 2.5 Encode and mux

```js
const encoder = new VideoEncoder({
  output: (chunk, meta) => muxer.addVideoChunk(chunk, meta),
  error: (e) => fail(e),
});
encoder.configure({ codec: 'avc1.640028', width, height, bitrate, framerate });
encoder.encode(new VideoFrame(canvas, { timestamp }), { keyFrame: i % 60 === 0 });
```

- **Codec level.** Pick the H.264 level from the frame size (e.g. `avc1.640033` for 4K) and check `VideoEncoder.isConfigSupported()` before starting; fall back to a lower profile if the browser refuses.
- **Bitrate.** Re-encode a little above the source: about `1.3 × (file bytes × 8 / duration)`, with a floor of 2 Mbps and a ceiling of 16 Mbps up to 2048×1152 (45 Mbps above). Lower and the re-encode softens the whole frame; much higher and files balloon for no visible gain.
- **Timestamps.** Reuse the decoded frame timestamps so variable-frame-rate phone clips stay in sync with the copied audio.

## 3. What it can and cannot do

- It cannot see under a solid mark. The box is filled to match its surroundings: natural on sky, water, walls, floors and soft backgrounds; smoothed where faces, fine text or detailed motion pass behind the mark.
- A mark that slides continuously across the frame needs a box over its whole path. Marks that sit still or jump between a few corners (TikTok, Sora 2) need one box per spot.
- Re-encoding means the file is not bit-identical. Pixels outside the boxes are not modified by the fill, and at the bitrate above they look the same at normal size.
- Container metadata (including C2PA content credentials) is not carried into the newly written MP4. That is a side effect of writing a new file, not something to target.

## 4. Use it responsibly

Clean only videos you made or have the rights to edit. Removing a visible mark does not change who owns a clip, what a generator's terms allow, or a platform's rules about labelling AI-generated video.

## Try it

- Any video, custom box: [freevideowatermarkremover.com](https://freevideowatermarkremover.com)
- Seedance 2.0 / Dreamina preset: [freevideowatermarkremover.com/seedance-watermark-remover](https://freevideowatermarkremover.com/seedance-watermark-remover)
- Sora 2 preset (two boxes): [freevideowatermarkremover.com/sora-2-watermark-remover](https://freevideowatermarkremover.com/sora-2-watermark-remover)
- Engine source as served to the browser: [freevideowatermarkremover.com/engine/](https://freevideowatermarkremover.com/engine/)

Free: 3 videos a day up to 1080p, no account. Also in Spanish, Portuguese and French: [/es](https://freevideowatermarkremover.com/es), [/pt](https://freevideowatermarkremover.com/pt), [/fr](https://freevideowatermarkremover.com/fr).

## License

The notes in this repository are released under CC BY 4.0. Code snippets are MIT.
