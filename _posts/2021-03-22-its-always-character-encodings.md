---
layout: post
title:  "It鈥檚 Always Character Encodings"
date:   2021-03-22
---

Recently a friend of mine brought up my last post on [weird subtitles]({% post_url 2020-10-28-weird-subtitles %}) in a group chat which made me want to figure out the issue again. This time, I had a few more ideas. I hoped that this issue would be somewhat widespread so that I could get more samples of messed up characters, so I did a Google search for `鈥檓` (which in the last post we saw replaced `'m`) and I found over 1 million results. This got my interested again and I decided I had to figure out what was going on.

I went through the results looking for substantial pieces of writing that contained a lot of these so I could get more samples. I found a few posts that had these errors and I kept adding them to my table. Similar to the last post, a lot of these replacement Chinese characters had a Unicode codepoint that was a common fixed distance away from the character they replaced. Based on this a predicted the replacement for a few more characters and verified they actually occurred by searching them up and finding reasonable results.

Eventually my table looked like this. This is an exerpt from a much larger table.

 original | replacement | orginal value | replacement value | difference
 -------- | ----------- | ------------- | ----------------- | ----------
 e        | 淓          | 101           | 28115             | 28014     
 f        | 攆          | 102           | 25862             | 25760     
 h        | 攌          | 104           | 25868             | 25764     
 l        | 檒          | 108           | 27282             | 27174     
 l        | 渓          | 108           | 28179             | 28071     
 n        | 淣          | 110           | 28131             | 28021     
 s        | 檚          | 115           | 27290             | 27175     
 t        | 檛          | 116           | 27291             | 27175     

A lot of characters had similar or identical differences, but it was still inconsistent. And if you notice, I put down l twice because I found two replacements for it which seemed to give me reasonable search results, we'll come back to this later. All the replacement characters were preceded by `鈥` and I *thought* they were all replacements for the original character preceded by an apostrophe.

## A New Plan

After gathering a lot of instances of the issue, all I learned was that this was pretty common. I was starting to suspect character encoding issues again. I considered this before, but never really tested it. There are several hundred [character sets](http://www.iana.org/assignments/character-sets/character-sets.xhtml) registered with IANA. My idea was to brute force encoding and decoding these Chinese characters as every pair of them. [Python](https://docs.python.org/3/library/codecs.html#standard-encodings) supports about 100 of them. Before testing every pair, I selected only the encodings that mentioned English or Chinese since that was the context in which these issues were occurring.

I made a simple nested for loop for going through each of my selected encodings and simply printing out any that had the missing `m` in it that we were expecting.

```python
char_encoding = [
    "ascii", "big5", "big5hkscs", "cp037", "cp437",
    "cp500", "cp950", "gb2312", "gbk", "gb18030",
    "hz", "iso2022_jp_2", "johab", "koi8_r", "koi8_t",
    "koi8_u", "kz1048", "mac_cyrillic", "mac_greek",
    "mac_iceland", "mac_latin2", "mac_roman", "mac_turkish",
    "ptcp154", "shift_jis", "shift_jis_2004", "shift_jisx0213",
    "utf_32", "utf_32_be", "utf_32_le", "utf_16",
    "utf_16_be", "utf_16_le", "utf_7", "utf_8", "utf_8_sig"
]

bad_text = "鈥檓"

for c1 in char_encoding:
    for c2 in char_encoding:
        try:
            r = bad_text.encode(c1).decode(c2)

            if "m" in r:
                print(c1, c2, r)
        except Exception:
            pass
```

We got something! My little script quickly printed out many lines, one of which was `gb18030 utf_8 ’m` which was exactly what we wanted. It also shows us GBK but GB18030 is a superset of GBK and supersedes it. What seems to be happening is that UTF-8 data was being wrongly interpreted as GB18030.

There is something to notice here. That apostrophe is not your standard ASCII `U+0027` [APOSTROPHE](https://fileformat.info/info/unicode/char/27). No, this was far worse. This is a fancy Unicode `U+2019` [RIGHT SINGLE QUOTATION MARK](https://fileformat.info/info/unicode/char/2019). The [GB18030](https://en.wikipedia.org/wiki/GB_18030) character encoding formats codepoints as either 1, 2, or 4 bytes. If a byte is within the range `00 - 7F`, then it is interpreted as ASCII. So misinterpreting the apostrophe as GB18030 doesn't have any side-effects since it is valid UTF-8 and GB18030. However, if a byte is outside that range, then depending on the byte immediately following it, it may be 2 or 4 bytes long. The Wikipedia page has a whole table explaining when you need four bytes, but for our case, all we need to know is that if the two bytes match the pattern `81–FE 40–FE` then they are just two bytes and not part of four bytes.

Our fancy single quotation mark is actually 3 bytes, `E2 80 99`, when encoded in UTF-8. The first two bytes will be interpreted as a whole GB18030 character since `E2 80` matched the pattern from earlier. In our previous example with `m` the next two bytes would be `99 6D` which also matched the pattern. `’m` is transformed into `鈥檓`.

## Wrapping Up

Finally we understand how these characters arise. Someone or some program entered UTF-8 formatted text somewhere that expected GB18030 formatted data. This text used fancy Unicode single quotes instead of ASCII apostrophes. This could be because of an OCR program that would sometimes use either one when scanning text. Or it could be from a word processor that inserts the fancy single quotes when you try to insert an apostrophe. Any of these scenarios is possible.

It seems like a simple mistake that could happen. Treating UTF-8 text as your native GB18030 and moving on. But this would never happen twice, right? Welll unfortunately for us we can find out!

To purposely corrupt some text we can just do `"don’t".encode("utf-8").decode("gb18030")` which gives us `"don鈥檛"`. Searching this in Google gives us 1,650,000 results. Now we can corrupt the same text again. This would be as if you interpreted the utf-8 text as gb18030, converted it into utf-8, and then reinterpreted it as gb18030. In that case we get another valid string `"don閳ユ獩"`. Searching this one up only returns 7,060 results. But that still means that this isn't a completely uncommon thing. We can't go a third time through because at that point our utf-8 bytes are no longer valid gb18030.

## Revisiting The Table

Now that we know how to purposely break or fix text, we can go back to our original table and see what I got right.

 guessed original | replacement | actual orginal
 ---------------- | ----------- | --------------
 'e               | 鈥淓        | “E            
 'f               | 鈥攆        | —f            
 'h               | 鈥攌        | —k            
 'l               | 鈥檒        | ’l            
 'l               | 鈥渓        | “l            
 'n               | 鈥淣        | “N            
 's               | 鈥檚        | ’s            
 't               | 鈥檛        | ’t            

For the most part, I correctly guessed the letter based on context. But I failed to get the capitalization correct. I also didn't realize that the preceding character didn't always have to be an apostrophe. It just had to be any 3 byte utf-8 encoded Unicode character that could be interpreted as valid GM18030. These just happen to be most commonly single quotes, double quotes, and dashes as far as my research could find.

I think the common differences with the characters is mostly coincidental. The character mapping from Unicode codepoints to GM18030 is just done with a premade map and isn't mapped linearly. So any characters that had the same difference just happened to be decoded to a linear portion of the character encoding.
