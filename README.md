# TTS costs

## 19 Dec 2025

I've ranked TTS models by value for money (in my opinion) when reading this 4K character [podcast](podcast.txt).

| Rank | Model                              | Cost $ | Audio (s) | Time (s) | $ / hour | $ / MChars | $ / MTok |      Tokens |
| ---: | ---------------------------------- | -----: | --------: | -------: | -------: | ---------: | -------: | ----------: |
|    1 | [gemini-2.5-flash-preview-tts][O1] | 0.0609 |       242 |       95 |    0.906 |       14.9 |     10.0 | 918 + 6,046 |
|    2 | [gemini-2.5-pro-preview-tts][O2]   | 0.1480 |       294 |      203 |    1.812 |       36.1 |     20.0 | 918 + 7,352 |
|    3 | [gpt-4o-mini-tts-2025-12-15][O3]   | 0.0612 |       253 |       46 |    0.871 |       14.9 |     69.8 |             |
|    4 | [gpt-4o-mini-tts][O4]              | 0.0652 |       268 |       46 |    0.876 |       15.9 |     74.4 |             |
|    5 | [tts-1][O5]                        | 0.0614 |       256 |       44 |    0.864 |       15.0 |     70.0 |             |
|    6 | [tts-1-hd][O6]                     | 0.1228 |       257 |       62 |    1.728 |       30.0 |    140.0 |             |

[O1]: gemini-2.5-flash-preview-tts.opus
[O2]: gemini-2.5-pro-preview-tts.opus
[O3]: gpt-4o-mini-tts-2025-12-15.opus
[O4]: gpt-4o-mini-tts.opus
[O5]: tts-1.opus
[O6]: tts-1-hd.opus

I like the `algieba` (male) / `kore` (female) [Gemini voices](https://docs.cloud.google.com/text-to-speech/docs/gemini-tts#voice_options)
`ash` (male) / `nova` (female) [OpenAI voices](https://platform.openai.com/docs/guides/text-to-speech#voice-options).

For the Gemini TTS models, I used this request:

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg text "$(cat podcast.txt)" '{
    "contents": [{"role": "user", "parts": [{"text": $text}]}],
    "generationConfig": {
      "responseModalities": ["audio"],
      "speech_config": {"voice_config": {"prebuilt_voice_config": {"voice_name": "Kore"}}}
    }
  }')" \
  --output gemini-2.5-flash-preview-tts.json
jq -r '.candidates[0].content.parts[0].inlineData.data' gemini-2.5-flash-preview-tts.json | base64 --decode > gemini-2.5-flash-preview-tts.pcm
ffmpeg -f s16le -ar 24000 -ac 1 -i gemini-2.5-flash-preview-tts.pcm gemini-2.5-flash-preview-tts.wav
```

## 2 Nov 2025

The OpenAI [text-to-speech](https://platform.openai.com/docs/guides/text-to-speech) cost documentation is confusing.

As of 2 Nov 2025:

- [GPT-4o mini TTS](https://platform.openai.com/docs/models/gpt-4o-mini-tts) costs $0.60 / MTok input and $12.00 / MTok audio output according to the [model page](https://platform.openai.com/docs/models/gpt-4o-mini-tts) and the [pricing page](https://platform.openai.com/docs/pricing). They also estimate this to be ~1.5c per minute - both for input and output. It supports up to 2,000 tokens input.
- [TTS-1](https://platform.openai.com/docs/models/tts-1) costs $15 / MTok speech generated according to the [model page](https://platform.openai.com/docs/models/tts-1) but the [pricing page](https://platform.openai.com/docs/pricing) says it's $15 / MChars. No estimate per minute is provided. Is supports up to 4,096 characters input.
- [TTS-1 HD](https://platform.openai.com/docs/models/tts-1-hd) is twice as expensive as TTS-1

I wanted to find the approximate total cost for a typical text input measured per character and token.

I converted this [podcast](podcast.txt) with 4,096 ASCII characters and 877 tokens on [o200k_base](https://github.com/openai/tiktoken) using:

I ran:

```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg text "$(cat podcast.txt)" '{
    model: "tts-1",
    voice: "coral",
    input: $text
  }')" \
  --output tts-1.mp3
```

This took 46 seconds to generate and produced a 5.1 MB MP3 file (256 seconds)

To measure the cost, I ran:

```bash
curl "https://api.openai.com/v1/organization/costs?start_time=$(date -d '1 day ago' +%s)&project_ids=$PROJECT_ID&group_by=line_item" \
  -H "Authorization: Bearer $OPENAI_ADMIN_KEY" \
  -H "Content-Type: application/json"
```

This cost: USD 0.061425.

Then I ran:

```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg text "$(cat podcast.txt)" '{
    model: "gpt-4o-mini-tts",
    voice: "coral",
    input: $text
  }')" \
  --output gpt-4o-mini-tts.mp3
```

This took 44 seconds to generate and produced a 4.3 MB MP3 file (268 seconds).

When I ran the admin API call again, the costs did not reflect for 5 minutes. So I ran it the GPT-4o mini TTS call again with the same input. This took 44 seconds to generate a 4.3 MB MP3 file.

When I ran the admin API call again, the total cost was: USD 0.12942 audio output and USD 0.0010524 input -- which is the cost for 2 requests.

I also checked for TTS-1 HD. Here are the costs in USD:

| Model           | $ / MTok | $ / MChars | $ / hour | Time (s) | Audio (s) | Cost $ |
| --------------- | -------: | ---------: | -------: | -------: | --------: | -----: |
| GPT-4o mini TTS |     74.4 |       15.9 |    0.876 |       46 |       268 | 0.0652 |
| TTS-1           |     70.0 |       15.0 |    0.864 |       44 |       256 | 0.0614 |
| TTS-1 HD        |    140.0 |       30.0 |    1.728 |       62 |       257 | 0.1228 |

The GPT-4o mini TTS audio ouptut cost was 0.06471 for the input of 877 tokens, i.e. $73.8 / MTok. Since the actual cost is $12 / MTok, this is a 6.15x multiplier. I guess **1 input text token produces ~6 output audio tokens**.

In terms of quality:

- GPT-4o mini TTS: Very slightly robotic. Looser interpretation of text (e.g. says "October" when I write "Oct").
- TTS-1: Natural. Strict interpretation of text.
- TTS-1 HD: Very slightly more natural. Strict interpretation of text.

**I will likely use TTS-1 for now** given the cost difference is small and the quality is good enough.

---

Incidentally, the [usage API](https://platform.openai.com/docs/api-reference/usage/audio_speeches) did not show an GPT 4o mini TTS line items even after 20 minutes.

```bash
curl "https://api.openai.com/v1/organization/usage/audio_speeches?start_time=$(date -d '1 day ago' +%s)&project_ids=$PROJECT_ID&group_by=model" \
  -H "Authorization: Bearer $OPENAI_ADMIN_KEY" \
  -H "Content-Type: application/json"
```
