# Keeping the original AAC audio: MP4Box sample copy

When you only change pixels, there is no reason to touch the sound. Decoding and re-encoding AAC costs time, adds a generation of lossy compression, and is the most common cause of audio drift in browser video tools. Copy the encoded audio samples into the new MP4 instead.

This is how [Free Video Watermark Remover](https://freevideowatermarkremover.com) keeps "original audio, untouched" (engine: [freevideowatermarkremover.com/engine/](https://freevideowatermarkremover.com/engine/)).

## 1. Read the audio track with MP4Box

```js
import { createFile } from 'mp4box';

const mp4 = createFile();
let audio = null;

mp4.onReady = (info) => {
  const track = info.audioTracks[0];
  if (!track || !track.codec.startsWith('mp4a')) return; // not AAC: skip audio
  audio = {
    id: track.id,
    codec: 'aac',
    sampleRate: track.audio.sample_rate,
    numberOfChannels: track.audio.channel_count,
    timescale: track.timescale,
    samples: [],
  };
  mp4.setExtractionOptions(track.id, null, { nbSamples: 1000 });
  // ...the video track is set up the same way, see README.md
  mp4.start();
};

mp4.onSamples = (trackId, _user, samples) => {
  if (!audio || trackId !== audio.id) return;
  for (const s of samples) {
    audio.samples.push({
      data: s.data,                                 // raw AAC frame
      type: s.is_sync ? 'key' : 'delta',            // AAC frames are all sync
      timestamp: (s.cts * 1e6) / s.timescale,       // microseconds
      duration: (s.duration * 1e6) / s.timescale,
    });
  }
};

// Feed the file in chunks; MP4Box needs fileStart on every buffer.
let offset = 0;
for await (const chunk of file.stream()) {
  const buf = chunk.buffer.slice(chunk.byteOffset, chunk.byteOffset + chunk.byteLength);
  buf.fileStart = offset;
  offset += buf.byteLength;
  mp4.appendBuffer(buf);
}
mp4.flush();
```

Notes:

- **Check the codec.** `mp4a.40.2` (AAC-LC) and `mp4a.40.5` (HE-AAC) copy cleanly. If the track is anything else (AC-3, Opus in MP4, ALAC), either re-encode it with `AudioEncoder` or drop it and say so in the UI. Never silently ship a broken track.
- **Timestamps are microseconds.** WebCodecs and most muxers use µs; MP4Box gives track timescale units. Convert once, right here.
- **Memory.** Audio is small (about 1 MB a minute at 128 kbps), so holding all samples is fine. Video is not: stream video samples to the decoder instead of buffering them.

## 2. Write it into the new MP4

With [mp4-muxer](https://github.com/Vanilagy/mp4-muxer), declare the audio track up front and add the raw samples after the video is encoded:

```js
import { Muxer, ArrayBufferTarget } from 'mp4-muxer';

const muxer = new Muxer({
  target: new ArrayBufferTarget(),
  video: { codec: 'avc', width, height, rotation },
  audio: audio && {
    codec: 'aac',
    sampleRate: audio.sampleRate,
    numberOfChannels: audio.numberOfChannels,
  },
  fastStart: 'in-memory',
  firstTimestampBehavior: 'offset',
});

// VideoEncoder output → muxer.addVideoChunk(chunk, meta) for every frame…
await videoEncoder.flush();

for (const s of audio?.samples ?? []) {
  muxer.addAudioChunkRaw(s.data, s.type, s.timestamp, s.duration);
}
muxer.finalize();
const blob = new Blob([muxer.target.buffer], { type: 'video/mp4' });
```

- **The decoder config travels too.** `addAudioChunkRaw` without metadata works for plain AAC-LC because the muxer writes a standard `esds` from the declared sample rate and channel count. For unusual streams (HE-AAC v2, odd channel layouts), pass the original `AudioSpecificConfig` from the source track's `esds` box as the decoder config so players read the stream exactly as before.
- **Keep video timestamps from the decoder.** If you rebuild video timestamps from a constant frame rate, variable-frame-rate phone clips will slowly drift away from the copied audio. Pass each decoded `VideoFrame.timestamp` through to the encoder.
- **`firstTimestampBehavior: 'offset'`** handles sources whose first sample does not start at zero (edit lists, trimmed exports).

## 3. Containers that MP4Box cannot read

WebM (VP8/VP9 + Opus/Vorbis) is not ISO-BMFF. Options, from simplest:

1. Play the file once in a hidden muted `<video>`, capture frames with `requestVideoFrameCallback`, re-encode the video, and **state that audio is not kept**.
2. Demux with a WebM parser, decode Opus with `AudioDecoder` and re-encode to AAC with `AudioEncoder` (support for AAC encoding varies by browser).
3. Copy Opus into MP4 (`Opus` in ISO-BMFF is valid, but some players still refuse it).

Free Video Watermark Remover uses option 1 for WebM and tells the user up front.

## 4. Browser support (WebCodecs + OffscreenCanvas)

| Browser | VideoDecoder / VideoEncoder | Notes |
|---|---|---|
| Chrome / Edge 94+ | Yes | Hardware H.264 encode on most machines; HEVC decode where the OS has it |
| Opera, Brave, Vivaldi (Chromium 94+) | Yes | As Chrome |
| Safari 17+ (macOS, iOS, iPadOS) | Yes | H.264 and HEVC decode; slower than Chrome on long 4K clips |
| Firefox 130+ desktop | Yes | H.264 encode and decode; HEVC input may fail to decode |
| Chrome for Android | Yes | Works, slower; mind memory on 4K |
| Older browsers | No | Feature-detect and explain instead of falling back to an upload |

Feature detection:

```js
const canProcessLocally =
  typeof window !== 'undefined' &&
  'VideoEncoder' in window &&
  'VideoDecoder' in window &&
  typeof OffscreenCanvas !== 'undefined';

const support = await VideoEncoder.isConfigSupported({
  codec: 'avc1.640028', width, height, bitrate, framerate: 30,
});
if (!support.supported) { /* try a lower profile/level, e.g. avc1.4d0028 */ }
```

## 5. Proving the audio was copied

- Compare `ffprobe -show_streams` on input and output: same codec, sample rate, channels and bitrate; `nb_frames` equal for the audio stream.
- Hash the audio packets: `ffmpeg -i in.mp4 -map 0:a -c copy -f data - | sha256sum` gives the same digest for input and output when the samples were copied byte for byte.
- In the product, say it plainly in the result line ("original audio kept") so users don't have to wonder.

## Related

- Pipeline overview and the boundary-ring fill: [README.md](README.md)
- Starting boxes per app and aspect ratio: [presets-aspect-ratios.md](presets-aspect-ratios.md)
- Live tool: [freevideowatermarkremover.com](https://freevideowatermarkremover.com) · Sora 2 preset: [freevideowatermarkremover.com/sora-2-watermark-remover](https://freevideowatermarkremover.com/sora-2-watermark-remover)
