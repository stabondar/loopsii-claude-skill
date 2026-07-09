# Loopsii × Claude

Teach Claude to understand [Loopsii](https://loopsii.com) screen-recording links natively.

Every Loopsii watch link publishes a machine-readable manifest — the full timestamped transcript,
AI summary, chapters, pre-extracted screen frames, and a signed video URL. With this skill
installed, Claude answers questions about any Loopsii recording in seconds, with `[m:ss]`
citations, **without downloading the video or paying for transcription**.

## Install

**Claude Code**

```
/plugin marketplace add stabondar/loopsii-claude-skill
/plugin install loopsii-watch@loopsii
```

**Any Agent Skills host**

```
npx skills add stabondar/loopsii-claude-skill -g
```

**Manual** — copy `skills/loopsii-watch/` into your agent's skills directory.

## Use

Paste a Loopsii link and ask:

```
https://acme.loopsii.com/watch/AbC123… — what did they decide about pricing?
what's on screen at 2:10 in <link>?
```

No skill installed? The manifest is self-describing — any agent can simply GET
`<watch-link>/manifest.json` and follow the instructions inside.

## How it works

Loopsii recordings are transcribed on-device by the Loopsii Mac app and published with an
agent-native manifest (`type: loopsii.recording.context`). This skill just teaches your agent that
the manifest exists and how to use it well: transcript + summary for text questions, the hosted
frame strip for visual questions, and ffmpeg over the signed video URL for frame-level review.
