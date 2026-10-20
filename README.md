# Text to Good Speech Guidelines

This guide shows how to turn text into natural-sounding MP3 audio, how to find the available voices, and which voices and settings sound best. All the code needed is included below.

## Overview

All audio is generated with **[edge-tts](https://github.com/rany2/edge-tts)** (tested with v7.2.8). It's a Python wrapper around the Microsoft Edge "Read Aloud" text-to-speech service. It gives free access to Microsoft's Azure **Neural** voices, with no API key or Azure account needed. It does need an internet connection.

```bash
pip install edge-tts
```

[ffmpeg](https://ffmpeg.org/) is used only to compress the final MP3s.

## The Pipeline

```
input.md  ──(optional) repeat blocks──▶  ──alternating voices──▶  output.mp3  ──ffmpeg──▶  output-compressed.mp3
```

### 1. Prepare the text

Split the text into **blocks** separated by **one blank line**. The code below splits on `\n\n`, so each blank line marks where one block ends and the next begins. A block can be a heading with its paragraph, a single paragraph, or a line of dialogue.

```markdown
First Topic
A sentence or paragraph explaining the first topic.

Second Topic
A sentence or paragraph explaining the second topic.
```

Tips for text that sounds good when spoken:
- Put headings or titles on their own line. The line break gives a natural pause before the text that follows.
- Avoid Markdown formatting (`**bold**`, bullets, tables). The text is sent as-is, so symbols may be read aloud or cause odd pauses.
- Spell out abbreviations if they should be read as words (e.g. "versus" instead of "vs.").
- Don't leave stray blank lines inside a block, or it will be split into two.

### 2. (Optional) Repeat each block

If you want each block read more than once, this writes every block `times` times into a new file:

```python
def repeat_blocks(input_file, output_file, times=3):
    with open(input_file, "r", encoding="utf-8") as f:
        blocks = [b.strip() for b in f.read().split("\n\n") if b.strip()]

    with open(output_file, "w", encoding="utf-8") as f:
        for block in blocks:
            for _ in range(times):
                f.write(block + "\n\n")

repeat_blocks("input.md", "input_repeated.md")
```

### 3. Synthesize with alternating voices

Switching between two voices on each block makes long audio much easier to listen to. Each change of speaker clearly marks a new section, so it sounds like a conversation rather than one long monologue.

```python
import asyncio
import edge_tts

VOICE_A = "en-US-ChristopherNeural"
VOICE_B = "en-US-EricNeural"

async def generate_alternating_audio(input_file, output_file):
    with open(input_file, "r", encoding="utf-8") as f:
        blocks = [b.strip() for b in f.read().split("\n\n") if b.strip()]

    with open(output_file, "wb") as out_f:
        for i, block in enumerate(blocks):
            voice = VOICE_A if i % 2 == 0 else VOICE_B
            print(f"[{i+1}/{len(blocks)}] {voice}: {block[:30]}...")
            communicate = edge_tts.Communicate(block, voice)
            async for chunk in communicate.stream():
                if chunk["type"] == "audio":
                    out_f.write(chunk["data"])

asyncio.run(generate_alternating_audio("input.md", "output.mp3"))
```

How it works:
1. Read the file and split it into blocks on blank lines.
2. For block `i`, use voice A if `i` is even, otherwise voice B.
3. Stream each block through `edge_tts.Communicate(block, voice).stream()` and append the `audio` chunks to one output file. MP3 frames can be concatenated directly, so no audio library is needed to join them.

> **Note:** If you repeat each block an odd number of times (e.g. 3), each repeat is read by a different voice than the one before it: A-B-A for the first block, B-A-B for the second, and so on.

For a single voice reading short text, one call is enough:

```python
await edge_tts.Communicate(text, "en-US-ChristopherNeural").save("out.mp3")
```

### 4. Compress the output

Edge-tts outputs **24 kHz mono MP3 at 48 kbps**. For speech, 16 kbps is still perfectly clear and makes the file about 3× smaller (e.g. a 15-minute recording drops from 5.3 MB to 1.8 MB):

```bash
ffmpeg -i "output.mp3" -ac 1 -ar 24000 -b:a 16k "output-compressed.mp3"
```

## Choosing a Voice: The Available Voice List

### How the list is obtained

Edge-tts can query the service for every voice it offers (300+ across all languages):

```python
voices = await edge_tts.list_voices()
en_voices = [v for v in voices if v["ShortName"].startswith("en-")]
```

Each entry is a dict that includes `ShortName` (e.g. `en-US-EricNeural`, which is what you pass to `Communicate`), `Gender`, and `Locale`. You can also list voices from the command line:

```bash
edge-tts --list-voices
```

The list comes from Microsoft's service, not from the library, so voices can be added or removed over time.

### Auditioning voices

This generates one file containing every English voice, sorted by name and announced as "Voice N. <name>…". Listen straight through and note the numbers you like:

```python
import asyncio
import edge_tts

async def generate_voice_demo(output_file):
    voices = await edge_tts.list_voices()
    en_voices = sorted((v for v in voices if v["ShortName"].startswith("en-")),
                       key=lambda v: v["ShortName"])

    with open(output_file, "wb") as out_f:
        for idx, v in enumerate(en_voices):
            name, gender = v["ShortName"], v["Gender"]
            text = f"Voice {idx + 1}. {name}. This is a {gender} voice."
            try:
                async for chunk in edge_tts.Communicate(text, name).stream():
                    if chunk["type"] == "audio":
                        out_f.write(chunk["data"])
            except Exception as e:
                print(f"Failed for {name}: {e}")

asyncio.run(generate_voice_demo("All_English_Voices_Demo.mp3"))
```

To get a separate clip per voice instead, call `await edge_tts.Communicate(text, name).save(f"{name}.mp3")` inside the loop.

### Voice naming

`<language>-<REGION>-<Name>[Multilingual|Expressive]Neural`

- **Region** sets the accent: `US`, `GB`, `AU`, `CA`, `IE`, `IN`, `NZ`, `ZA`, `NG`, `KE`, `TZ`, `PH`, `SG`, `HK`.
- **Multilingual** voices (e.g. `en-US-AndrewMultilingualNeural`) can switch languages mid-text.
- **Expressive** voices (e.g. `en-IN-NeerjaExpressiveNeural`) have more emotional range.

### The 47 English voices (as of 2026)

| Region | Female | Male |
|---|---|---|
| Australia (AU) | Natasha | WilliamMultilingual |
| Canada (CA) | Clara | Liam |
| United Kingdom (GB) | Libby, Maisie, Sonia | Ryan, Thomas |
| Hong Kong (HK) | Yan | Sam |
| Ireland (IE) | Emily | Connor |
| India (IN) | Neerja, NeerjaExpressive | Prabhat |
| Kenya (KE) | Asilia | Chilemba |
| Nigeria (NG) | Ezinne | Abeo |
| New Zealand (NZ) | Molly | Mitchell |
| Philippines (PH) | Rosa | James |
| Singapore (SG) | Luna | Wayne |
| Tanzania (TZ) | Imani | Elimu |
| United States (US) | Ana, Aria, Ava, AvaMultilingual, Emma, EmmaMultilingual, Jenny, Michelle | Andrew, AndrewMultilingual, Brian, BrianMultilingual, Christopher, Eric, Guy, Roger, Steffan |
| South Africa (ZA) | Leah | Luke |

Full names are `en-<REGION>-<Name>Neural`.

## What Works Best

- **Voices:** `en-US-ChristopherNeural` + `en-US-EricNeural`. Both are clear and measured. They're different enough to tell apart but similar enough that switching between them isn't jarring.
- **Alternative:** `en-US-ChristopherNeural` + `en-US-MichelleNeural` gives a stronger contrast between blocks if you prefer a male/female pair.
- **Structure:** short blocks separated by a blank line, with headings on their own line.
- **Output:** compress to 16 kbps mono for a small file that still sounds clear.

## Troubleshooting

- **Network errors / `NoAudioReceived`:** the service is online-only and occasionally rate-limits. Re-run, since the output file is overwritten from scratch.
- **A voice fails:** voices get retired. Check `edge-tts --list-voices` for the current names.
- **Blocks merged or split wrongly:** check the blank lines in the input. Exactly one blank line should separate blocks.
