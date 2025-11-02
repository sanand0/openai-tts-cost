# OpenAI TTS costs

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
