# fetch-youtube

`fetch-youtube` is an OpenClaw skill and CLI for extracting full YouTube transcripts reliably.

Instead of generating a short summary first, it returns the transcript text itself. That means OpenClaw can read the full content of the video directly and decide for itself how to analyze, summarize, translate, quote, or structure it.

## Why this exists

A lot of YouTube transcript tools fail on videos where normal caption fetching breaks, or only work when the uploader has explicitly provided subtitle tracks.

`fetch-youtube` takes a more resilient path:

watch page scrape -> `INNERTUBE_API_KEY` extraction -> InnerTube player API fallback across multiple client profiles -> caption XML download -> transcript text extraction

This makes it much better at recovering transcripts from videos where simpler libraries return `No transcript available`.

## Main selling points

It is built around three ideas.

First, it fetches the transcript itself, not just a summary. Because there is no summary layer in the middle, OpenClaw can inspect the entire spoken content and keep full context.

Second, it can often recover YouTube's automatic captions even when manually attached subtitles are not present. In practice, this is the big win. A video may look like it has no uploader-provided subtitle set, but YouTube auto-generated captions are still available through the InnerTube caption path.

Third, it outputs plain JSON with the transcript included, so it is easy to plug into other OpenClaw flows.

## What it can fetch

`fetch-youtube` can extract transcripts from:

- a single YouTube video URL
- a recent set of videos from a channel or handle
- a batch config for repeated runs

When transcript extraction succeeds, the JSON output includes a `transcript` field containing the full extracted text.

## How transcript recovery works

The core recovery flow is:

1. Fetch the YouTube watch page HTML
2. Extract `INNERTUBE_API_KEY`
3. Call `youtubei/v1/player`
4. Try multiple client identities in order:
   - `ANDROID`
   - `WEB`
   - `TVHTML5_SIMPLY_EMBEDDED_PLAYER`
   - `IOS`
5. Read caption track metadata from the player response
6. Download the caption XML from the selected `baseUrl`
7. Parse the XML into transcript text

This fallback approach is based on the same family of techniques used to recover captions in cases where normal transcript APIs are flaky.

## Why no summary is a feature

A summary throws information away.

If the goal is to let OpenClaw understand a video well, returning only a summary is weaker than returning the transcript. With `fetch-youtube`, OpenClaw can read the source material directly, which is better for:

- detailed analysis
- extracting exact claims or quotes
- translation
- structured output generation
- custom summaries tailored to the user's task
- reusing the transcript in other tools or agents

So this project is intentionally transcript-first, not summary-first.

## Usage

Single video:

```bash
./fetch-youtube --url "https://www.youtube.com/watch?v=VIDEO_ID"
```

Recent videos from a channel or handle:

```bash
./fetch-youtube --channel "@channel_handle" --hours 24
```

Batch mode with config:

```bash
./fetch-youtube --config config/channels.example.json --daily
```

You can also write output to a specific file:

```bash
./fetch-youtube --url "https://www.youtube.com/watch?v=VIDEO_ID" --output /tmp/fetch_youtube.json
```

## Output format

The tool writes JSON like this:

```json
{
  "generated_at": "2026-04-09T04:00:00+00:00",
  "items": [
    {
      "video_id": "...",
      "title": "...",
      "url": "...",
      "channel": "...",
      "duration": "12:34",
      "published": "20260408",
      "has_transcript": true,
      "metadata": {
        "view_count": 12345,
        "like_count": 678
      },
      "transcript": "full transcript text here"
    }
  ],
  "stats": {
    "total_videos": 1,
    "with_transcript": 1,
    "without_transcript": 0
  }
}
```

If transcript extraction fails, `has_transcript` becomes `false` and an error message is included.

## Add it to OpenClaw

This repository is structured as an OpenClaw skill.

The simplest way is to clone or copy this repository into your OpenClaw skills directory.

Example:

```bash
cd ~/clawd/skills
git clone https://github.com/ioridev/fetch-youtube.git
```

After that, OpenClaw can discover the skill from the `SKILL.md` metadata.

If you just want to run it directly, the entrypoint is:

```bash
./fetch-youtube
```

The main script is:

```bash
scripts/fetch_youtube.py
```

Typical OpenClaw-side use is: fetch the transcript first, then let OpenClaw read the transcript text directly for analysis, translation, extraction, or summarization.

## OpenClaw usage idea

A good pattern is:

1. run `fetch-youtube` on the target video
2. get JSON containing `transcript`
3. pass that transcript to OpenClaw as source material

That keeps the workflow transcript-first and avoids losing information through a premature summary layer.

## Practical note on auto captions

One of the main reasons to use this project is that uploader-provided subtitles are not required.

If YouTube has generated automatic captions for the video, `fetch-youtube` can often recover them even when ordinary transcript methods fail. That is the core value of the project.

## License

See `LICENSE`.
