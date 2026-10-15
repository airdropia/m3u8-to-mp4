# AGENTS.md — m3u8-to-mp4 repo

Repo me sirf **2 workflows** hain. Kaam ke hisaab se sahi wala chuno. Dono me
**h265 encoding default** hai aur dono GitHub release me direct download link dete hain.

| Source | Workflow | Kaise pehchano |
|---|---|---|
| m3u8 / HLS / tweet | `m3u8-to-mp4.yml` | URL `.m3u8` ya `.m3u8?...` se khatam hoti hai, ya `EXT-X-STREAM-INF` wali master playlist |
| Telegram post (mp4) | `telegram-to-mp4.yml` | URL `t.me/<channel>/<post>` form me hai |

Koi bhi local build/test/install nahi karna. Verification sirf GitHub Actions se.

---

## Flow A — m3u8 / HLS / tweet video

```bash
gh workflow run m3u8-to-mp4.yml \
  -f url="<MASTER m3u8 url>" \
  -f chunks=8 -f crf=28 -f preset=ultrafast
```

### URL kaise nikalo (ye zaroori hai)

Agar user ne **stream page** diya hai (e.g. `https://play.niazitv.pk/drama/83/single-serie?watch=1&episode=40940`),
toh use seedha mat bhejo. Pehle **master playlist URL** nikalo:

```bash
curl -sL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" "<PAGE_URL>" | grep -oE 'https://[^"'\'']+\.m3u8[^"'\'']*' | head
```

- `tinyfish_fetch` / koi bhi HTML-stripping fetch use **mat** karo — wo `<script>` hata deta hai
  aur player config miss ho jata hai. Raw `curl` chahiye.
- Jo URL mile usme `/pl/<hash>.m3u8` ho = **master** hai (video+audio dono). Yehi chahiye.
- `/pl/avc1/<res>/<hash>.m3u8` = video-only variant. **Kabhi mat bhejo** — isme audio track nahi hota,
  output silent aayega.

Verify ki playlist me dono hain:
```bash
curl -sL -A "Mozilla/5.0" "<MASTER_URL>" | grep -cE 'EXT-X-MEDIA.*TYPE=AUDIO'   # >=1 hone chahiye
curl -sL -A "Mozilla/5.0" "<MASTER_URL>" | grep -c 'EXT-X-STREAM-INF'             # resolution variants
```

Agar source **video-only** hai aur audio alag hai, to `audio_url` input do. Merge job use
audio encode karke attach kar deta hai.

### Pipeline (kya hoga internally)

1. **probe** — yt-dlp se duration. Direct mp4 URL pe yt-dlp duration nahi deta, to fallback
   me poora source download karke `ffprobe` se duration nikaalta hai.
2. **encode** (matrix, `chunks` jobs) — har job poora source download karta hai, phir apna
   time range `-ss/-t` se slice karke h265 encode karti hai. `libx265`, `-tag:v hvc1` (Apple
   compatibility), audio `aac 128k`.
3. **merge** — chunks concat (`-c copy`), optional audio attach, verify gate (dono tracks +
   duration drift <5s), phir `video.zip` + `audio.m4a` release.

Note: chunks parallel hone se speed 4x+ aati hai (2h23m video: ~110 min → ~28 min).

---

## Flow B — Telegram post (mp4)

```bash
gh workflow run telegram-to-mp4.yml \
  -f link="https://t.me/<channel>/<post>" \
  -f transcode=h265 -f chunks=8 -f crf=28 -f preset=ultrafast
```

- `transcode=h265` (default) → `video.zip` + `audio.m4a`
- `transcode=no` → as-is raw mp4 ka direct link (Telegram ka file already compressed hota hai,
  dobara encode karne ka koi fayda nahi)

### Login

tdl ka session `tdl-session-v1` cache me hota hai — **har run me login nahi lagta**. Cache
miss hone par step automatically QR mode me chali jati hai:

- ASCII QR Actions log me render nahi hota (phone scanner fail karta hai), isliye workflow
  QR ko PNG me convert karke release asset ke roop me publish karta hai.
- Live log me QR URL dikhta hai → phone me browser me kholo → Telegram
  **Settings > Devices > Link Desktop Device** se scan karo.
- Runner `script -qec` (pseudo-TTY) use karta hai kyunki tdl ko TTY chahiye.

Repo secrets me sirf `TG_PHONE` set hai (`+923015503764`). Use tab chahiye jab QR kaam na kare.

### Important

- Telegram media anonymous web pe expose nahi hota (embed page me zero video tags). Isliye
  ye workflow authenticated session use karta hai — `yt-dlp` se Telegram nahi chalega.
- MTProto proxy (`t.me/proxy?...`) tdl support **nahi** karta (source-confirmed: tdl sirf
  socks5/http CONNECT). GitHub runners Telegram seedha reach karte hain, proxy ki zarurat nahi.
- Multi-file post me `find | head -1` sirf pehla file leta hai.

---

## Release tags

| Tag | Meaning |
|---|---|
| `tg-src-<epoch>` | raw Telegram download (as-is, no encode) |
| `converted-fast-<epoch>` | h265 encode result: `video.zip` + `audio.m4a` |
| `tg-qr-<epoch>` | temporary QR PNG (login ke time banta hai, login ke baad delete ho jata hai) |

`video.zip` 2GiB se bada hua to `video.zip.partaa/partab` me split hota hai; release notes
me rebuild command diya hota hai.

Zip >2GiB hota hai tab bhi `gh release upload` allowed hai — split sirf convenience ke liye
 hai, GitHub asset limit nahi hai.

---

## Gotchas jo dobara mat ghana

1. `yt-dlp --download-sections` HLS + separate audio pe **kaam nahi karta** (262-byte empty
   file). Hamesha poora source download karke ffmpeg se slice karo.
2. fMP4 HLS pe `-c copy` se slice karne se corrupt file banti hai (`Invalid NAL unit size`).
   Input-seek + re-encode safe hai.
3. `pip install X` Ubuntu 24.04 runners pe fail hota hai (PEP 668). `apt-get install python3-pil`
   use karo.
4. `gh release create` ko git repo chahiye → `actions/checkout@v4` zaroori hai.
5. `gh release create "tag" "file#label"` **rename nahi karta**, sirf display label set karta.
   Asset ka asli URL `gh api .../releases/tags/<tag> -q '.assets[0].browser_download_url'` se lo.
6. Job logs tabhi readable hote hain jab run complete ho. Fail jaldi dekhna ho to
   `gh run cancel <id>` karke phir log padho.
7. Long/repetitive command output summarize ho sakta hai — output ko file me redirect karke
   phir padho.

## Repo me kuch aur nahi hai

Sirf `.github/workflows/m3u8-to-mp4.yml` aur `.github/workflows/telegram-to-mp4.yml`.
Local code, tests, ya build config nahi hai — sab processing CI pe hoti hai.