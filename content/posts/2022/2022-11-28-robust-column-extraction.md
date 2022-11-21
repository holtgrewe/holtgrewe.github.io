---
title: "Unix Sleight of Hand (2/?): Robust Column Extraction"
date: 2022-11-21
draft: true
categories:
  - Unix Sleight of Hand
tags:
  - bash
  - sleight of hand
author: Manuel Holtgrewe
images:
  - /posts/2022/2022-11-28-fishermen.jpg
---

Many Bioinformaticians spend a lot of their time handling text files.
Linux (and other Unixes) excel at processing text files and offer many tools for handling it.
However, etracting a particular column from text files is surprisingly hard.
This blog post explores some robust solutions.

<!--more-->

{{< figure src="/posts/2022/2022-11-28-fishermen.jpg" width="50%" caption="'etching of three fisherman with a fishing rod' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}

## Well-formatted data? `cut`!

In the case that you have well-formed data, `cut` is probably your best friend.
By default, `cut` assumes that you want to use the tabulator character for text separation.

```bash
$
```

## `awk` -- a Text Wrangler's Best Friend

{{< figure src="/posts/2022/2022-11-28-lion.jpg" width="50%" caption="'a lion tamer having a lion jump through a burning hoop as an etching' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}
