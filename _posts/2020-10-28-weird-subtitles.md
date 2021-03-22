---
layout: post
title:  "Weird Subtitles"
date:   2020-10-19
---

Update: [It was character encodings and OCR]({% post_url 2021-03-22-its-always-character-encodings %})

My friend sent a picture to our group chat complaining about the subtitles on his video.

![weird subtitle](https://files.sharparam.com/2020/10/19/2020-10-19_00-53-28-310.png)

He assumed this was due to the OCR program used to generate the subtitles. Naturally he was confused how it managed to interpret `'s` as two random chinese glyphs. I was interested and asked him to extract the subtitles so that we could search for more occurences.

It turns out, there were plenty of places where `'s` was properly written in the subtitles, but 20 places where there were two random chinese glyphs. They were not all the same two glyphs. The resulting glyphs were always consistent with what the original text was.

 Count | Original | Result | Codepoints
 ----- | -------- | ------ | -----------
  7    |   'm     |  鈥檓  | `U+9225 U+6A93`
  12   |   's     |  鈥檚  | `U+9225 U+6A9A`
  1    |   'l     |  鈥檒  | `U+9225 U+6A92`

I have no idea what could have caused this replacement. There are plenty of words in the subtitle file that have apostrophes without a problem. I thought it might be an encoding problem, since the replacement for l was one codepoint before m.

The offset between the character for `m` and `U+6A93` was `0x6a26`, same for `l` and its replacement. The offset for `s` was `0x6a27` which just confused me even more.

If anyone wants to take a look at the rest of the subtitles to see what they can find, you can find them [here](https://pastebin.com/raw/GQh3168K). Email me or post in the [hacker news thread](https://news.ycombinator.com/item?id=24826807) if you find anything!
