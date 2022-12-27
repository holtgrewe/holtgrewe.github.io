---
title: "Unix Sleight of Hand (2/?): Robust Column Extraction"
date: 2022-12-24
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
However, extracting a particular column from text files is surprisingly hard.
This blog post explores some robust solutions.

<!--more-->

{{< figure src="/posts/2022/2022-12-24-fishermen.jpg" width="50%" caption="'etching of three fisherman with a fishing rod' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}

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

```bash
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

```bash
$ cut -f 2,1 test.tsv
col1    col2
a1      a2
b1      b2
```

We cannot change the order of columns.
More on this later.

Note that we can also change the delimiter.

```bash
$ head -n 3 /etc/passwd | cut -d : -f 1,2
root:x
daemon:x
bin:x
```

Neat.
You can also cut out certain bytes.

```bash
$ head -n 3 /etc/passwd | cut -b 2-4
oot
aem
in:
```

## Interlude: Turn It Around

What if you are interested in the last bytes of each line?
Did you know about `rev`?

```bash
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

```bash
$ tac /etc/passwd | head -n 3
rtkit:x:114:123:RealtimeKit,,,:/proc:/usr/sbin/nologin
redis:x:113:122::/var/lib/redis:/usr/sbin/nologin
postgres:x:112:121:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
```

{{< figure src="/posts/2022/2022-12-24-laughing-cat.png" width="50%" caption="'a cat laughing madly as an etching' according to [[Dalle-E 2](https://labs.openai.com)]." >}}

## `awk` -- a Text Wrangler's Best Friend

Of course, you could write a little program that does the column extraction.
For example, you can pass a commands links to the Python interpreter like this:
`python -c "print('Hello'); print('World')"`.
However, why not try a programming language that is focused at string processing.
Modern Linux distributions ship with a couple of them preinstalled: sed, grep, awk, perl.

You will most likely know `grep` which allows you to find lines matching a certain pattern (not of use here).
Sed is the **s**tream **ed**itor and provides a very terse (you could say hard to read) language for manipulating lines either interpreted as fields or based on regular expressions.
With sed, we would probably express the extraction of a column by deleting all others or all text before and after the column.
This sounds not so helpful for our task.
Most people will know perl which is of course up to the task but quite bulky for such a simple task.

{{< figure src="/posts/2022/2022-12-24-lion.jpg" width="50%" caption="'a lion tamer having a lion jump through a burning hoop as an etching' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}

Let us thus look how to handle this in awk:

```bash
$ awk '{print $1}' test.tsv
col1
a1
b1
```

What does this do?
Awk applies the program `{print $1}` to the file `test.tsv`.
The program executes the block between the curly braces to each line of the input and ... prints the first field.
By default, `awk` will split the input at each whitespace.
The neat part of this is that awk will not interpret whitespace at the beginning of each line as a separator by default.
Thus, the result would be the same for the following file.

```text
   col1    col2    col3
  a1      a2      a3
b1      b2      b3
```

Note that `awk` is quite flexible.
So, to use the tabulator character as the field separator, you can pass `-F $'\t'` (on Bash shells for interpreting the `$` dollar sign).

```bash
$ awk -F $'\t' '{print $1}' test.tsv
col1
a1
b1
```

The result is the same for our file.
Consider the following example:

```bash
$ echo -e "col1\tcol2\tcol3" > test.tsv
$ echo -e "a 1\ta2\ta3" >> test.tsv
$ echo -e "b 1\tb2\tb3" >> test.tsv
$ cat test.tsv
col1    col2    col3
a 1     a2      a3
b 1     b2      b3
$ awk -F $'\t' '{print $1}' test.tsv
col1
a 1
b 1
```

We can also print more than one field:

```bash
$ awk -F $'\t' '{print $1, $2}' test.tsv
col1 col2
a 1 a2
b 1 b2
```

Hm, a bit disappointing, right?
Awk uses a single whitespace for the output field separator.
We can fix this by adding a `BEGIN` block that sets the output field separator `OFS` to the input field separator `FS`.

```bash
$ awk -F $'\t' 'BEGIN {OFS=FS} {print $1, $2}' test.tsv
col1    col2
a 1     a2
b 1     b2
```

Oh, and you can do much more with awk.
For example, [Jon Bentley's](https://en.wikipedia.org/wiki/Jon_Bentley_(computer_scientist)) *Programming Pearls* features a lot of interesting awk programs as does the [GNU Awk User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html).
If you are wrangling text, you might find the following examples interesting:

```awk
# run block for lines with field 1 not matching regex
($1 !~ /^col/) { ... }
# the same for the whole line (stored in $0)
($0 !~ /^col/) { ... }
# starting from line 2 (NR is the row number)
(NR > 1) { ... }
# filter based on field count stored in NF
(NF > 42) { ... }
# use $ to look at the **value** of the last field
($NF > 1) { ... }
# run before the first and after the last line
BEGIN { ... }
END { ... }
```

Let me close with a bonus tip.
Famous Heng Li of BWA (and other) fame has written a [bioawk](https://github.com/lh3/bioawk) that attempts to bring the simplicity and power of Awk to bioinformatics file formats.
