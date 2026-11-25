# LiteTube

A lightweight Android app for downloading YouTube videos and audio (MP3 / M4A),
with a built-in audio trimmer.

> **Status:** design phase — no code yet.
> See [the design document](docs/superpowers/specs/2026-11-25-litetube-design.md) (Vietnamese).

## Goals

- Small APK (target < 10 MB) — no bundled FFmpeg.
- Download audio as **MP3** (128 / 192 / 320 kbps) or **M4A** (no re-encode).
- Download video as MP4, up to 1080p.
- Trim audio with sample-accurate cut points, optional fade in/out.

## How it works

- Stream extraction: [NewPipeExtractor](https://github.com/TeamNewPipe/NewPipeExtractor)
- MP3 encoding: [LAME](https://lame.sourceforge.io/) via JNI
- Decoding, muxing and trimming: Android's built-in `MediaExtractor` / `MediaCodec` / `MediaMuxer`

Inspired by [Tyrrrz/YoutubeDownloader](https://github.com/Tyrrrz/YoutubeDownloader).

## Disclaimer

LiteTube is not affiliated with YouTube or Google. You are responsible for
complying with YouTube's Terms of Service and the copyright of the content you
download.

## License

[GPL-3.0](LICENSE)
