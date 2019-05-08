---
layout: post
title: "Read-Writable Regular Expressions"
date: 2019-05-07 21:21:45 -0700
categories: regex documentation
---
Regex is great, right? It's concise, it's precise, and the process of developing
an expression that works just right is a hell of a lot of fun. Until you submit
a patch to your coworkers with your beautiful expression, and they leave you
comments like,

* "How does this regex work?"
* "What does this even do?"
* "Is this busproof? Nobody else on the team knows regex!"

It's probably not a good idea to encourage your coworkers to think critically
about what life would be like if you got hit by a bus, but the good news is that
you can simultaneously document your regular expression and teach your coworkers
some regex basics, if you comment them carefully!

For the purposes of this post, we'll use pseudocode and extended regular
expressions (`grep -e` or `egrep`).

# A source of frustration

Sometime in your life, you've probably seen a code review that looks something
like this:

```
# Capture the friendly text only.
matches = match("\[(.*?)\]\(https?://[a-zA-Z./?=-_]+\)")
friendly = matches[1]
```

When you see code like this, either you wrote it, or you have no idea what it
does (inclusive). Even if you know how it works today, by the time you get back
from your meeting after lunch, you will probably have forgotten and need to
reexamine the expression. And if that's how you feel as the regex wizard, how do
you think your team feels?

# How can we do better?

Lucky for us, regular expressions are already pretty well divided. A good rule
of thumb is to document each atom individually. For example:

```
# Capture the friendly text only.
matches = match(
    "\["                # The opening brace on the friendly text
    "(.*?)"             # The contents of the friendly text
    "\]"                # The closing brace on the friendly text
    "\(http"            # The opening brace and beginning of URL
    "s?"                # HTTPS optional
    "://"               # Continue the URL
    "[a-zA-Z./?=-_]+"   # The variable part of the URL
    "\)"                # The closing brace of the target URL
)
friendly = matches[1]
```

This is better. At least your reviewers can see what you're trying to do. But
why does your friendly text need to end in `?`? Are you only highlighting links
which are questions? Is this some heavily-indirected FAQ?

You may be thinking, "I know regex! I know that means I'm doing a lazy match!
Come on!" But who among us hasn't gone back for a quick Google when we
accidentally match the entire HTML page while just trying to match a single tag?
(Side note:
[do not use regex to parse HTML](https://stackoverflow.com/a/1732454).)

So while documenting each atom is a good start, it's a good idea to document any
particularly dark magic you may invoke, including (but not limited to):

* Character class shorthands (`\s`)
* Capture groups (`foo(.*)`)
* Backreferences (`(\w+) \1ovich`)
* Lookaround (`(?=[a-f0-9]+).{8}`)
* Greedy, lazy, possessive quantifiers (`<\w+?>`)
* Literal characters which usually have special meaning (`\[[a-z]+\]`)
* Bounded repeat quantifiers (`\w{8,36}`)

Let's try again.

```
# Capture the friendly text only.
matches = match(
    "\["                # The opening brace on the friendly text (literal [)
    "(.*?)"             # The contents of the friendly text - capture group 1,
                            # lazy match to avoid capturing the next link too
    "\]"                # The closing brace on the friendly text (literal ])
    "\(http"            # The opening brace and beginning of URL (literal ()
    "s?"                # HTTPS optional
    "://"               # Continue the URL
    "[a-zA-Z./?=-_]+"   # The variable part of the URL (alpha plus URL symbols)
    "\)"                # The closing brace of the target URL (literal ))
)
# Friendly text in capture group 1
friendly = matches[1]
```

That's much nicer, right? And hey, your senior colleague just came by your desk
to mention that for the last decade she has been using `([^)]*)` because she
didn't know about lazy matching, and she learned something new today. Even
seasoned developers don't know every way there is to use regex, and commenting
your expressions explicitly and verbosely will teach your team members things
they had never even thought to look up.

We're almost there. Finally, you want your coworkers to have some idea of what
sort of string they should write in order to match your expression. An example
string can play double duty as sort of section markers for your expression,
showing which part of the target string is matched by which part of the regex.

```
# Capture the friendly text from Markup hyperlinks.
# [Friendly text](https://target-website.com)
matches = match(
    # [Friendly text]
    "\["                # The opening brace on the friendly text (literal [)
    "(.*?)"             # The contents of the friendly text - capture group 1,
                            # lazy match to avoid capturing the next link too
    "\]"                # The closing brace on the friendly text (literal ])
    # (https://target-website.com)
    "\(http"            # The opening brace and beginning of URL (literal ()
    "s?"                # HTTPS optional
    "://"               # Continue the URL
    "[a-zA-Z./?=-_]+"   # The variable part of the URL (alpha plus URL symbols)
    "\)"                # The closing brace of the target URL (literal ))
)
# Friendly text in capture group 1
friendly = matches[1]
```

This is awesome! Let's go over some highlights.

* Your coworkers can point out easier ways to do something that you've done in
  your expression (e.g. using a shorthand character class for your URL)
* Your coworkers can learn ways to use regex that they didn't know about before
  (e.g. lazy matching)
* Your coworkers can tell easily if your regex makes a bad assumption (e.g. are
  you sure you got the
  [bracket order right](https://twitter.com/gregmcintyre/status/1123720996404584449)?)
* If someone needs to update part of the regex, they know exactly where to look.
* If someone doesn't understand why their string isn't matching, they know
  pretty much what their string should look like.

Never let anybody tell you that regices are write-only!
