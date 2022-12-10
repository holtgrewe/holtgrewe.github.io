---
title: "Unix Sleight of Hand (2/?): Robust Column Extraction"
date: 2022-12-10
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

Let us prepare a clean TSV file first.

```bash
$ echo -e "col1\tcol2\tcol3" > test.tsv
$ echo -e "a1\ta2\ta3" >> test.tsv
$ echo -e "b1\tb2\tb3" >> test.tsv
$ cat test.tsv
col1    col2    col3
a1      a2      a3
b1      b2      b3
```

What do these commands to?
The `-e` flag enables escape characters, that is `\t` will be translated to a tabulator.
With `> test.tsv` we create the file `test.tsv` and overwrite the file if it already exists.
With `>> test.tsv` we append a new line to the file.
The `cat shows` us the file contents.
Wonderful.

Let us take a test drive.

```
$ cut -f 1 test.tsv
col1
a1
b1
$ cut -f 1,2 test.tsv
col1    col2
a1      a2
b1      b2
```

Neat, we can also print more than one column.
However, the following might disappoint you.

```
$ cut -f 2,1 test.tsv
col1    col2
a1      a2
b1      b2
```

We cannot change the order of columns.
More on this later.

Note that we can also change the delimiter.

```
$ head -n 3 /etc/passwd | cut -d : -f 1,2
root:x
daemon:x
bin:x
```

Neat.
You can also cut out certain bytes.

```
$ head -n 3 /etc/passwd | cut -b 2-4
oot
aem
in:
```

## Interlude: Turn It Around

What if you are interested in the last bytes of each line?
Did you know about `rev`?

```
$ head -n 3 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
$ head -n 3 /etc/passwd | rev
hsab/nib/:toor/:toor:0:0:x:toor
nigolon/nibs/rsu/:nibs/rsu/:nomead:1:1:x:nomead
nigolon/nibs/rsu/:nib/:nib:2:2:x:nib
```

Let that sink in a bit.
You can reverse each line to extract the n-th field or byte of each line.
I find this really neat.

By the way, did you now about `tac`?
`tac` is the opposite of `cat` and allows you to print a text file line by line, starting with the last one and ending with the first one.

```
$ tac /etc/passwd | head -n 3
rtkit:x:114:123:RealtimeKit,,,:/proc:/usr/sbin/nologin
redis:x:113:122::/var/lib/redis:/usr/sbin/nologin
postgres:x:112:121:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
```

{{< figure src="/posts/2022/2022-12-10-laughing-cat.png" width="50%" caption="'a cat laughing madly as an etching' according to [[Dalle-E 2](https://labs.openai.com)]." >}}


## `awk` -- a Text Wrangler's Best Friend

{{< figure src="/posts/2022/2022-11-28-lion.jpg" width="50%" caption="'a lion tamer having a lion jump through a burning hoop as an etching' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}
