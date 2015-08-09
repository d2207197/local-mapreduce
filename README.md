# Local MapReduce

Simulating Hadoop MapReduce Streaming in a multicore computer. Any mapper, reducer compatible with Hadoop Streaming can work on this shell script.

## Performance Comparison

- single process

  Elapsed time: 7 mins 9 secs

  ```console
  $ time pv citeseerx_descriptions_sents.txt.300000 | python map.py | sort -k1 | python reduce.py > result_single
  37.6MiB 0:01:12 [ 528KiB/s] [====================================================================================================>] 100%
  pv citeseerx_descriptions_sents.txt.300000  0.06s user 0.18s system 0% cpu 1:12.81 total
  python map.py  156.55s user 2.34s system 76% cpu 3:27.56 total
  sort -k1  67.68s user 1.94s system 16% cpu 7:09.31 total
  python reduce.py > result_single  217.10s user 0.26s system 50% cpu 7:09.38 total
  ```

- local-mapreduce

  Elapsed time: 1 min 18 secs. Five times faster


  ```console
  $ rm -r result ; time pv citeseerx_descriptions_sents.txt.300000 | ./lmr 2m 16 'python map.py' 'python reduce.py' result
  37.6MiB 0:00:23 [1.58MiB/s] [====================================================================================================>] 100%
  pv citeseerx_descriptions_sents.txt.300000  0.01s user 0.03s system 0% cpu 23.789 total
  ./lmr 2m 16 'python map.py' 'python reduce.py' result  1024.95s user 11.64s system 1318% cpu 1:18.63 total
  ```

## Prerequisite

- Essential
    - bash
    - GNU parallel
    - sort
    - python

- Recommanded
    - pv
    
## Install

```console
$ git clone  --depth 1 https://github.com/d2207197/local-mapreduce.git
$ sudo ln -s local-mapreduce/lmr /usr/local/bin
$ hash -r
```
## Usage

```console
$ cat <data> | ./lmr <chunk size> <num of reducer> <mapper> <reducer> <output directory>
$ ./lmr <chunk size> <num of jobs> <mapper> <reducer> <output directory> < <data>
```

- `<chunk size>`: Split input data into chunks with `<chunk size>`. The number of chunks equals to the total number of mapper.
- `<num of reducer>`: Each output line from mappers would then be hashed into `<num of reducer>` different reducer.
- `<mapper>`, `<reducer>`: Any executable shell command can be mapper or reducer.
- `<output directory>`: The output directory.


## Mapper Output Format

Mapper reads input from Standard Input and prints `key<TAB>value` per line to Standard Output.


Example:

```
hello world		3,15 4,12 16,33
world peace		2,11 7,44 9,59
john doe		5,21 8,5 11,37
world peace		2,4 3,60 6,28
john doe		9,89 5,39 11,2
john doe		1,56 3,62 8,42
```

## Reducer Input Format


As in Hadoop Streaming, all lines with the same key will be grouped together and pass to the same reducer.

Example:

```
hello world		3,15 4,12 16,33
john doe		5,21 8,5 11,37
john doe		9,89 5,39 11,2
john doe		1,56 3,62 8,42
world peace		2,11 7,44 9,59
world peace		2,4 3,60 6,28
```

## Mapper Reducer Example 1: Word Count

Mapper: `tr -sc "a-zA-Z" "\n"`
Mapper testing:

```console
$ echo 'aaa bbb ccc\naaa bbb aaa' | tr -sc "a-zA-Z" "\n"
aaa
bbb
ccc
aaa
bbb
aaa
```

Reducer: `uniq -c`
Reducer testing:

```console
$ echo 'aaa bbb ccc\naaa bbb aaa' | tr -sc "a-zA-Z" "\n" | sort -k 1,1 -t $'\t' | uniq -c
      3 aaa
      2 bbb
      1 ccc
```

Run mapper and reducer with local mapreduce

```console
$ cat data | ./lmr 5m 8  'tr -sc "a-zA-Z" "\n"' 'uniq -c' out
$ cat out/* | head
 148601 a
  10605 A
     19 aa
      1 Aa
      9 AA
      1 aaai
      5 AAAI
      4 Aachen
      2 AAIA
     23 AAM
```



## mapper reducer Example 2：ngram count in python

### mapper

mapper *nc-map.py*

```python
#!/usr/bin/env python
import re
def tokens(str1): return re.findall('[a-z]+', str1.lower())

def to_ngrams( words, length):
    return zip(*[words[i:] for i in range(length)])  

import fileinput
from collections import Counter
for line in fileinput.input():
    words = tokens(line)
    for n in range(1, 6):
        ngrams_counts = Counter(to_ngrams(words, n))
        for ngram, count in ngrams_counts.iteritems():
            print  '{}\t{}'.format(' '.join(ngram), count)
```
	
mapper testing

```console
$ echo 'aaa bbb ccc\naaa bbb aaa' | python nc-map.py
bbb     1
ccc     1
aaa     1
aaa bbb 1
bbb ccc 1
aaa bbb ccc     1
bbb     1
aaa     2
aaa bbb 1
bbb aaa 1
aaa bbb aaa     1
```



### reducer

reducer *nc-reduce.py*

```python
#!/usr/bin/env python
import fileinput
from collections import Counter, defaultdict
ngram_counts = defaultdict(Counter)
for line in fileinput.input():
    ngram, count = line.split('\t', 1)
    ngram, count = tuple(ngram.split(' ')), int(count)
    length = len(ngram)
    ngram_counts[length][ngram] += count

for length in ngram_counts:
    for ngram, count in ngram_counts[length].iteritems():
        print '{}\t{}'.format(' '.join(ngram), count)
```

reducer testing

```console
$ echo 'aaa bbb ccc\naaa bbb aaa' | python nc-map.py | sort -k 1,1 -t $'\t' | python nc-reduce.py
bbb     2
ccc     1
aaa     3
aaa bbb 2
bbb ccc 1
bbb aaa 1
aaa bbb ccc     1
aaa bbb aaa     1
```

### run with localmapreduce  ###

```console
$ cat data | ./lmr 5m 16  'python nc-map.py' 'python nc-reduce.py' nc-out
$ cat nc-out/* | head
scoring 121
kazhdan 2
crex    3
blastsets       3
glycoside       1
brieflyreview   1
seriesparallel  2
stle    1
abar    1
paralleled      4
```


