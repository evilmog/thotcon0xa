# DISCLAIMER
Opinions expressed in this github repo are mine and mine alone.

IBM, X-Force Red, the Church of Wifi and any other organization take no responsibility for the contents of this repo.

No warrantees express or implied, , I am not responsible for property damage including damage to video cards, house fires, high data center bills etc.

There was going to be a tool but its broke so now you have a bunch of shell scripts

Consult a “professional” before attempting at home

This is purely for fast hashes like NTLM, MD4, MD5, SHA1, not salted hashes or slow hashes like bcrypt

My code sucks

# INTRODUCTION

-Infinite monkeys on infinite keyboards will eventually type your password
-I don’t have monkeys, I have GPU’s which are good enough
-I want to do everything I can to avoid brute force
-I want to try random things, they might just work
-Its time to share my password cracking methodology

# Combinator mode
Hashcats original mode was combinator, or –a 1 mode

There is a left side, and a right side

Given the inputs on the left side of ‘1234, 5678, 90ab’ and a right side of ‘test’ and ‘pass’ you get the output of ‘1234test’, ‘1234pass’, ‘5678test’, ‘5678pass’, ’90abtest’, and ‘90abpass’

```
./hashcat –m 1000 hashlist.txt –a1 –O –w 3 left.lst right.lst
```
- Run hashcat
- Use mode 1000 (NTLM)
- Combinator attack mode
- Optimized
- Workload Profile 3 (fast)
- Left and right list

# Password Splicing
Given the following passwords, extract prefixes and postfixes
```
Thotcon2019
1234Pass
Summer2019
EvilSpring2019
```

```
Thotcon2019 gives us Thot, Tho, 2019, 19, etc
1234Pass gives us 1, 12, 123, 1234, Pass, ass, ss, s, etc
Summer2019 gives us Summer, 2019, 19, 019, Sum, Summe, etc
EvilSpring2019 gives us Evil, Ev, 2019, 19, Spring2019, g2019 etc
```

Combined we get candidates like EvilSummer2019, 1234Spring2019, SummerPass

Basically we can extract prefix and postfix padding

# Password Splicing Implemented
```
#!/bin/bash
hu_path="/opt/utils/hashcat-utils/src"
work_path="/opt/cutb"

for i in {1..8}; do
 $hu_path/cutb.bin 0 $i < $1 | sort -u > $workpath/$i-first.txt
done

for i in {1..8}; do
  $hu_path/cutb.bin -$i < $1 | sort -u > $workpath/$i-last.txt
done

cat $workpath/*-first.txt $workpath/*-last.txt | sort -u > $workpath/cand.cutb
```

# improved password splicing
EvilSpring2019
This can expand into more words, such as Evil, Spring, 2019, ring, etc
Instead of prefix and postfix padding we can extract out of the center
Feed into -a 1 and profit

```
#!/bin/bash
hupath="/opt/utils/hashcat-utils/src"
hcpath="/opt/cracken2/hashcat"
workpath="/opt/evilmog/tmp"

echo "[+] Sorting Potfile"
cut -d: -f2- < $hcpath/hashcat.potfile | sort -u > $workpath/cand.lst
for i in {1..8}; do
  for s in $( eval echo {0..$i} ); do
    echo "[+] s: $s i:$i"
    $hupath/cutb.bin $s $i < $workpath/cand.lst | sort -u > $workpath/$i-first-seq.txt
  done
done
for i in {1..8}; do
  echo "[+] rev: $i"
  $hupath/cutb.bin -$i < $workpath/cand.lst | sort -u > $workpath/$i-last-seq.txt
done

echo "[+] final cat"
cat $workpath/*-first-seq.txt $workpath/*-last-seq.txt | sort -u > $workpath/cand.seq.cutb
echo "[+] final sort"
$hupath/expander.bin < $workpath/cand.seq.cutb | sort -u > $workpath/cand.seq.cutb.exp
```

# expander - fingerprinting
Expander takes a wordlist and expands it

Ilikehash123!

```
I
Il
Ili
Ilik
Ilike
Ilikeh
Ilikeha
Ilikehas… etc
```

By default it only expands words to 4 characters, you can and should increase this

Edit hashcat-utils expander.c

Edit LEN_MAX from 4 to 8 or even 10, run your potfile through it or your wordlists like rockyou.txt

Don’t forget to recompile

```
./hashcat –m 1000 –a 1 hashlist.txt cand.seq.cutb cand.seq.cutb
```

# standard attack methedology
```
./hashcat –m 1000 –O –w 3 hashlist.txt rockyou.txt –r rockyou-30000.rule
cut –d: -f2- < hashcat.potfile | sort –u > cand.lst
../hashcat-utils/src/expander.bin > cand.lst | sort –u > cand.exp
./hashcat –m 1000 –O –w 3 hashlist.txt –a 1 cand.exp cand.exp
```

repeat cut, expander and hashcat until stuff stops cracking

then run cutb password splicing followed by hashcat

```
./hashcat –m 1000 –O –w 3 hashlist.txt –a 1 cand.seq.cutb cand.seq.cutb
```

repeate splicing off candidates until nothing new comes out

# Prince Processor
PRobability INfinite Chained Elements

https://github.com/hashcat/princeprocessor

Basically a giant candidate generator

Fully pipelineable

Shuffle the wordlist order and new output comes out

Random Input = greatly varying output

So lets run some shenanigans

# Expander + Cutb + Prince = Fun

## Option 1 expander

```
shuf cand.exp | ../pp/src/pp.bin | ./hashcat64.bin –m 1000 –O –w 3 hashlist.txt –g 100000 –debug-mode=4 –debug-file=debug.txt
```

Translated means shuffle expanded candidates, pipe into prince processor, pipe into hashcat, use NTLM, optimized, workload profile 3 on hashlist.txt, generate 100000 random rules, turn on debug mode and capture basewords + rules + final words to a file for later processing


## Option 2 cutb

```
shuf cand.seq.cutb | ../pp/src/pp.bin | hashcat64.bin –m 1000 –O –w 3 hashlist.txt –g 100000 –debug-mode=4 –debug-file=debug.txt
```

Translated means shuffle password spliced candidates, pipe into prince processor, pipe into hashcat, use NTLM, optimized, workload profile 3 on hashlist.txt, generate 100000 random rules, turn on debug mode and capture basewords + rules + final words to a file for later processing


# Combinator + Cutb + Prince + Shuffle

```
../hashcat-utils/src/combinator.bin cand.seq.cutb cand.seq.cutb | shuf | ../pp/src/pp.bin | hashcat64.bin –m 1000 –O –w 3 hashlist.txt –g 100000 –debug-mode=4 –debug-file=debug.txt
```
Translated means run combinator on password spliced candidates, pipe into shuffle, pipe into prince processor, use NTLM, optimized, workload profile 3 on hashlist.txt, generate 100000 random rules, turn on debug mode and capture basewords + rules + final words to a file for later processing

# raking

## cut up the debug file
```
cut –d: -f1 debug.txt | sort -u > cand.lst
cut –d: -f3 debug.txt | sort -u >> cand.lst
cut –d: -f2 debug.txt | sort -u > debug.rule
```

## expander cutb cand.lst
```
../hashcat-utils/src/expander.bin < cand.lst | sort -u > cand.exp
```

## run hashcat with debug and and the rules
```
./hashcat64.bin -m 1000 -a 0 cand.lst –r debug.rule
```

## run hashcat expanded with debug rules
```
./hashcat64.bin -m 1000 -O -w 3 –a 1 cand.exp –j debug.rule cand.exp –k debug.rule
```

## go crazy

```
../hashcat-utiles/src/combinator.bin cand.exp cand.exp |  ./hashcat64.bin –m 1000 -O -w 3 –g 100000
```

# lets go really crazy

## run expander on hashcat candidates
```
cut -d: -f2- < $hcdir/hashcat.potfile | sort -u | $hcutils/expander.bin | sort -u > cand.exp
```

## run combinator to combine cand.exp together
```
$hcutils/combinator cand.exp cand.exp > combi.lst
```

## run PACK statsgen on the completed combi file
```
python $packpath/statsgen.py --maxlength=24 -o $workpath/cand.lst.mask --hiderare combi.lst
```

## run maskgen to generate masks
```
python $packpath/maskgen.py --pps=$pps --minlength=8 --minoccurrence=10 -t $rt -o $workpath/combi.lst.hcmask $workpath/combi.lst.mask
```

use pps of 160000000000
use rt of 360

## run hashcat attack mode 3
```
./hashcat –m 1000 –a 3 –O –w 3 hashlist.txt combi.lst.hcmask
```

## get coffee and come back in a few weeks

# Other insane ideas

Pipe prince into itself as in Prinception
- credit to jermei gosney aka epixoip

Run prepare and permute on your wordlist, sort for uniq and then pipe into prince or direct to hashcat

Rake forever, generate 100000 random rules, loop over your wordlists, setup debug, capture and repeat
- like generated2.rule

Grab an AI chatbot and pipe its input into PRINCE with random rules and shuffle

Pipe Slashdot, reddit, 4chan, urban dictionary into PRINCE, generate random rules, collect debug and repeat

## TL;DR Try RANDOM things, the crazier the better, mix in keyboard walks etc for a good time

