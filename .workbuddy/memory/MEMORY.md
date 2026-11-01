# Project notes — Lemo-Opuscar (local copy)

This workspace is a full clone of the Lemo-Opuscar film-style library
(github.com/lemomo-ai/lemo-opuscar): 66 styles, each `styles/<slug>/STYLE.md` +
`DEMO.md` + `demo/`. It is **the library itself**, not a film project. Films may live in `films/<name>/` (git-ignored) **or anywhere outside the library** —
`core/render/serve.mjs` mounts an out-of-repo film folder at `/@film/` while `/core/…`
still comes from the repo root. Never build inside `styles/`.

## Windows setup on this machine (done 2026-10-30)

| Piece | Where / what |
|---|---|
| ffmpeg 7.1 + ffprobe 6.1.1 | `C:\Users\13666\bin` (on PATH). ffmpeg from PyPI `imageio-ffmpeg` wheel, ffprobe from npmmirror `ffmpeg-static/b6.1.1/ffprobe-win32-x64`. gyan.dev direct download is ~6 KB/s — unusable. |
| npm packages | installed; registry switched to `https://registry.npmmirror.com` (official npm was 14 min and unfinished) |
| Playwright chromium-headless-shell | `%LOCALAPPDATA%\ms-playwright`, via `PLAYWRIGHT_DOWNLOAD_HOST=https://cdn.npmmirror.com/binaries/playwright` |
| `.venv` | created with system Python 3.13; **`.venv/bin/python` is a hand-written bash shim** to `.venv/Scripts/python.exe` — the library's scripts assume Unix `bin/python`. Core tier (numpy/scipy/soundfile/soxr/pillow/fonttools) installed from `mirrors.aliyun.com/pypi`. Voice and music tiers NOT installed. |
| Skill | copied to `~/.workbuddy/skills/lemo-opuscar/`, SKILL.md carries a "This machine (Windows)" section with all of the above. |

## Gotchas

- Run `setup.sh` with `LEMO_OPUSCAR_HOME=/z/DevCodeBase/AiSkillLibrary/Video/lemo-opuscar`. Git for Windows returns `Z:/…` from `git rev-parse --show-toplevel`, which `setup.sh` treats as relative and then tries to clone the library onto itself.
- Anything handing a path to a Windows binary (ffmpeg, Python) needs `C:/…`. Git-bash `/z/…` and `/tmp/…` fail (e.g. `soundfile.write('/tmp/x.wav')` → LibsndfileError).
- Set `LEMO_ANGLE=default` when rendering.
- Network: GitHub and jsdelivr unreachable. **fonts.googleapis.com is reachable** — the normal font workflow works.
- WSL is blocked by the sandbox program blacklist; the pipeline was verified natively instead.
- Verified: `still.mjs`, `video.mjs` (72 frames / 7 s), `mux.sh` (measured −14.0 LUFS).

## First film made here: 钱学森弹道 (75 s, whiteboard)

Made in `films/qianxuesen-trajectory`, since **moved** to the user's own folder
`Z:\UserFilesData\Videos\抖音创作\vibe知识大赏\第1集 钱学森弹道` (episode 1 of a Douyin series).
Out-of-library projects need two edits: `build.sh` must be told the library path
(`LEMO`/`LEMO_OPUSCAR_HOME`, not `../..`), and any tool imported from `core/` must be
imported by absolute path (`await import(pathToFileURL(LEMO + '/core/render/page.mjs'))`).

- **CJK in a stroke-order engine**: single-line fonts can't write Chinese, so the film adds a
  per-character CJK layer in `film.js` (reveal each char at `t0 + (t1-t0)*(i+1)/n`, jitter + 0.09 s pop).
  Font: Ma Shan Zheng (OFL), subset via Google Fonts `&text=`, self-hosted `fonts/MaShanZheng.woff2`.
- **readcheck contract**: `window.TEXTS(t)` should report *every* on-screen text, board handwriting
  included (see `caption-news` demo for the house style). Report a per-char-revealed label only from
  its `t1` (whole phrase on the board). Budget (chars÷4.5 + punct÷15 + 1.5) s of hold *in frame* —
  a label dies the moment the camera leaves it, so place labels where the camera is heading, or cut
  redundant ones. Debug box layout with a throwaway script that calls `window.TEXTS(t)`.
- **Loudness / true peak**: after mux's loudnorm the output true peak ≈ (crest factor − 14) LU.
  A sparse mix (quiet room tone, short VO) sits at 17–21 LU crest → the limiter engages and AAC
  overshoot lands at −0.7 dBTP (mux warns). Fix: normalise VO to −5 dBFS peak after a dB-domain
  compressor, then a stereo-linked block limiter (`mix.py: limit()`, CEIL ≈ 0.24–0.34, 2 ms attack /
  40 ms release) to bring crest near 12 → −14.0 LUFS / −1.1 dBTP. Don't squash below that.
- **Whisper-small on Chinese TTS**: reliably mishears proper names and technical terms
  (钱学森→前学生, 超声速→超生素, 机动→激动, 弹道→诞道, 滑翔→华想). Same error across two voices and
  slower rates — it's the model, not the audio. Per TECHNIQUE §4, put what it hears in `lines.json`
  `"asr"` and the check passes. `--model medium` failed to download (hf-mirror 401 on xet CAS).
- **Don't run two heavy node page jobs at once**: render, then readcheck, then stills — each opens
  its own browser; overlapping them makes every one time out (SIGTERM at 2 min default).
