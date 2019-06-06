---
layout: post
title: "A Tour of Git's Object Types"
date: 2019-06-06
categories: git open-source version-control scm
---

Naming is one of the hard problems in computer science, right? It's hard for
Git developers too. One of the more arcane concepts in Git - object
reachability - becomes simpler to understand with a little bit of naming
indirection.

Reachability is an important concept in Git. It's how the server determines
what objects you need in order to get it up to what the server itself knows.
It's also how merges and rebases work. With all this big stuff riding on
reachability, it seems intimidating to try to understand - but it turns out if
we give it a slightly simpler name, things become a little clearer.

## Git's four object types

Under the covers, Git is mostly a directed graph of objects. Those objects come
in four flavors; from root to leaf (generally), those flavors are:

- Tag
- Commit
- Tree
- Blob

We'll take a closer look in the opposite order, though.

# Blob

Surprise! It's a file. Well, kind of - it can also be a symlink to a file - but
this is the most atomic type of object. We'll explore these a little more later,
but really, it's just a file.

# Tree

A tree references zero or more trees or blobs. Said another way, a tree holds
one or more trees or files. This sounds familiar - basically, a tree is a
directory. (Okay, or a symlink to a directory.) It points to more trees
(subdirectories) or blobs (files). It can also point to commits in the case
of submodules, but that's another conversation.

By the way, "tree" is one that gets a little sticky, because we also talk about
"working tree" as well as "worktree". We'll touch back on that in a minute.

# Commit

This is the one we're all familiar with - commits are those things we write at
1am, angry at a pesky bug, and label with something like "really fix it this
time", right?

A commit references exactly one tree. That's the root directory of your project.
It also references zero or more other commits - and this is where we diverge
from the filesystem parallel, because the other commits it references are its
parent(s), each of which has its own copy of the project at that commit's point
in time. (Commits with more than one parent are merge commits; otherwise, your
commit only has the one parent.)

Commits represent a specific state of the repository which someone thought was
worth saving - a new feature, or a small step of progress which you don't want
to lose as you're hacking your way through that homework assignment. Each commit
points to a complete image of the repository - but that's not as bad as it
sounds, as we'll see when we try it out.

# Tag

Tags are a little lackluster after all the exciting types up until now. They are
essentially a label; they serve as an entry point into the graph and point to
either another tag or a commit. They're generally used to mark releases, and you
can see them pretty easily with `git log --oneline --decorate=full`.

# A quick return to an overloaded word

"Tree", "worktree", and "working tree" seem to refer to different concepts. A
tree is a folder. Your working tree is your project state (and we can talk about
having a "clean" working tree, which means you don't have any staged or unstaged
changes pending). And "worktree" is a way for you to work on multiple branches
simultaneously in a different directory in a safe way (read `git help worktree`
for more - worktrees are awesome). But they're all named tree!

It's a little clearer now that we know that every commit points to one tree -
the root of the project, a.k.a. your working tree, a.k.a. your worktree. `git
worktree` lets you put the tree associated with the commit at the tip of your
current branch in a different directory than the one you cloned into, and
having a clean working tree means that your filesystem is the same as the tree
your `HEAD` points to.

## Try It And See<sup>TM</sup>

It turns out the details of what objects Git knows about and what those objects
contain isn't as opaque as we might think. Git exposes a number of "plumbing
commands" which aren't so handy for interactive use but which are very useful
for scripting, as they describe the state of the repository in a concise and
predictable way. Let's walk through creating a pretty basic repository and
examining it with some low-level plumbing commands!

# An empty repo

For starters, we'll make a new, shiny, totally empty repo.

{% highlight shell_session %}
$ mkdir demo
$ cd demo
$ git init
{% endhighlight %}

We've got nothing. If we try `git log`, we'll be assured that we have no
commits, and if we try `git branch -a` we'll see we have no branches, either.
So let's make a very simple first commit.

# A single commit

{% highlight shell_session %}
$ echo "abcd" >foo.txt
$ git add foo.txt
$ git commit -m "first commit"
{% endhighlight %}

I know this is boring, but bear with me and run `git ls-tree HEAD`.

Hey, look, a blob! You'll see the object mode, the type, the _object ID_, and
the name of the file. For the rest of the post, I'll refer to the object ID as
the OID.

The OID is a hash of the file contents. You can verify this for yourself with
`git hash-object foo.txt` - it's the same as your new blob's OID. The new blob
is literally just your file foo.txt, which you can verify by running `git
cat-file -p <oid>`:

{% highlight shell_session %}
$ git ls-tree HEAD
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
$ git cat-file -p acbe86c7c89586e0912a0a851bacf309c595c308
abcd
{% endhighlight %}

While we're here, we can also take a look at the commit object. Use `git log`
to determine your commit's OID, then use `git cat-file -p` to print the
contents:

{% highlight shell_session %}
$ git log
commit a491754d3256b5823607d7ea4afb835c07f9fc2c (HEAD -> master)
Author: Emily Shaffer <nasamuffin@gmail.com>
Date:   Wed Jun 5 22:56:25 2019 -0700

    first commit
$ git cat-file -p a491754d3256b5823607d7ea4afb835c07f9fc2c
tree 7376230624de9a38f4ea6ca4c41d47e41304b1bd
author Emily Shaffer <nasamuffin@gmail.com> 1559800585 -0700
committer Emily Shaffer <nasamuffin@gmail.com> 1559800585 -0700

first commit
{% endhighlight %}

This gave us the OID of our root tree, which we can also examine:

{% highlight shell_session %}
emily@xenaTWP:~/demo$ git cat-file -p 7376230624de9a38f4ea6ca4c41d47e41304b1bd
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
{% endhighlight %}

And what do you know - it's precisely the same output as `git ls-tree HEAD`.
Because we are literally printing the tree pointed to by HEAD.

# A new file

Let's see what happens when we add another file.

{% highlight shell_session %}
$ echo "efgh" >bar.txt
$ git add bar.txt
$ git commit -m "The noise you make after too much time at the bar."
[master acc255b] The noise you make after too much time at the bar.
 1 file changed, 1 insertion(+)
 create mode 100644 bar.txt
{% endhighlight %}

Now we'll take a look at `git ls-tree HEAD` again and compare it to the output
from the prior commit (if you've scrolled past, you can run `git ls-tree
HEAD^`).

{% highlight shell_session %}
$ git ls-tree HEAD^
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
$ git ls-tree HEAD
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
{% endhighlight %}

It looks like we didn't actually create a new blob for foo.txt. That's why this
concept of each commit containing a copy of every file in the repository is
actually okay - the only new objects of substance being created are new copies
of whatever thing you've changed. (This is also why it's historically a bad idea
to check in your compiled binaries - someone doing `git clone` with no arguments
will get not just your latest release binary, but every release binary you ever
checked in. Oof.)

But wait - if we shouldn't check in our 50MB release build, why is it okay for
us to check in our 5000-line legacy monolithic class? (Don't be embarrassed. It
happens to all of us.) It turns out that I'm not being totally honest when I say
we store "a copy of every file". All the objects are stored in `.git/objects/`,
so we can have a look with `cat
.git/objects/ac/be86c7c89586e0912a0a851bacf309c595c308`. Breathe a sigh of
relief; blobs are stored in a compressed state and `git cat-file` unpacks it for
us. The issue here is that your binary is much more difficult to compress than
a text file.

# A modified file

So what happens when we modify a file?

{% highlight shell_session %}
$ echo "aaaa" >foo.txt
$ git add foo.txt
$ git commit -m "the scream of despair"
$ git ls-tree HEAD
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	foo.txt
{% endhighlight %}

Looks like our bar.txt object remains, but we've got a new OID for foo.txt. So
what exactly lives in the blob of a file modification?

{% highlight shell_session %}
$ git cat-file -p 5d308e1d060b0c387d452cf4747f89ecb9935851
aaaa
{% endhighlight %}

No diff. It's the whole file. And our old version isn't gone; we can still pull
out the OID we used to know about:

{% highlight shell_session %}
$ git cat-file -p acbe86c7c89586e0912a0a851bacf309c595c308
abcd
{% endhighlight %}

# A subdirectory

We mentioned earlier that trees can point to trees. Let's put it into practice:

{% highlight shell_session %}
$ mdkir baz
$ echo frotz >baz/zork.txt
$ git add baz
$ git commit -m "ran out of placeholder words"
{% endhighlight %}

So we expect to see a new tree and a new blob. A first crack at `git ls-tree
HEAD` doesn't go so well:

{% highlight shell_session %}
$ git ls-tree HEAD
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
040000 tree 22741c26d13c9539d0ab7476ce09074dbd62a977	baz
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	foo.txt
{% endhighlight %}

What happened to `zork.txt`? It's still there:

{% highlight shell_session %}
$ git ls-tree 22741c26d13c9539d0ab7476ce09074dbd62a977
100644 blob 8e4020bb5a8d8c873b25de15933e75cc0fc275df	zork.txt
{% endhighlight %}

By default, `git ls-tree` kind of behaves like `ls` - it doesn't recurse into
the trees it finds.  So we'll ask it to recurse (`-r`) and to also show the
names of tags it's recursing into (`-t`):

{% highlight shell_session %}
$ git ls-tree -rt HEAD
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
040000 tree 22741c26d13c9539d0ab7476ce09074dbd62a977	baz
100644 blob 8e4020bb5a8d8c873b25de15933e75cc0fc275df	baz/zork.txt
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	foo.txt
{% endhighlight %}

# Summary

In the end, we have four commits:

{% highlight shell_session %}
$ git log --oneline
230a4e9 (HEAD -> master) ran out of placeholder words
a0aa5e4 the scream of despair
acc255b The noise you make after too much time at the bar.
a491754 first commit
{% endhighlight %}

Each one has its own tree:

{% highlight shell_session %}
$ git ls-tree 230a4e9
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
040000 tree 22741c26d13c9539d0ab7476ce09074dbd62a977	baz
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	foo.txt
$ git ls-tree a0aa5e4
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	foo.txt
$ git ls-tree acc255b
100644 blob ac38e5aa3826c56e2f32df05d23d2c27f09e7782	bar.txt
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
$ git ls-tree a491754
100644 blob acbe86c7c89586e0912a0a851bacf309c595c308	foo.txt
{% endhighlight %}

And in total, we should have **4** blob objects (two versions of `foo.txt` and
one version each of `bar.txt` and `zork.txt`), referenced by **5** tree objects
(one tree per commit, plus one tree for `baz/`), which are tracked through time
by **4** commits, giving us a total of 13 objects.

We can check:

{% highlight shell_session %}
$ git rev-list --all --objects
230a4e968f007b822d780c70e2f145f5b46ed170
a0aa5e491bb62906dc746aa10c548566e5dc8e4c
acc255b3a1ba1a3471133ff3aa7c2630437dd38c
a491754d3256b5823607d7ea4afb835c07f9fc2c
f2e243fb4052f01287a1d2225a231f9131af6840 
ac38e5aa3826c56e2f32df05d23d2c27f09e7782 bar.txt
22741c26d13c9539d0ab7476ce09074dbd62a977 baz
8e4020bb5a8d8c873b25de15933e75cc0fc275df baz/zork.txt
5d308e1d060b0c387d452cf4747f89ecb9935851 foo.txt
5a684389c10291eb7a187d37c5fb5e6ae6e44b3a 
076c36567233c1e9c22becf6008a96af7f6e865d 
acbe86c7c89586e0912a0a851bacf309c595c308 foo.txt
7376230624de9a38f4ea6ca4c41d47e41304b1bd 
$ git rev-list --all --objects | wc -l
13
{% endhighlight %}

We can prove it to ourselves a little more easily with some pretty formatting:

{% highlight shell_session %}
$ git rev-list --all --objects --header
230a4e968f007b822d780c70e2f145f5b46ed170
tree f2e243fb4052f01287a1d2225a231f9131af6840
parent a0aa5e491bb62906dc746aa10c548566e5dc8e4c
author Emily Shaffer <nasamuffin@gmail.com> 1559803688 -0700
committer Emily Shaffer <nasamuffin@gmail.com> 1559803688 -0700

    ran out of placeholder words
a0aa5e491bb62906dc746aa10c548566e5dc8e4c
tree 5a684389c10291eb7a187d37c5fb5e6ae6e44b3a
parent acc255b3a1ba1a3471133ff3aa7c2630437dd38c
author Emily Shaffer <nasamuffin@gmail.com> 1559802687 -0700
committer Emily Shaffer <nasamuffin@gmail.com> 1559802687 -0700

    the scream of despair
acc255b3a1ba1a3471133ff3aa7c2630437dd38c
tree 076c36567233c1e9c22becf6008a96af7f6e865d
parent a491754d3256b5823607d7ea4afb835c07f9fc2c
author Emily Shaffer <nasamuffin@gmail.com> 1559801779 -0700
committer Emily Shaffer <nasamuffin@gmail.com> 1559801779 -0700

    The noise you make after too much time at the bar.
a491754d3256b5823607d7ea4afb835c07f9fc2c
tree 7376230624de9a38f4ea6ca4c41d47e41304b1bd
author Emily Shaffer <nasamuffin@gmail.com> 1559800585 -0700
committer Emily Shaffer <nasamuffin@gmail.com> 1559800585 -0700

    first commit
f2e243fb4052f01287a1d2225a231f9131af6840 
ac38e5aa3826c56e2f32df05d23d2c27f09e7782 bar.txt
22741c26d13c9539d0ab7476ce09074dbd62a977 baz
8e4020bb5a8d8c873b25de15933e75cc0fc275df baz/zork.txt
5d308e1d060b0c387d452cf4747f89ecb9935851 foo.txt
5a684389c10291eb7a187d37c5fb5e6ae6e44b3a 
076c36567233c1e9c22becf6008a96af7f6e865d 
acbe86c7c89586e0912a0a851bacf309c595c308 foo.txt
7376230624de9a38f4ea6ca4c41d47e41304b1bd 
{% endhighlight %}

## Back to reachability

We're usually worried about finding out which objects are accessible from which
other objects. It turns out that "reachability" can be described in a couple
ways. Most succinctly, we can say it means that an object can be reached from
another object during a revision walk. But we can also describe it by explaining
what reachability means for each type of object.

- Tag A is reachable from tag B if tag B can eventually be dereferenced to tag
A. Tags can be thought of as pointers, so if `*b==a` or `***b==a` or so on is
true, then tag A is reachable by tag B.

- Commit A is reachable from commit B if commit A is an ancestor of commit B.
For commits, "reachability" is synonymous with "ancestry".

- Tree A is reachable from tree B if it is a subdirectory of the directory tree
B is talking about. That is, if `find b-dir -name a-dir -type d` would succeed.

- Blobs don't point to other blobs. But blob A is reachable from tree B if blob
A is contained within the directory of tree B - that is, if `find b-dir -name
a-file -type f` would succeed.

- Tree A is reachable from commit B if it's commit B's root tree, or if it's a
subdirectory of that root tree.

- Commit A is reachable from tag B if tag B can eventually be dereferenced to a
tag which points directly to commit A, since tags can point to other tags or to
commits.

So while it means something a little different for each object, it's ultimately
trying to answer the question: "Somewhere in my history, did I know about this
object?"
