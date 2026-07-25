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
| `user_auth` | `true` | Playlists require user OAuth, not just app credentials — see below |
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

## Playlists need user OAuth

Client Credentials (app-only) auth is enough for tracks and albums, but Spotify returns
`401 Valid user authentication required` on `/playlists/{id}/items` — **even for a public
playlist you own**. Playlist metadata succeeds; item listing does not.

Verify the split yourself:

```sh
# 200 - playlist metadata is fine
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.spotify.com/v1/playlists/<id>?fields=name,owner,public"

# 401 - item listing requires user auth
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.spotify.com/v1/playlists/<id>/items?limit=5"
```

Fix: set `user_auth: true`. spotdl then swaps `SpotifyClientCredentials` for
`SpotifyOAuth`, requesting scope `playlist-read-private` and opening a browser once.

**The redirect URI is hardcoded** in `spotdl/utils/spotify.py` to
`http://127.0.0.1:9900/`. Register it in the dashboard under Settings → Redirect URIs —
add **both** forms, since the dashboard may store it without the trailing slash:

```
http://127.0.0.1:9900/
http://127.0.0.1:9900
```

Then click **Save**. Added URIs appear in the field before they are persisted, which is
easy to miss.

Note that hitting `accounts.spotify.com/authorize` with `curl` is **not** a valid way to
test whether a redirect URI is registered — Spotify validates it only after login, so a
bogus URI returns the same `303` to the login page as a correct one.

## Gotchas

- **Stale token cache.** spotipy caches the access token at `~/.config/spotdl/.spotipy`
  and reuses it while `expires_at` is in the future, *without* checking whether
  `client_id` changed. After editing credentials, delete it or you will keep
  authenticating as the old app for up to an hour:

  ```sh
  rm ~/.config/spotdl/.spotipy
  ```

- **`save_errors` can crash.** Writing a `SpotifyException` to the error log raises
  `TypeError: unsupported operand type(s) for +: 'int' and 'str'` — spotdl's
  `entry_point.py:169` assumes every item in `exc.args` is a string, but
  `SpotifyException` puts an int status code first. Upstream bug; rely on
  `print_errors` (console) as the dependable channel.

- **`--use-official-api` takes no value.** It is a boolean flag, so
  `--use-official-api false` fails with "invalid choice: 'false'" because `false` is
  parsed as the operation. Set it in `config.json` instead.

- **Don't pipe to `tail` while debugging.** Pipe buffering withholds output until exit,
  so a hanging command looks like it produced nothing. Redirect to a file instead.
