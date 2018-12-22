+++
title = "Advent of Code 2015, Day 5: Doesn't He Have Intern-Elves For This?"
date = 2018-12-22T14:47:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

Now that Santa has finished disappointing all the kids by putting worthless
altcoins in their stockings, it's time to check his list twice and find the
naughty and nice... strings.

> Santa needs help figuring out which strings in his text file are naughty or nice.
> 
> A __nice string__ is one with all of the following properties:
> 
> * It contains at least three vowels (`aeiou` only), like `aei`, `xazegov`, or `aeiouaeiouaeiou`.
> * It contains at least one letter that appears twice in a row, like `xx`, `abcdde` (`dd`), or `aabbccdd` (`aa`, `bb`, `cc`, or `dd`).
> * It does not contain the strings `ab`, `cd`, `pq`, or `xy`, even if they are part of one of the other requirements.
> 
> For example:
> 
> * `ugknbfddgicrmopn` is nice because it has at least three vowels (`u...i...o...`), a double letter (`...dd...`), and none of the disallowed substrings.
> * `aaa` is nice because it has at least three vowels and a double letter, even though the letters used by different rules overlap.
> * `jchzalrnumimnmhp` is naughty because it has no double letter.
> * `haegwjzuvuyypxyu` is naughty because it contains the string `xy`.
> * `dvszwmarrgswjxmb` is naughty because it contains only one vowel.
> 
> How many strings are nice?

## Analysis

<!--more-->
You are given a set of strings that you need to apply some criteria to.

There are two general ways in which this problem can be tackled:

* Loop through the strings, tracking the criteria byte-by-byte
* Use regular expressions

Neither path is necessarily easy. If you loop through the strings manually, there
will be a lot of backreferencing of parts of the string you've already scanned
looking for matches. If not done carefully, these loops will quickly drag your
runtime up.

Finding a solution using regular expressions requires using advanced regular
expression capabilities that are not universally available. In particular, you
need the capability to do both positive and negative lookahead assertions as
well as backreferences. Neither of the POSIX flavors nor the Google RE2 engine 
which backs the Go `regexp` package support any of these features. You will need
to use PCRE or a similar backtracking engine.

That said, I will discuss using PCRE to find the solution because of its
ubiquity, and the fact that I believe applying the compiled regular expression
to each string ultimately ends up being faster than manually looping and
backtracking through the strings.

For the first part of this problem, there are three criteria:

* at least three vowels (`aeiou`)
* at least one repeating character
* does not contain one of the substrings `ab`, `cd`, `pq`, `xy`.

In order to check all three criteria with one regular expression, we need to
use the positive and negative lookahead functionality to combine rules.

For the vowels criteria, we need three vowels anywhere in the string. This can
be accomplished with the subpattern `(?=(?:.*[aeiou]){3,})`. Further
breaking down this pattern, we are looking for a character in the vowel class
(`[aeiou]`), to appear three or more times (`{3,}`), with zero or more
characters (`.*`) between them. Because we are matching both the vowel and
the potential characters between them, we wrap the characters and the vowel in
a non-matching group (`(?:)`), to get the total match (`(?:.*[aeiou])`), which
we then further wrap in a positive lookahead (`(?=)`) in order to allow further
matching on the already matched area, resulting in `(?=(?:.*[aeiou]){3,})`.

For the repeating character criteria, we need to use a backreference, which
results in the pattern `(?=.*(.)\1)`. Breaking this down, we are searching for 
any character (`.`) and then wrapping it in a group (`()`) so that it can be
backreferenced. Then, we actually perform the backreference (`\1`), which 
combines into the pattern `(.)\1`. Because the dot would greedily match the
first character regardless of whether there was a following match, we need to
add a wildcard to the front of the pattern `.*(.)\1`. Finally, we again wrap in
a positive lookahead (`(?=)`) so that we can match the same text again, to give
a subpattern of `(?=.*(.)\1)`.

Finally, we have a set of substrings to not match, which can be accomplished
with the subpattern `(?!(?:ab|cd|pq|xy))`. To assemble this, we take each of
our subpatterns and join them with the pipe (`|`) separator to create a group
which matches any of the criteria (`ab|cd|pq|xy`). We wrap this in a 
non-matching group (`(?:)`) so that it does not keep a reference, to achieve
`(?:ab|cd|pq|xy)`. Finally, we wrap this in a negative lookahead (`(?!)`), which
asserts that the pattern is _NOT_ found in the proceeding string, to achieve the
full subpattern `(?!(?:ab|cd|pq|xy))`.

Finally, we join these subpatterns together `(?!(?:ab|cd|pq|xy))(?=.*(.)\1)(?=(?:.*[aeiou]){3,})`.
Since we ostensibly want to match the whole string assuming the other criteria
match, we add a wildcard (`.*`) to the end (since the subpatterns are all
lookaheads), and then add beginning-of-string (`^`) and end-of-string (`$`)
markers, to create a final pattern of `^(?!.*(?:ab|cd|pq|xy))(?=.*(.)\1)(?=(?:.*[aeiou]){3,}).*$`.

The actual complexity will depend on the mechanics of the regular expression
engine and how it handles the lookaheads. For our purposes, however, we are
doing one operation per input line, for a complexity of _O(n)_.

### C
{{< highlight c >}}
#include <openssl/md5.h>
FILE *input = fopen(input_file, "r");

const char *pcre_err;
int pcre_erroffset;

char *nice_pattern = "^(?!.*(?:ab|cd|pq|xy))(?=.*(.)\\1)(?=(?:.*[aeiou]){3,}).*$";
pcre *nice = pcre_compile(nice_pattern, 0, &pcre_err, &pcre_erroffset, NULL);

int nice_count = 0;
char buf[64];
int ovector[30];
while(fgets(buf, 64, input) != NULL) {
    int rc = pcre_exec(nice, NULL, buf, strlen(buf), 0, 0, ovector, 30);
    if (rc >= 0)
        nice_count++;
}
{{< /highlight >}}

### Optimization
Testing would be required to determine if a regular expression or manual
backtracking ends up being faster. That said, if we did choose the manual
backtracking route, our optimizations would involve ensuring we are making as
few loops as possible.

---

Santa rightly realized his arbitrary rules were silly, and replaced them with
arbitrary rules.

> Realizing the error of his ways, Santa has switched to a better model of determining whether a string is naughty or nice. None of the old rules apply, as they are all clearly ridiculous.
> 
> Now, a nice string is one with all of the following properties:
> 
> * It contains a pair of any two letters that appears at least twice in the string without overlapping, like `xyxy` (`xy`) or `aabcdefgaa` (`aa`), but not like `aaa` (`aa`, but it overlaps).
> * It contains at least one letter which repeats with exactly one letter between them, like `xyx`, `abcdefeghi` (`efe`), or even `aaa`.
> 
> For example:
> 
> * `qjhvhtzxzqqjkmpb` is nice because is has a pair that appears twice (`qj`) and a letter that repeats with exactly one letter between them (`zxz`).
> * `xxyxx` is nice because it has a pair that appears twice and a letter that repeats with one between, even though the letters used by each rule overlap.
> * `uurcxstgmygtbstg` is naughty because it has a pair (`tg`) but no repeat with a single letter between them.
> * `ieodomkazucvgmuy` is naughty because it has a repeating letter with one between (`odo`), but no pair that appears twice.
> 
> How many strings are nice under these new rules?

## Analysis

We can use the same methods to analyze the strings for the second part.

Solving with regular expressions, we will again use lookaheads to achieve
checking multiple rules in the same expression.

We have two criteria in this part of the problem:

* a two-or-longer character substring that repeats
* two identical characters with any other character between.

The repeating substring rule can be satisifed by the subexpression
`(?=.*(..).*\1)`. Breaking this down, we search for any two characters (`..`)
in a group (`()`). Then we search for zero or more characters (`.*`), and
finally use a backreference to match the original pattern (`\1`), to achieve
a subexpression `(..).*\1`. As before, the dot is greedy so we need to prefix
the expression with a wildcard `.*`, and then wrap the whole expression in a
positive lookahead (`(?=)`) for a final subexpression of `(?=.*(..).*\1)`.

The two identical characters separated by one other character rule is matched
by the subexpression `(?=.*(.).\2)`. We first match any character (`.`) in a
group (`()`) so that it can be backreferenced. Then we match any other character
(`.`), and finally add a backreference to the first matched character
(`\2`--we use 2 here because it will be the second matched group in the full
expression) to achieve `(.).\2`. Again we need to prefix with a wildcard (`.*`) 
due to the greedy dot, and then wrap in the positive lookahead for a final
subexpression of `(?=.*(.).\2)`.

We bring the subexpressions together in the same manner as before, adding a
wildcard (`.*`) at the end and bracing the expression with the beginning- (`^`)
and end- (`$`) of pattern markers, for a full expression of 
`^(?=.*(..).*\1)(?=.*(.).\2).*$`. We can replace the previous expression
with this one using the same supporting code to get the result for part 2.

Next: _coming soon_