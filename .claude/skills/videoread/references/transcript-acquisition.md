# Getting a transcript

Ordered by what actually works from a cloud/remote session. Each entry lists the
failure you'll see, so you can recognise it and move on rather than debugging it.

## 1. Metadata first — oEmbed

Always works, no auth, no rate limit. Do this before anything else: the title
and channel are what you'll search with later, and they confirm you have the
right video.

```bash
curl -sS "https://www.youtube.com/oembed?url=https://youtu.be/<ID>&format=json"
```

Returns `title`, `author_name`, `author_url`, `thumbnail_url`.

## 2. Indexed crawl — normally the winner

Exa's `crawling_exa` against the **full watch URL** (not the `youtu.be` short
form) returns the transcript as clean prose:

```
mcp__EXA2__crawling_exa(
  urls: ["https://www.youtube.com/watch?v=<ID>"],
  maxCharacters: 200000
)
```

Set `maxCharacters` high. Rough sizes:

| Runtime | Characters |
|---|---|
| 20 min | ~20k |
| 1 hour | ~55k |
| 2 hours | ~110k |
| 3 hours | ~170k |

Anything over roughly 30k exceeds the tool output limit and gets written to a
file instead — that is normal and fine, not an error. The message tells you the
path. See "Reading a large transcript file" below.

Caveat: this returns the auto-generated transcript, so speaker turns are not
marked. In multi-speaker content you infer turns from context. Where that's
genuinely ambiguous, a podcast transcript aggregator (search the episode title)
often has a speaker-labelled version worth cross-checking.

## 3. yt-dlp — usually fails from cloud IPs, try if the above misses

```bash
pip install -q yt-dlp
python3 -m yt_dlp --skip-download --write-auto-subs --write-subs \
  --sub-langs "en.*" --sub-format vtt -o "vid.%(ext)s" "<URL>"
```

Expected failure from a datacenter IP:

```
ERROR: Sign in to confirm you're not a bot. Use --cookies-from-browser or --cookies
```

Often preceded by `HTTP Error 429: Too Many Requests`. This is IP reputation,
not a fixable configuration problem — don't burn turns on it. It works fine from
a residential connection, so it's the right first choice locally.

## 4. Aggregator sites — the fallback, and the chapter-list source

Searching the episode title surfaces sites that have already transcribed and
chaptered it. These are worth hitting **even when you already have the
transcript**, because they carry two things the raw transcript doesn't:

- **Chapter lists with timestamps** — your coverage checklist. This is the single
  highest-value artifact for avoiding dropped sections.
- **Speaker names and credentials** — for correcting mangled ASR names.

Search shape that works: `<episode title> <channel> transcript` or
`<episode title> podcast notes`.

## Reading a large transcript file

The file has very few line breaks, so the Read tool's `offset`/`limit` can't
chunk it — it will tell you the lines are too long. Slice by character range:

```bash
F=<path-from-the-tool-result>
python3 -c "
d=open('$F').read()
print('TOTAL:', len(d))
print(d[0:16000])
"
```

Then step forward: `d[16000:32000]`, `d[32000:48000]`, and so on to the end.

Keep windows at or under ~16k characters. Larger ones get persisted to yet
another file rather than returned inline, costing a round trip for nothing.

To spot-check a specific figure without re-reading everything:

```bash
python3 -c "
import re
d=open('$F').read()
for kw in ['treasuries','first island chain','38% of world product']:
    m = re.search(re.escape(kw), d)
    if m:
        print('>>>', kw, '@', m.start())
        print(d[max(0,m.start()-400):m.start()+400].replace(chr(10),' '), '\n')
"
```

Do this before quoting any number. Auto-transcription garbles figures
routinely — `$4,200` for `$42 trillion`, `2020` for `2028` — and a wrong number
in a delivered analysis is the most damaging error available to you.

## Common ASR corruptions to watch for

Auto-transcription reliably mangles proper nouns and terms of art. Correct them
against the real names once you've identified the speakers and topic:

- Personal names — anything non-Anglophone, and surnames generally
- Institutions and acronyms
- Technical terms, treaties, and named documents
- Foreign-language terms
- Large numbers and units

If a term looks almost-but-not-quite like something real, it usually is that
thing. Search it rather than transcribing the corruption into your output.
