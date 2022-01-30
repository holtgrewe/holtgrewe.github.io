---
title: Thoughts on Checksums
date: '2022-01-29'
categories:
- checksums
author: Manuel Holtgrewe
---

> Give me checksums or give me death.
> 
> -- me (ca. 2014)

It's 2019 and checksums are pervasive everywhere...
Everywhere?
No, a small community somewhere north in - agh, forget about it!
I'm looking at you *bioinformatics*.

28 years after Ronald Rivest invented [MD5](https://en.wikipedia.org/wiki/MD5), 57 years after [CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), and 68 years after the "invention" of the [parity bit](https://en.wikipedia.org/wiki/Parity_bit) I'm still getting **files without checksums**.

## What is a checksum?

For those dear readers who live in caves or under rocks (or both as I'm occasionally doing), a good example is [ISBN](https://en.wikipedia.org/wiki/International_Standard_Book_Number), you know the number they print on the back of books (*books* are still universally known, right?).
The wisdom of Wikipedia tells us that if we have a ten-digit ISBN `isbn` (given as a Python `str`), then its first digit must be chosen such that

```python
0 == sum(i * x for i, digit in enumerate(isbn, 1)) % 11
```

That is, using 1-based counting, the sum of the product of the first to the last digit multiplied by its position *modulo* 11 must be zero.
Imagine you're on a phone call and need to make the other side write down the ISBN.
The most probable mistakes are that one digit is changed or that two adjacent digits are written down flipped.
And as helpful Wikipedia tells us, it can be proven *mathematically* that such errors will be detected by checking the condition from above.
That is, the first digit `isbn[0]` is a **checksum**.

## Checksum all the things!

![CHECKSUM ALL THE THINGS (image)](/post/2022/2022-08-27-checksum-all-the-things.jpg)

There are various uses for checksums but the easiest one is making sure that we have the data that we want without errors (or at least making it highly improbable that we have corrupted data).
[Digital signatures](https://en.wikipedia.org/wiki/Digital_signature) are only a fancy kind of checksum.

[Bit rot](https://en.wikipedia.org/wiki/Data_degradation) is a thing.
Data transmission errors are a thing.
Connections braking up are a thing.

> Listen, you're a nice guy but I don't trust you further than I trust me.
> **Pretty please** use checksums next time.
> 
> -- me (ca. 2019)

If I'm having a file `$file` and a checksum file `$file.md5` then I can verify the file by using the following to check the file.

```bash
# pushd $(dirname $file)
# md5sum --check $file.md5
```

Some people will tell you that MD5 is bad and you should really use SHA256.
Fine, just `s/md5/sha256/`, or `s/md5/sha1/` if you're retro-data-snobbish.
However, please give me a checksum.
I've seen corrupted data too often.

## Computing checksums is slow/hard!

No, it is not.

- Consider [hashdeep](http://md5deep.sourceforge.net) for computing checksums of directory trees.
- Just use `parallel -t -j $CORES 'pushd {/}; md5sum {.} >{.}.md5' ::: $(find $path -type f | grep -v '\.md5$')` (or `sha256sum`...) for parallelising checksums.

## Last resort: built-in checksums of `.gz` files

**Pretty please** give me checksum-ed files.
Using `md5sum --check` is fast and convenient.

Otherwise, I'll have to fall back to the following.

```bash
# gzip --verbose --test $gzfile
```

While it works it is more time-consuming than `md5sum --check` plus `gzip` does not like [bgzip](http://www.htslib.org/doc/tabix.html) so I'll have to ignore the warnings:

```bash
# gzip --verbose --check $bgzfile 2>&1 | grep -v 'ignoring'
```

## One checksum file to rule them all?

All of the `*sum` tools allow to create one checksum file with on entry each for multiple files.
However, I'd argue that in most cases you want one checksum file for each payload file.
For example, if you have a file at `$path`, you'll want a `$path.md5`.

> Why?

The reason is simple.
More often than not you want to transfer a portion of a directory to another location.
This would leave you with the error-prone task of cutting out the section of the multi-file-checksum-file to transfer with the file.
If you have a partner checksum file, just copy `$file*` and you're done.

Of course there is an exception to every rule.
If you have tens of thousands of files to copy (I'm looking at you, Illumina raw output directories) then you don't want to double that file count.
Here, using tools such as `hashdeep` is highly beneficial in my opinion.

## Conclusion

I hope I could convince you that

- checksums are good,
- you should have checksums for everything,
- you should send around data around only with checksums,
- ideally each file having a partner checksum file.

[![XKCD on data rot](/post/2022/2022-08-27-xkcd-1683-data-degradation.png)](https://xkcd.com/1683/)

## Comments?

I don't do comments here but [posted my blog to reddit](https://www.reddit.com/r/bioinformatics/comments/cwcf1j/thoughts_on_checksums/) where they do.

## Edits

- I used `gzip --check` where I should have used `gzip --test` and now fixed this (thanks to Epistaxis at reddit).
