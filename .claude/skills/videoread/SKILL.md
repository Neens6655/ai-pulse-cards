---
name: videoread
description: Extract the full transcript of a YouTube video (or podcast episode) and turn it into a structured, labelled analysis — optionally emailed or published as a page. Use this whenever the user pastes a YouTube/youtu.be link and wants to know what's in it, invokes /videoread, or asks you to summarize, break down, analyse, "read", or pull the insights out of a video, talk, interview, podcast, or lecture — including when they just paste a link with no instructions, or ask for it to be sent to their email. Also use it when they want the key points, findings, predictions, or takeaways from long-form spoken content they can't watch themselves.
---

# videoread

Turn a video into an analysis someone can act on without watching it.

The hard part is not fetching the transcript. It's that long-form spoken content
is *dense and unstructured* — a three-hour panel contains a hundred distinct
claims, and the natural failure is to compress it into a tidy essay that reads
well and loses the specifics that made it worth watching. The specifics **are**
the value. Guard them.

## The two failures this skill exists to prevent

Both have happened. Both look like success until someone checks.

**1. Summarising from the first chunk.** Transcripts of long videos exceed the
tool output limit and get written to a file. If you read only what came back
inline, you will confidently summarise the first 20 minutes of a three-hour
episode and never notice. Read all of it, and say how much you read.

**2. Writing a phrase that sounds like content but explains nothing.** "An
economic shock propagating from the Gulf into US debt refinancing" is not a
finding — it's a gesture at one. The actual finding was a four-step mechanism:
Gulf states and Japan buy US Treasuries → the war wrecks Gulf revenue and
weakens the yen → they stop buying → the US cannot refinance ~$40tn of debt.
When you catch yourself writing an abstract noun phrase, ask whether a reader
could explain the mechanism back to you. If not, you've dropped the content and
kept the shape of it.

The test for both: could the user ask you a follow-up question about any part of
the video and get a real answer? If parts would send you back to the transcript,
you haven't finished.

## Workflow

### 1. Identify the video

`oEmbed` is reliable, cheap, and needs no auth — use it first to get the title
and channel, which you need anyway to search for context:

```bash
curl -sS "https://www.youtube.com/oembed?url=<URL>&format=json"
```

### 2. Get the transcript

See `references/transcript-acquisition.md` for the fallback chain and the
specific errors each method throws. Summary: `yt-dlp` usually fails from cloud
IPs with a bot check, so an indexed-content crawler (Exa `crawling_exa` on the
`youtube.com/watch?v=` URL) is normally the fastest path. Ask for a large
`maxCharacters` — a 3-hour episode runs 150k–200k characters.

### 3. Read the entire transcript

If the result exceeded the token limit it was saved to a file. Its lines are too
long for the Read tool's offset/limit, so slice by character range instead:

```bash
python3 -c "
d=open('<file>').read()
print('TOTAL:', len(d))
print(d[0:16000])
"
```

Then walk forward in ~16k-character windows until you reach the end. Chunks
larger than that get persisted to another file rather than returned inline,
which wastes a round trip.

State the character count you read in the final output. It's a claim the user
can hold you to, which is exactly why it's worth making.

### 4. Build a coverage map before you write

This is the step that prevents whole topics from silently vanishing, and it
costs one search.

Long videos are chaptered, and podcast aggregators publish those chapter lists
with timestamps. Search for the episode title and pull the chapter list from
whatever indexed it. Now you have an outline of everything the video covers,
written by someone else — use it as a checklist and confirm each chapter is
represented before you deliver.

Without this you will drop entire sections and not know it. In one case an
entire Taiwan and Pacific-strategy segment went missing from a summary; the
chapter list named it plainly.

Also identify the speakers properly at this stage. Automatic transcription
mangles names ("Professor Jen" for Jiang Xueqin, "Kato the Elder" for Cato,
"clean brick memo" for the Clean Break memo). Search for the episode's guest
list, get real names and credentials, and correct the mangled terms — a
misattributed claim is worse than an omitted one.

### 5. Extract by claim type, not by topic

This is the core of the skill. As you read, sort what you find into these
buckets, because a reader needs to know *what kind of thing* each statement is:

| Type | What it is |
|---|---|
| **Position** | The speaker's overall stance, in one sentence |
| **Insight** | A claim about how something works that you didn't already know |
| **Prediction** | A forecast — capture the number, the date, and the hedge |
| **Evidence** | What backs the claim: studies, data, direct experience, access |
| **Mechanism** | A causal chain. Preserve every step; these are the first thing lost to compression |
| **Weak point** | Where the speaker is unsupported, conflicted, or self-interested |
| **Contested** | A claim other speakers or the wider record dispute |

For multi-speaker content, keep these **per speaker**. Merging three people's
views into one voice destroys the most useful information in a debate: who
believes what, and on what grounds.

### 6. Structure the output

Label every section by what it contains so the reader never has to infer whether
they're reading a fact, a forecast, or your opinion. A structure that works:

```
CONTEXT          Anything needed to make the rest legible (setting, timing, stakes)
PER SPEAKER      One block each: position / insight / evidence / weak point
AGREEMENTS       What everyone accepts — often the best-supported material present
DISAGREEMENTS    Each question, each speaker's answer, and whether it resolved
PREDICTIONS      One table: who / figure / claim. Never merge these into prose
KEY FINDING      The single most important item, given room
CONTESTED        Disputed claims, attributed, with a note on their status
CONCLUSION       Explicitly labelled, so your synthesis is never mistaken for theirs
WHAT TO WATCH    Concrete indicators and dates, where the content supports them
```

Adapt it to the material — a solo lecture has no disagreements section, a
how-to video wants steps rather than positions. The rule that carries across is
that **the label tells the reader what kind of claim follows**.

Two things to hold to regardless of format:

- **Attribute everything.** "Pape says X" not "the panel suggests X." When
  speakers disagree, that attribution is the content.
- **Separate their claims from your assessment.** Put your read in its own
  labelled section at the end. Users want both; they need to know which is which.

### 7. Handle contested and sensitive claims honestly

Long-form political, health, and financial content contains claims that are
disputed, unfalsifiable, or advanced well beyond their evidence. Neither
laundering them as findings nor refusing to describe them serves the user.

Report what was claimed, attribute it precisely, note where other speakers
pushed back, and flag its status in a sentence — "these are contested claims
about motive, not established findings." The user asked what's in the video.
Tell them, and tell them how much weight it carries.

Where a speaker is selling something the content promotes — a book, a supplement
line, a fund — note it under their weak point. It's relevant to how the claims
should be weighed and it costs one clause.

### 8. Deliver

Terminal output is the default. If the user asks for email or a page, or the
analysis is long enough to be worth keeping, offer it in one line rather than
assuming.

For **email** (Gmail MCP `send_message`): send `htmlBody` *and* `body`. Use real
structure — section headers, tables for predictions, blockquote-style callouts
for the key finding. Plain-text alternative should mirror it with capitalised
headers and `---` rules.

For a **page** (Artifact): load `artifact-design` first, and prefer it when the
analysis has comparison structure that benefits from side-by-side layout.

If both, put the page link at the top of the email so nothing is orphaned.

## Calibration

Match depth to source. A 10-minute explainer needs a tight structured summary; a
3-hour debate needs the full treatment and will run long. Long is correct when
the source is dense — a user asking about a three-hour episode has already
signalled they want substance. Don't trim to look elegant.

If you find yourself cutting to keep it readable, split it — deliver the core
analysis, then offer the remainder — rather than dropping material silently. The
user cannot ask about what they don't know you removed.
