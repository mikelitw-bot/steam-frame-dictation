# steam-frame-dictation (zh-TW fork)

Fork of [khvn26/steam-frame-dictation](https://github.com/khvn26/steam-frame-dictation): an offline dictation daemon for the Steam Frame (SteamOS, aarch64), written in C on top of whisper.cpp. This fork adds non-English dictation, aimed at Traditional Chinese, and sends the changes upstream as pull requests.

回覆用繁體中文；code 註解、commit message、PR 內容用英文（要送回上游）。

## Remotes and branches

- `origin` = `mikelitw-bot/steam-frame-dictation` (this fork), `upstream` = `khvn26/steam-frame-dictation`.
- `main` mirrors `upstream/main`. Do not commit to it.
- Branches are stacked, one upstream PR each:
  1. `feat/language-option` — `--language`, `--whisper-prompt`, UTF-8-aware transcript filter
  2. `feat/paste-output` — `--paste-command`, bundled `sfd-x11-copy` clipboard helper, "Other languages" README section
  3. `docs/zh-tw` — `README.zh-TW.md`, Traditional Chinese `--help`
- `zh-tw` = `docs/zh-tw` plus this file. It is what runs on Mike's Frame and is never sent upstream.
- This file must not appear in an upstream PR.

## Rules for upstream-bound changes

- Keep features generic (a flag, not hard-coded Chinese) and leave defaults unchanged.
- Upstream is small and terse (see the `deslop` commit). Match the existing dense C style, small diffs, no new dependencies in the main binary.
- Every flag is documented in three places that must stay in sync: `usage()` in `native/main.c`, `native/help_zh_tw.c`, and the tables in `README.md` / `README.zh-TW.md`.

## Layout

- `native/` — daemon sources. `main.c` (flags, startup), `session.c` (recording, segmentation), `transcriber_whisper.c` (Whisper call, transcript filter), `output_uinput.c` (virtual keyboard, paste), `x11_copy.c` (standalone clipboard helper, links libX11 only).
- `include/` — vendored whisper.cpp / ggml headers. `lib/whisper/` and `models/` are not in git; CI builds and downloads them.
- `packaging/` — release archive builder and the installer that writes the systemd user service.

## Build and test

The code is Linux-only (`linux/input.h`, uinput) and does not compile on macOS.

- Build in a `linux/arm64` Ubuntu 24.04 container with whisper.cpp built at the commit pinned in `.github/workflows/release.yml`. Ubuntu 24.04 and the Frame both use glibc 2.39, so the binary runs on the Frame as built.
- End-to-end test without hardware: put a fake `pw-record` on `PATH` that streams a raw 16 kHz s16 mono file at real-time pace, pass `--device <fifo>`, then write one `struct input_event` (`EV_KEY`, code 353, value 1) into the FIFO. Add `--dry-run` to skip uinput.
- The Frame has gcc, make and libX11 headers, so `make -C native sfd-x11-copy` works there.

## The Frame

- Reach it with `ssh frame` (SteamOS host, user `steamos`). adb is the wrong channel: it connects to the Lepton Android instance, not SteamOS.
- Session is gamescope with two Xwayland displays: `:0` is the Steam UI, `:1` is the running game. The user systemd environment has `DISPLAY=:0`.
- No `wl-copy`, `xclip` or `xdotool` on the device; hence `sfd-x11-copy`.
- The trigger is the headset's own aux button (`gpio-keys`, evdev code 353), which the daemon grabs exclusively along with Volume Up.
- Install lives in `~/.local/share/steam-frame-dictation`; the Chinese settings are a systemd drop-in in `~/.config/systemd/user/frame-dictation.service.d/`.

## Measurements on the Frame (Chinese, audio ctx 768)

| Model | Size | 2 threads | 4 threads | Notes |
| --- | --- | --- | --- | --- |
| `ggml-base.bin` | 148 MB | ~3.2 s | ~1.6 s | in use |
| `ggml-base-q5_1.bin` | 60 MB | ~1.9 s | ~0.9 s | accurate on synthetic speech, poor on Mike's live speech |
| `ggml-tiny.bin` | 78 MB | ~1.3 s | — | mixes Simplified characters, wrong words |

Timings came from synthetic (macOS `say`) speech. Live speech with the default 0.35 s actionable pause produced 1.4–2.5 s segments that were recognized badly, so the Frame runs `--actionable-pause-seconds 0.6`.

## Not done

- `install.sh` and the release workflow hard-code `ggml-tiny.en.bin` and the upstream repo, so there is no one-line Chinese install.
- Pasting into in-game text fields (display `:1`) is unverified.
- Whisper sometimes emits bracketed non-speech such as `(招呼)`; nothing filters it.
