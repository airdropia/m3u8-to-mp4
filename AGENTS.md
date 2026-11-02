# AGENTS.md — m3u8-to-mp4 repo (LOCKED CONFIGURATION)

Repo me sirf **2 workflows** hain. Yeh AGENTS.md **locked configuration** hai —
koi bhi agent future me isi file ka hawala de, aur m3u8 file ke liye **sabse neeche**
likha tareeka hi use kare. Bina sochne ke badlaav na kare.

---

## Wohi cheez pehle thi, jo ab fix ho gayi

| Pehle (galat) | Ab (sahi) |
|---|---|
| Pehle convert-fast.yml tha jisme probe fail hota tha (ffmpeg install nahi tha) | Ab `m3u8-to-mp4.yml` hai jisme ffmpeg + yt-dlp dono install hote hain |
| chunks default 8 tha, par default crf 28 thi (quality low thi) | Ab **crf default 24** hai (behtar quality) |
| Har baar naya experiment hota tha | Ab **ek hi locked workflow** har m3u8 file ke liye |

---

## m3u8 file ka tareeka (sirf yehi chahiye)

### Step 1: Master playlist URL nikalo

Agar user ne page link diya hai (e.g. `https://play.niazitv.pk/drama/83/single-serie?watch=1&episode=XXXX`), toh seedha mat bhejo. Pehle raw page se master URL nikalo:

```bash
curl -sL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64)" "<PAGE_URL>" | grep -oE 'https://[^"'"'"']+\.m3u8[^"'"'"']*' | sort -u | head
```

- **tinyfish_fetch / koi bhi HTML-stripping fetch mat karo** — wo `<script>` hata deta hai, player config miss ho jata hai. Raw `curl` chahiye.
- Master URL pe `/pl/<hash>.m3u8?tag=29` hota hai — **video-only variant** (`/pl/avc1/1920x1080/...`) **mat bhejo**, usme audio track nahi hota.

### Step 2: Master verify karo

```bash
curl -sL -A "Mozilla/5.0" "<MASTER_URL>" | grep -cE 'EXT-X-MEDIA.*TYPE=AUDIO'   # >= 1 hona chahiye
curl -sL -A "Mozilla/5.0" "<MASTER_URL>" | grep -c 'EXT-X-STREAM-INF'         # video variants
grep '1920x1080' <MASTER_URL>                                                   # 1080p present?
```

### Step 3: Workflow chalao

```bash
cd "C:/Users/Admin/Desktop/m3u8-Downloader/m3u8-to-mp4"
"/c/Program Files/GitHub CLI/gh" workflow run m3u8-to-mp4.yml \
  -f url="<MASTER m3u8 URL>" \
  -f chunks=8 -f crf=23 -f preset=ultrafast
```

- **chunks=8** (default: 8, parallel jobs)
- **crf=23** (quality focus — lower = better; 23 = good quality, 28 = smaller but lower)
- **preset=ultrafast** (speed)

### Step 4: Download link

Run complete hone ke baad release page se:

```
https://github.com/airdropia/m3u8-to-mp4/releases/download/<TAG>/video.zip
```

- `video.zip` → unzip karo = `output.mp4` (video+audio **merged**, h265, 1080p)
- `audio.m4a` → bonus, sirf agar chahiye

---

## Telegram post ka tareeka (alag workflow)

Telegram video ke liye **alag** workflow hai:

```bash
"/c/Program Files/GitHub CLI/gh" workflow run telegram-to-mp4.yml \
  -f link="https://t.me/<channel>/<post>" \
  -f transcode=h265
```

- Telegram file pehle se merged/compressed hoti hai → `transcode=no` bhi option hai (as-is link, fast)
- Login session `tdl-session-v1` cache me hai — QR scan **sirf pehli baar**

---

## Workflow files (sirf ye do rahengi)

| File | Kaam |
|---|---|
| `.github/workflows/m3u8-to-mp4.yml` | m3u8/HLS/tweet → h265 → merge → zip → release |
| `.github/workflows/telegram-to-mp4.yml` | Telegram post → tdl → merge → zip → release |

Koi aur workflow file mat banana. `convert.yml`, `download.yml`, `playlist.yml`, `debug-tg.yml` pehle hi delete ho chuki hain.

---

## Gotchas (again mat gharna)

1. `yt-dlp --download-sections` HLS pe kaam nahi karta (262B empty file). Hamesha poora source download karke ffmpeg se slice karo.
2. fMP4 HLS pe `-c copy` se slice corrupt file deta hai. Input-seek + re-encode safe hai.
3. `pip install X` Ubuntu 24.04 pe fail hota hai (PEP 668). `apt-get install python3-pil` use karo.
4. `gh release create` ko git repo chahiye → `actions/checkout@v4` zaroori hai.
5. `gh release create "tag" "file#label"` rename nahi karta, sirf label dikhata hai. Asli URL `gh api .../releases/tags/<tag> -q '.assets[0].browser_download_url'` se lo.
6. Job logs tabhi readable hote hain jab run complete ho. Fail ho to turant cancel karke root cause dekho, wait loops mat lagao.

---

## Fixed defaults (kisi bhi agent ke liye yaad rakhna)

| Input | Value |
|---|---|
| workflow | `m3u8-to-mp4.yml` |
| chunks | 8 |
| crf | 23 (quality preserved, 28 se behtar) |
| preset | ultrafast |
| timeout | 120 min |
| merge | stream copy (fast, no encode) |
| h265 | only chunk encode, crf 23 |
| output | `video.zip` (merged A+V) + `audio.m4a` |

Yahi configuration har m3u8 file ke liye use hogi. Koi change mat karna jab tak user khud na bole.