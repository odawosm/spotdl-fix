# spotdl setup

Working [spotdl](https://github.com/spotDL/spotify-downloader) configuration on Fedora,
plus the failures hit getting there and what actually fixed each one.

Recorded 2026-07-25 · spotdl 4.5.2 · Python 3.14.6 · yt-dlp 2026.7.4

## Install

spotdl is a Python CLI, so it goes in its own venv rather than the system Python.
No `sudo`, no new system packages — `pipx` and `uv` do the same thing if you'd rather.

```sh
python3 -m venv ~/.local/share/spotdl-venv
~/.local/share/spotdl-venv/bin/pip install --upgrade pip spotdl
ln -sf ~/.local/share/spotdl-venv/bin/spotdl ~/.local/bin/spotdl
```

`ffmpeg` must be present (`sudo dnf install ffmpeg`). Upgrade later with:

```sh
~/.local/share/spotdl-venv/bin/pip install -U spotdl
```

The symlink follows automatically.

## Config

Copy `config.example.json` to `~/.config/spotdl/config.json` and fill in your own
Spotify credentials (see below).

| Key | Value | Why |
|---|---|---|
| `format` | `m4a` | YouTube Music serves AAC natively; keeping it skips a re-encode |
| `bitrate` | `disable` | Source is AAC-LC 128 kbps — 128k MP3 was generation loss at equal bitrate |
| `threads` | `8` | Downloads are network-bound; parallelism is the only real speedup for playlists |
| `yt_dlp_args` | `--cookies-from-browser firefox` | Clears YouTube's "confirm you're not a bot" gate |
| `use_official_api` | `true` | Default path scrapes Spotify's internal API and trips its fraud check |
| `print_errors` | `true` | Default hides yt-dlp's real error behind a one-line summary |
| `save_errors` | `~/.config/spotdl/errors.log` | spotdl keeps **no logs at all** by default |

### Spotify credentials

spotdl ships public credentials shared by every user of the tool, so their rate limit
is a shared resource that runs dry. Register your own free app:

1. <https://developer.spotify.com/dashboard> → **Create app**
2. Redirect URI `http://127.0.0.1:8080`, tick **Web API**
3. Copy the Client ID and secret into `config.json`

Verify them independently of spotdl:

```sh
curl -s -X POST https://accounts.spotify.com/api/token \
  -d grant_type=client_credentials \
  -d client_id=YOUR_ID -d client_secret=YOUR_SECRET \
  -w '\nHTTP %{http_code}\n'
```

`HTTP 200` with an `access_token` means the credentials are fine.

## Shell aliases

Append `zsh_aliases.snippet` to `~/.zsh_aliases`. `--format` works directly too —
the aliases only save pairing it with the right bitrate.

| Command | Output |
|---|---|
| `spotdl "<url>"` | m4a (config default) |
| `spotm4a "<url>"` | m4a, no re-encode |
| `spotmp3 "<url>"` | mp3 @ 192k, for older car stereos |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `zsh: no matches found` | zsh expands `?si=` as a glob | Quote the URL, or drop the `?si=…` parameter |
| `Some YouTube downloads require Deno` | yt-dlp needs a JS runtime for YouTube's signature challenge | `spotdl --download-deno` |
| `Sign in to confirm you're not a bot` | Anonymous requests are gated | `--cookies-from-browser firefox` (in config) |
| `HTTP Error 429` (YouTube) | IP rate-limited | Cookies usually clear it; else wait and use `--threads 1` |
| `Could not get client token` | Spotify's internal API fraud check, via `spotapi` | `use_official_api: true` |
| `reached a rate/request limit … 86400 s` | Spotify Web API 429, 24h server-side cooldown | Own credentials — see above |
| `AudioProviderError`, no detail | spotdl truncates yt-dlp's message | Read `errors.log`, or run `yt-dlp -F <youtube-url>` directly |
| Hangs with no output | — | Don't pipe to `tail`; pipe buffering hides output until exit. Redirect to a file |
| Empty `~/.config/spotdl/temp/` | Failure happens before any bytes transfer | Auth or rate limit, not a stalled connection |

### Playback

Files are AAC-LC in `.m4a`. Fine on phones, desktops, and car head units from roughly
2010 on via USB. Older decks and burned data CDs are often MP3-only — convert a copy
rather than downgrading the library:

```sh
ffmpeg -i in.m4a -b:a 192k out.mp3
```

192k rather than 128k because lossy→lossy transcoding always loses; the higher target
absorbs it.

## Measured

- Re-encoding to MP3 costs **~8.5s CPU per track** (6:34 track, measured with ffmpeg, no network)
- Typical download is **~70s wall clock**, dominated by YouTube throttling
- Wall-clock timings are network-bound and noisy — single samples are meaningless.
  An early 80s-vs-61s comparison looked like a 24% win but did not reproduce.

## Open issue

As of 2026-07-25 the Spotify Web API returns 429 to spotdl even with fresh personal
credentials, while the identical request via `curl` returns 200:

```sh
# works — HTTP 200, correct track data
curl -H "Authorization: Bearer $TOKEN" https://api.spotify.com/v1/tracks/<id>

# fails — spotipy logs "rate/request limit … 86400 s"
spotdl save "https://open.spotify.com/track/<id>" --save-file t.spotdl
```

Credentials load correctly (verified via `spotdl.utils.config.get_config`), so the
difference is in how spotipy makes the request rather than the credentials themselves.
Unresolved.
