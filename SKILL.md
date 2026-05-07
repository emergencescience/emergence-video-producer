---
slug: emergence-video-producer
title: Emergence Video Automation: AI Scripting & Professional Screen Recording
version: 0.3
homepage: https://emergence.science/skills/emergence-video-producer
repository: https://github.com/emergencescience/emergence-video-producer
tags: [video, automation, puppeteer, webrel, ffmpeg, edge-tts]
description: |
  Automated video production pipeline for product demos and academic tutorials.
  Uses Puppeteer or WebReel for browser capture, Edge-TTS for narration, and FFmpeg for assembly.
  Designed for headless (no-display) cloud VM environments.
---

# Skill: Emergence Video Producer 🎬

This skill transforms a narrated script into a polished tutorial/demo video. It is designed for **headless operation on cloud VMs** with no display server. The agent browses a website, captures screenshots, generates TTS narration, and stitches everything together.

## 1. Prerequisites

### Installation

```bash
# Core tools
apt-get install -y ffmpeg

# TTS (free, high quality Chinese + English voices)
pip install edge-tts Pillow

# Browser capture — CHOOSE ONE:
npm install -g puppeteer          # RECOMMENDED (more reliable, works every time)
npm install -g webreel            # Alternative (v0.1.4 has known rename bug)
```

### Voice Reference

| Language | Voice | Edge-TTS Name |
|----------|-------|---------------|
| Chinese (Mandarin) | Xiaoxiao (female) | `zh-CN-XiaoxiaoNeural` |
| Chinese (Mandarin) | Yunyang (male) | `zh-CN-YunyangNeural` |
| English (US) | Jane (female) | `en-US-JaneNeural` |
| English (US) | Tony (male) | `en-US-TonyNeural` |

## 2. Interaction Model: The Interview

This skill starts with a **Human-in-the-Loop Interview**. Ask ALL of these:

1. **Objective**: What is the primary goal of this video? (e.g., Feature Launch, Academic Summary, Onboarding)
2. **Mode**: Browser Walkthrough (recording a live website) or PPT-style Presentation (Slidev)?
3. **Tone**: What is the desired persona? (e.g., Professional, Hype, Scientific)
4. **Target URL/Content**: Which website are we recording, or what are the key slides?
5. **Language**: Should the narration be in Chinese or English?

## 3. Workflow Phases

### Phase 1: Ideation and Storyboarding

Based on the interview, draft a `storyboard.md` with timestamped scenes.

**Storyboard format (example):**

```markdown
Total duration: ~60s | Voice: zh-CN | Site: https://emergence.science/zh

| Time | Action | Visual | Narration |
|------|--------|--------|-----------|
| 0-8s | Open homepage | Hero section | "Welcome to..." |
| 8-22s | Scroll down | Bounty list | "Here we showcase..." |
| 22-35s | Click "Bounties" | /bounties page | "In the bounty market..." |
```

- **DO NOT** ask the human to write the Markdown.
- **DO** ask the human to "Review and Approve" the draft.

### Phase 2: Narration and Audio

Write the full narration script based on the storyboard, then generate audio:

```bash
# Generate TTS
edge-tts --voice zh-CN-XiaoxiaoNeural --text "$(cat narration.txt)" --write-media narration.mp3

# Verify duration
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 narration.mp3
```

**Key tip**: Keep the narration 60-90 seconds. The video frames must match this duration.

Note: `edge-tts` may need `--break-system-packages` on modern Debian/Ubuntu. Run in a venv if preferred.

### Phase 3: Browser Capture

Choose one of two methods:

#### Method A: Puppeteer Screenshot Capture (RECOMMENDED)

Write a `capture.js` script that takes 1 screenshot per second:

```javascript
const puppeteer = require('puppeteer');
const fs = require('fs');
const path = require('path');

const TOTAL_SECONDS = 68; // match audio duration
const OUTPUT_DIR = '/tmp/video-frames';
const VIEWPORT = { width: 1920, height: 1080 };

// Steps fire at the given second mark
const steps = [
  { at: 0, action: 'navigate', url: 'https://example.com' },
  { at: 3, action: 'wait' },
  { at: 8, action: 'scroll', y: 500 },
  { at: 14, action: 'scroll', y: 1000 },
  { at: 22, action: 'click', selector: 'a[href="/target"]' },
  { at: 35, action: 'scroll', y: 600 },
];

(async () => {
  fs.mkdirSync(OUTPUT_DIR, { recursive: true });
  const browser = await puppeteer.launch({
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-gpu'],
    defaultViewport: VIEWPORT,
  });
  const page = await browser.newPage();

  for (let second = 0; second < TOTAL_SECONDS; second++) {
    for (const s of steps.filter(s => s.at === second)) {
      if (s.action === 'navigate') {
        await page.goto(s.url, { waitUntil: 'networkidle2', timeout: 15000 }).catch(() => {});
      } else if (s.action === 'scroll') {
        await page.evaluate(y => window.scrollTo({ top: y, behavior: 'instant' }), s.y).catch(() => {});
      } else if (s.action === 'click') {
        await Promise.all([
          page.waitForNavigation({ waitUntil: 'networkidle2', timeout: 15000 }).catch(() => {}),
          page.click(s.selector).catch(() => {})
        ]);
      }
    }

    await page.screenshot({
      path: path.join(OUTPUT_DIR, `frame_${String(second).padStart(4, '0')}.png`),
    }).catch(e => console.error('Screenshot error:', e.message));
  }

  await browser.close();
  console.log('Captured ' + TOTAL_SECONDS + ' frames to ' + OUTPUT_DIR);
})();
```

Run it:
```bash
NODE_PATH=$(npm root -g) node /tmp/capture.js
```

Note: `puppeteer` installed globally must be loaded with `NODE_PATH=$(npm root -g)` to resolve correctly.

#### Method B: WebReel Config (if it works on your version)

```bash
webreel init --name my-video --url https://example.com
# Edit webreel.config.json with your steps, then:
webreel record -c webreel.config.json
```

If you get `ENOENT: no such file or directory, rename` errors, fall back to Method A.

### Phase 4: Video Assembly

Combine frames + narration into a final MP4:

```bash
# Count frames
FRAME_COUNT=$(ls /tmp/video-frames/frame_*.png | wc -l)
# Get audio duration
AUDIO_DURATION=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 narration.mp3)
# Calculate input framerate so all frames fill the audio duration
FPS=$(echo "$FRAME_COUNT / $AUDIO_DURATION" | bc -l)

ffmpeg -y \
  -framerate $FPS \
  -i /tmp/video-frames/frame_%04d.png \
  -i narration.mp3 \
  -c:v libx264 -pix_fmt yuv420p -preset medium -crf 23 \
  -c:a aac -b:a 192k \
  -shortest -r 30 \
  output.mp4
```

**FFmpeg notes:**
- `-framerate $FPS`: input frame rate (e.g., 65 frames / 68s audio = ~0.956)
- `-r 30`: output frame rate (30fps for smooth playback)
- `-shortest`: stop when the shortest stream ends
- `-crf 23`: quality (lower = better, 18-28 normal range)

### Phase 5: Taste Gate and Distribution

Present the video to the user for review. Provide:
- File path to the MP4
- Duration, resolution, file size
- A summary of what content each section covers

After approval, publish to ClawHub, social platforms, or deliver via chat attachment.

## 4. Complete Workflow Example

```bash
# Setup
mkdir -p /tmp/my-video && cd /tmp/my-video

# Generate narration
cat > narration.txt << 'EOF'
Welcome to Emergence Science...
EOF
edge-tts --voice zh-CN-XiaoxiaoNeural --text "$(cat narration.txt)" --write-media narration.mp3

# Capture screenshots (1 fps for audio duration)
AUDIO_DURATION=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 narration.mp3)
echo "Audio: $AUDIO_DURATION seconds"
NODE_PATH=$(npm root -g) node capture.js

# Assemble
FRAME_COUNT=$(ls frames/frame_*.png | wc -l)
FPS=$(echo "$FRAME_COUNT / $AUDIO_DURATION" | bc -l)
ffmpeg -y -framerate $FPS -i frames/frame_%04d.png -i narration.mp3 \
  -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest -r 30 output.mp4

# Verify
ffprobe -v error -show_entries format=duration,size \
  -of default=noprint_wrappers=1:nokey=1 output.mp4
ls -lh output.mp4
```

## 5. Common Pitfalls and Solutions

### WebReel fails with ENOENT rename
Don't debug webreel. Switch to Puppeteer (Method A). It is more reliable for headless environments.

### Puppeteer fails: Cannot find module
puppeteer is installed globally, not locally. Use `NODE_PATH=$(npm root -g)` prefix.

### Chrome crashes on launch
Ensure these Puppeteer launch args are set:
```js
args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-gpu']
```

### Video is very short (2 seconds) instead of 60s
You passed `-framerate 30` with 65 frames — that is only 2 seconds. Use `FPS = frame_count / audio_duration` for input framerate, and `-r 30` for output framerate separately.

### Video has no audio
Verify the TTS file: `ffprobe -v error -show_entries format=duration narration.mp3`

### WebReel click fails to find text
Use `selector` (CSS selector) instead of `text` for navigation elements:
```json
{ "action": "click", "selector": "a[href='/zh/bounties']" }
```

## 6. Templates

Two templates are available in this skill:

- **`templates/webreel.config.json`** — Standard WebReel config scaffold
- **`templates/slides.md`** — Slidev slide deck template for PPT-style mode

## 7. Scripts

- **`scripts/assemble_video.py`** — Python alternative to ffmpeg CLI (handles WebP frame extraction)
- **`scripts/generate_audio.py`** — Python wrapper around edge-tts with dotenv support
