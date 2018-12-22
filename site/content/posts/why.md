+++
Description = "Why e3b0c442?"
title = "Why e3b0c442?"
date = "2018-07-02"
markup = "markdown"
+++

The better question is... why not?

Not satisfied? Fine. I have an interest in cryptography, and e3b0c442 represents the first 16 bytes of an SHA256 hash on an empty input:

```bash
$ echo '' | shasum -a 256 -0 | awk '{print $1}'
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

```
<!--more-->