---
title: "Like a Glove"
platform: "HackTheBox"
category: "ai-ml"
difficulty: "easy"
date: "2026-07-04"
flag: "htb{h4rm0n1ou5_hymn_0f_h1ghd1m3ns10n4l_subl1me_5ymph0ny_0f_num3r1cal_nuanc3_1n_tr3mend0u5_t4p3stry_0f_t3xtu4l_7r4n5f0rma7ion}"
---

# Like a Glove

## Summary

`chal.txt` contains 84 lines of the form `Like A is to B, C is to?` — analogies encoded
as **word-vector arithmetic** over the `glove-twitter-25` embedding model. Solving each
analogy as `D = B - A + C` (nearest neighbor by cosine similarity) and concatenating the
84 answers, in order, reconstructs the flag character-by-character.

## Solution

### 1. Parse the analogies and set up the embedding model

Each line matches `Like (A) is to (B), (C) is to?`. The challenge description states the
model is `glove-twitter-25`, available via `gensim.downloader`.

### 2. Solve each analogy as vector arithmetic and concatenate

Standard word2vec analogy resolution: `vec(D) = vec(B) - vec(A) + vec(C)`, then take the
single nearest vocabulary word to that vector. Note this uses `similar_by_vector` on the
raw computed vector (not `most_similar(positive=.., negative=..)`), which — unlike
`most_similar` — does **not** exclude the input words `A`/`B`/`C` from the candidate
results; some analogies resolve to one of their own inputs, which is intentional here.

```python
#!/usr/bin/env python3
import re
import unicodedata
import gensim.downloader as api

model = api.load("glove-twitter-25")

with open("chal.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()

results = []
for line in lines:
    line = line.strip()
    if not line:
        continue
    m = re.match(r"Like (.+?) is to (.+?), (.+?) is to\?", line)
    if not m:
        results.append("???")
        continue

    a_word, b_word, c_word = m.group(1).strip(), m.group(2).strip(), m.group(3).strip()
    if not all(w in model.key_to_index for w in [a_word, b_word, c_word]):
        results.append("???")
        continue

    # Analogy: D = B - A + C
    vec = model[b_word] - model[a_word] + model[c_word]
    answer = model.similar_by_vector(vec, topn=1)[0][0]
    results.append(answer)

raw_flag = "".join(results)
flag = unicodedata.normalize("NFKC", raw_flag)  # fullwidth digits (０-９) -> ASCII
print(flag)
```

Output:

```
RAW:  htb{h４rm０n１ou５_hymn_０f_h１ghd１m３ns１０n４l_subl１me_５ymph０ny_０f_num３r１cal_nuanc３_１n_tr３mend０u５_t４p３stry_０f_t３xtu４l_７r４n５f０rma７ion}
FLAG: htb{h4rm0n1ou5_hymn_0f_h1ghd1m3ns10n4l_subl1me_5ymph0ny_0f_num3r1cal_nuanc3_1n_tr3mend0u5_t4p3stry_0f_t3xtu4l_7r4n5f0rma7ion}
```

### 3. Normalize fullwidth Unicode digits

`glove-twitter-25`'s vocabulary (scraped Twitter text) keeps fullwidth Unicode digits
(`０１２３４５６７８９`, U+FF10-FF19) as distinct tokens from ASCII digits, so several
answers land on the fullwidth form. `unicodedata.normalize("NFKC", ...)` folds them back
to ASCII, which is required since the flag must be fully ASCII.

## Flag

```
htb{h4rm0n1ou5_hymn_0f_h1ghd1m3ns10n4l_subl1me_5ymph0ny_0f_num3r1cal_nuanc3_1n_tr3mend0u5_t4p3stry_0f_t3xtu4l_7r4n5f0rma7ion}
```
