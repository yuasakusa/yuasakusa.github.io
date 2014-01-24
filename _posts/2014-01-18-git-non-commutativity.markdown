---
layout: post
title:  "A pitfall of reordering commits by git rebase -i"
date:   2014-01-18 07:00:00
categories: git
---

### Summary

Some people may expect the following “commutativity law for commits”
in Git: if you change the order of commits by using `git rebase -i`
and Git does not report conflicts,
the content of the files should be the same before and after rebase.
However, this expectation is false.
To avoid surprise, always check the result of rebase,
even when Git does not report conflicts.

### Counterexample

Make a new Git repository.  Create the following file:

    A
    B

and commit it.  Remove the first line so that the file now looks like:

    B

and commit it.  Add back the first line and add another line at
the end, so that the file now looks like:

    A
    B
    C

and commit it.

Now run `git rebase -i HEAD~2` and swap the two “pick” lines.
Git happily reports:

    Successfully rebased and updated refs/heads/master.

Open the file.  The content is now:

    B
    C

Where did the first line go?

### Why did this happen?

After reordering the commits, we ended up with a different file
content, even though Git did not report any conflicts.
Why did this happen?

In the interactive rebase, we told Git to start from the initial
content, pick the second edit on it, and pick the first edit on
it.  What was the result of the first pick?  Because the second
edit added line A at the beginning and line C at the end, we may
expect that its result would be:

    A
    A
    B
    C

However, the actual result of the first pick is:

    $ git show HEAD^:test.txt
    A
    B
    C

This is where the expectation failed.

After the first pick, the second pick rightly removed line A at
the beginning, and the end result was that line A was missing
after swapping the order of the two commits.

Why was line A not added in the first pick?

The pick command in interactive rebase uses three-way merge, so
we should understand what three-way merge is.

The input of three-way merge is one base file and two derived
files.  The purpose of three-way merge is to create the file
which contains both the modifications in the first derived file
and the modifications in the second derived file.  However, the
operation does not require that the derived files were actually
the results of modifying the base file, or the derived files were
created after the base file chronologically.  How three-way merge
works is not very important here.

Back to the pick command in interactive rebase.  If we have file
F0, and we tell Git to pick a commit which modifies the file from
F1 to F2, the pick command performs three-way merge with F1 as
base file and F0 and F2 as derived files.  As explained above, it
does not matter to three-way merge whether the derived files were
really made based on the base file.

In the first pick in the steps shown in the previous section, it
means that Git performs three-way merge where the base file is

    B

and the two derived files are

    A
    B

and

    A
    B
    C

In the three-way merge, the modifications performed in these
derived files are recognized as follows.

* The first derived file adds A at the beginning.
* The second derived file adds A at the beginning and adds C at the end.

Line C causes no problem.  It is added in one of the two derived
files, so the result should contain this line.  However, line A
is problematic.  Both derived files contain this line.  Should
three-way merge add this line once?  Or twice?  Or should it
report conflict?

Three-way merge used in the pick command in interactive rebase
resolves the same modification applied in both derived files by
applying it only once.  So the result of the three-way merge in
this case becomes:

    A
    B
    C

In many cases, this decision of three-way merge is desirable.
When we merge a topic branch to the main branch and the main
branch already contains some modifications done in the topic
branch, it is great that this is not reported as a conflict.
However, in the scenario in question, this behavior of three-way
merge was the cause of the unexpected behavior.

### Is this a bug of Git?

I do not know whether this should be considered
to be a bug of Git or not.
I do not think that Git has ever promised that reordering commits
with `git rebase -i` preserves the file contents,
so this may well be working as designed,
just counter to the expectation some people may have.

For the record, I am currently using
Git for Windows 1.8.5.2 (1.8.5.2.msysgit.0;
the latest release version of Git for Windows as of writing).

It is interesting to compare this issue with a different kind
of failure of expectation in Git
found by [Zooko Wilcox-O’Hearn][badmerge]
which [Russell O’Connor][nonassoc]
summarized as the failure of the associativity law for merges.
As I understand it, this is a fundamental limitation of a merge
method which uses a base file and two derived files as the input
instead of a base file and two histories.

Our case, the failure of the commutativity law of commits,
seems less fundamental.
As far as I can see, it is all about how three-way merge handles
the case where the same modification has been applied in both the
derived files.
It seems to me that if three-way merge treats this case as a conflict,
the commutativity of commits may indeed hold,
although I have not seen a proof.
If the developers of Git fix the issue by this route,
probably the same-modification case should not be treated as a conflict
during the usual `git merge`
but only during `git rebase -i` and some other selected operations.

### Conclusion

Even when `git rebase -i` does not report conflicts,
we should verify that Git did what we meant.
When we change the order of commits, we should at least verify
that the files are the same before and after the rebase by
`git diff ORIG_HEAD`.
This recommendation is probably not new to many people
(at least I was verifying the result of `git rebase -i`
even before I found the example presented here),
but the example of non-commutativity of commits presented here
gives a yet another concrete reason why such verification is
necessary.

[badmerge]: https://tahoe-lafs.org/~zooko/badmerge/simple.html
[nonassoc]: http://r6.ca/blog/20110416T204742Z.html
