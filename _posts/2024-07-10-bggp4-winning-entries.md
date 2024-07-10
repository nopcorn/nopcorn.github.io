---
layout: post
title: My Binary Golf Grand Prix 4 (BGGP4) Winning Entries
author: nopcorn
description: A cheeky interpretation of the rules allowed me to win the python and shell challenges
summary: A cheeky interpretation of the rules allowed me to win the python and shell challenges
tags: [bggp4, challenge, python, bash]
---

> - The Binary Golf Grand Prix (BGGP) is a yearly code golf challenge
> - I won the `python` and `shell` categories of BGGP4 by cheekily interpreting the rules&nbsp;👑

The Binary Golf Grand Prix (BGGP) is an annual competition that began in 2020, challenging participants to create the smallest possible binary files that perform specific tasks. Organized by [@netspooky](https://x.com/netspooky), the event builds on the concept of code golf, focusing on the creation of minimal executable binaries across various formats and architectures. 

Participation in BGGP has grown steadily, with numerous entries each year showcasing innovative techniques and deep technical knowledge. The event has become a highlight in the niche community of binary golf enthusiasts, blending creativity with technical prowess. In 2023, BGGP4 opened up entries to non-binary formats which allowed many more people to participate. 

The [Binary Golf Grand Prix 4](https://binary.golf/4/) (BGGP4) had specific rules that challenged participants to create the smallest executable files that could replicate themselves with an additional few constraints:

- The file must print or return the string `4`
- The replica it creates must be called `4`
- It must not execute its replica

I decided to try my hand at the `python` and `shell` categories.

## Python (42 bytes)

My `python` entry relied on a few common tricks such as loading all objects in the `shutil` module using the `*` symbol (fun fact: the space after `import` is optional). This gives us access to the `copy()` function, which only takes up 4 bytes. We then use the builtin `__file__` variable as the first argument and the string `'4'` as the name of the destination file. 

```python
from shutil import*
copy(__file__,'4');4/0
```

Finally we get to the cheeky ace up my sleeve in order to get python to print `4` to stdout. The rules just stated `4` needed to be printed, it didn't state that _other_ output couldn't accompany it. The shortest way I found of producing this sort of output is to try dividing 4 by 0, which will cause a `ZeroDivisionError` exception and terminate the script. I got [confirmation from the organizers](https://infosec.exchange/@const/110986639752255684) of the competition that this was within the spirit of the rules.

## Bash (10 bytes)

Armed with a technical workaround for one of the rules, I tried the `shell` category. Most of the other competitors were using `cp` to move the script and `echo` to print out the number 4. I found that if you give `cp` the verbose flag, it will happily tell you the names of the files it is operating on - which prints out `4`.

```bash
cp -v $0 4
```

Reviewing the other entries I noticed an efficiency which would have saved me a single byte. [@caioluders](https://twitter.com/caioluders/) submitted an 11 byte entry using `*` in place of `$0` which selects all files in the current directory (see [globbing](https://tldp.org/LDP/abs/html/globbingref.html)). Had they also used the verbose flag, they would have won!


__BGGP4 was a fun way to spend a few hours - big props to @netspooky and the rest of the team!__

![](/assets/img/happy-cat.jpg)