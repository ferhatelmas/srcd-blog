---
author: vadim
date: 2016-09-13
title: "397 Languages, 18,000,000 GitHub repositories, 1.2 billion files, 20 terabytes of code: Spaces or Tabs"
draft: false
image: /post/tab_vs_spaces/intro.png
description: "Comprehensive study of spaces and tabs usage in source code in GitHub repositories"
---
Tabs or spaces. We are going to parse every file among all programming languages known by GitHub to decide which one is on top.

<a href="http://content.blog.sourced.tech/tabs_spaces/index.html">![image](/post/tab_vs_spaces/intro.png)</a>

This was inspired by Felipe Hoffa's analysis of 400k repositories and 14 programming languages ([here](https://medium.com/@hoffa/400-000-github-repositories-1-billion-files-14-terabytes-of-code-spaces-or-tabs-7cfe0b5dd7fd)).

[Interactive results presentation.](http://content.blog.sourced.tech/tabs_spaces/index.html)

[GitHub repository with the data.](https://github.com/src-d/tab-vs-spaces)

The rules
---------
* Data source: GitHub files stored in source{d}'s own GitHub mirror as of July 2016.
* Stars don't matter: we’ll consider every repository. Everybody votes disregarding elitism. It appears that the 80-20 principle holds for GitHub, so unpopular repos do not constitute a substantial bias after all.
* Every file is analyzed: there are a lot of small files in some languages; "hello, world" counts.
* No duplicates: forks are not taken into account. Hard forks (those not marked as a fork by GitHub but really are, the number is about 800k) are not counted as well. From one side, this does not guarantee uniqueness of each file in particular; from the other side, slightly different files from hard forks are excluded from the analysis. Since a typical GitHub hard fork is something like Linux kernel, it makes sense.
* One vote per line: we consider the sum of line votes. Some lines are indented with a mix of spaces with tabs. We’ll place such into the third category, "mixed".
* All languages: we’ll look into all files and determine the language using `simple-linguist` which is
source{d}'s super fast reimplementation of [github/linguist](https://github.com/github/linguist) in Go.
It's way more reliable than simply using file extensions. E.g. .cs files can be either C# or Smalltalk,
.h files belong to C, C++ and Obj-C.

Numbers
-------
The raw numbers are stored in JSON on [GitHub](https://github.com/src-d/tab-vs-spaces/blob/master/tabs_spaces.json).
The total number of languages processed is 397. The following table reflects the statistics for some randomly picked ones:

 language |  bytes  |  files  |  lines  |  mixed | spaces  |  tabs   
----------|--------:|--------:|--------:|-------:|--------:|--------:
JavaScript|  2,341GB|  253.62M|  58,906M|  1,174M|  37,690M|  7,595M
XML       |  2,479GB|   56.93M|  41,546M|    158M|  35,423M|  2,794M
PHP       |  1,069GB|  163.01M|  31,515M|  1,464M|  15,795M|  7,712M
HTML      |  1,309GB|   81.07M|  21,698M|    256M|   7,122M|  3,545M
JSON      |  1,125GB|   51.94M|  14,421M|     15M|  12,115M|    998M
C         |    505GB|   38.99M|  14,118M|    331M|   5,632M|  2,279M
Java      |    352GB|   72.36M|  10,285M|    390M|   5,483M|  2,251M
C++       |    326GB|   34.62M|   9,440M|     89M|   4,397M|  1,553M
Python    |    241GB|   34.23M|   6,140M|      5M|   4,126M|    178M
C#        |    137GB|   29.71M|   3,675M|     15M|   2,248M|    665M
Ruby      |     98GB|   49.49M|   2,965M|      6M|   2,141M|     77M
Go        |     21GB|    2.87M|     642M|    700k|      12M|    434M

<style>th { text-align: center; }</style>

How-to
------
Well, I have to admit that this is not something that one can do in his or her garage.
I used 32-node [Dataproc](https://cloud.google.com/dataproc/) Spark cluster in n1-highmem-4 configuration
(that is, 4 cores and 26 GB RAM). Normally I use preemptible nodes which are 3 times cheaper though can be
restarted at any time. Not at this time: Spark stores the reduction's intermediate results
in memory and I didn't have time to mess with the persistence option. I guess I could use
preemptible nodes if I was a Spark guru after all.

The job took about 2 days (left it running for the weekend). I had to find the best cluster parameters so
I recreated the cluster several times and this is where [source{d}'s Jupyter Cloud Storage backend](https://github.com/src-d/jgscm)
shined. Read more about how I use Dataproc with Jupyter in the [previous article](http://blog.sourced.tech/post/dataproc_jupyter/).
Doing everything in the same Python environment from your web browser with unlimited computing resources?
That's why I like Jupyter+Dataproc.

Here is the most important source code part which does the line indentation analysis:
```python
def extract_stats(name, session=requests):
    with tmpdir(name) as outdir:
        if not fetch_repo(name, outdir, session):
            return {}
        clusters = json.loads(subprocess.check_output(["slinguist"], cwd=outdir).decode("utf-8"))
        result = {}
        for lang, files in clusters.items():
            if lang == "Other":
                continue
            result[lang] = lr = {}
            tabs = spaces = mixed = srcbytes = srclines = 0
            for file in files:
                try:
                    with open(os.path.join(outdir, file), "rb") as fobj:
                        for line in fobj:
                            srclines += 1
                            if not line:
                                continue
                            p = 1
                            c = line[0]
                            if c == ord(b' '):
                                spaces += 1
                                while p < len(line):
                                    c = line[p]
                                    if c == ord(b'\t'):
                                        mixed += 1
                                        spaces -= 1
                                        break
                                    elif c != ord(b' '):
                                        break
                                    p += 1
                            elif c == ord(b'\t'):
                                tabs += 1
                                while p < len(line):                                
                                    c = line[p]
                                    if c == ord(b' '):
                                        mixed += 1
                                        tabs -= 1
                                        break
                                    elif c != ord(b'\t'):
                                        break
                                    p += 1
                        srcbytes += fobj.tell()
                except:
                    continue
            lr.update({
                "spaces": spaces,
                "tabs": tabs,
                "mixed": mixed,
                "bytes": srcbytes,
                "lines": srclines,
                "files": len(files)
            })
        return result
```
This approach feels better than counting single votes from files. E.g., read
[here](https://habrahabr.ru/post/308974/#comment_9784722) why (sorry, the discussion is in Russian).

I'd like to notice that the sum of lines indented with spaces, tabs and mix of them is
less than the overall number of lines since there are empty and unindented lines
as well.

As for the [interactive demo app](http://content.blog.sourced.tech/tabs_spaces/index.html),
I used good ol' [matplotlib](http://matplotlib.org/) to draw the initial SVG and the awesome
[d3.js](https://d3js.org/) for the rest. I applied [t-SNE](https://lvdmaaten.github.io/tsne/)
clustering to the language vectors so that similar ones appear near each other.
The radius of each pie chart is proportional to the square root
of the number of lines written in the corresponding language.
Special thanks goes to [Miguel](https://github.com/mvader) for turning my pathetic HTML into an eye-candy.

Which one is on top?
--------------------
Spaces are used for indentation in the majority of the languages. The exclusions
in the lower-left cluster are: Go, ActionScript, PostScript, Assembly (GAS),
Makefile, Mathematica and JSP.

More?
-----
Want more stories? How about writing them yourself? [Join us.](mailto:talent@sourced.tech)
