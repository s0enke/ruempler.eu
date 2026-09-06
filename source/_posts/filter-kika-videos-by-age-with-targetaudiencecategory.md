---
date: "2026-09-06 12:00"
title: "Filter KiKA videos by age with targetAudienceCategory"
categories: til
---

## Situation

You want to pull videos from a KiKA mediathek section, but only the ones aimed at
older kids — no preschool.

## Challenge

FSK looks like the field for the job. It isn't. Most of the catalogue carries no
rating at all, and where one exists it doesn't separate the age bands: the same
FSK 0 covers both a preschool cartoon and a live-action film for eleven-year-olds.

<!--more-->

## Solution

Every video document carries a second, undocumented field next to `fsk`:

```bash
curl -s https://www.kika.de/_next-api/proxy/v1/videos/<video-slug> \
  | jq '{title, fsk, targetAudienceCategory}'
```

**`targetAudienceCategory` is an ordinal audience band — higher means older — and
unlike FSK it is populated on everything.** Sampling a couple of hundred videos, it
sorts exactly the way you'd want: the preschool channel sits at 4, a bee cartoon at
6, boarding-school adventures at 9, the kids' news show at 10.

Don't read it as an age in years, though. That's the obvious guess and it's wrong:
the news show is documented for seven-year-olds and up but reports 10, the observed
values skip 5 and stop at 11, and one franchise had its series episodes at 4 and
its films at 6.

So "no preschool" is `targetAudienceCategory >= 9` — an empirical cut, not a
documented one.

To get every video in a section, the brand document points at a paginated listing:

```bash
curl -s https://www.kika.de/_next-api/proxy/v1/brands/<brand-slug> \
  | jq -r .videoSubchannel.videosPageUrl
```

Follow `.links.next` until it's gone.

One catch if you were hoping to filter in yt-dlp: its KiKA extractor keeps neither
field, so `--match-filter` can't see them without an extractor plugin.

hth :)
