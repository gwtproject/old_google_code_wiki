# Introduction

When we go to ready a release candidate, we create a branch under /releases for the purpose of bugfixes related to that release cycle while new development continues in the trunk.  As an example:

http://google-web-toolkit.googlecode.com/svn/releases/1.5/

# Creating a release branch

The command that created the 1.5 release branch was:

```
svn copy https://google-web-toolkit.googlecode.com/svn/trunk@2939 https://google-web-toolkit.googlecode.com/svn/releases/1.5
```

I recommend recording your own svn copy command into the log message for the commit, as well as a textual description.  Here is the commit message for the above copy:

```
Creating 1.5 release branch.

svn copy https://google-web-toolkit.googlecode.com/svn/trunk@2939 https://google-web-toolkit.googlecode.com/svn/releases/1.5
```

Bugfixes for this release now continue in this branch.

# Tagging a release

Whenever we actually ship bits, it's important to "tag" the exact revision we shipped.  This is what we did for 1.5 RC1.

```
svn copy https://google-web-toolkit.googlecode.com/svn/releases/1.5@2941 https://google-web-toolkit.googlecode.com/svn/tags/1.5.0/
```

Again, I'd recommend recording the into the commit log the exact svn command used and a textual description:

```
Tagging the 1.5.0 release (1.5 RC1).

svn copy https://google-web-toolkit.googlecode.com/svn/releases/1.5@2941 https://google-web-toolkit.googlecode.com/svn/tags/1.5.0/
```

# Creating a `branch-info.txt`

As soon as a release branch is created, it is very important to create a `branch-info.txt` file in the root of the release branch to track interactions between this branch and other branches, especially the trunk.  Please refer to this example of the log for the 1.5 release branch:

http://google-web-toolkit.googlecode.com/svn/releases/1.5/branch-info.txt

The "Copies" section is straightforward and records the initial creation and any tags that were made.  The last section is somewhat optional but makes it very easy to create release notes for subsequent release candidates.  It's the "Merges" section that merits detailed explanation.

# Merging a release branch up into the trunk

If you're lucky, all code intended for your release will be checked into the release branch.  If someone screws up and checks something into trunk that you need to get into the release, see the section below.  Under all circumstances, you should avoid making the same change to the release branch and the trunk; some merge tools can silently screw this up.

So how do you actually get changes out of the branch and into the trunk?  The answer is controlled, periodic merges.  I'll go through the steps for the first merge and for subsequent merges.

## The First Merge

You need three things to get started:
  1. The svn revision that created the release branch.
  1. A recent svn revision where the release branch built successfully.
  1. A clean, up-to-date working copy of the trunk (run `svn status` to make sure it's absolutely clean).  You probably also want to be sure the trunk build is not currently broken.

Once you have all those things, go into your trunk working copy and run the merge command.  Let's suppose for this example that your release branch (1.5) was created at r3000 (on Google Code) and just built successfully at r3050 (on Google Code).

```
svn merge -r3000:3050 https://google-web-toolkit.googlecode.com/svn/releases/1.5 .
```

This affects only your working copy.  It will probably take a while, but when it's done your working copy should have a set of changes from the release branch merged into trunk.  At this point, you'll want to run `svn status` to make sure no files are Conflicted "C".  If there are, you'll need to manually fix them, which might include getting advice from the owners of the conflicted file(s).

Since this is your first merge, you'll have also pulled in the `branch-info.txt` file you created.  But you don't actually want this file to be in trunk, so you'll want to revert it (you won't have to do this in subsequent merges).

You're ready to commit now, but before you do that, I recommend doing one last "svn up" to synchronize against the repository before you commit.  The commit will take a while, so you'll want to minimize your chances of rejection.  When you're ready to commit, you should provide a log message like this:

```
Merging releases/1.5@3000:3050 trunk.
- Optional note about what changes you're merging in

svn merge -r3000:3050 https://google-web-toolkit.googlecode.com/svn/releases/1.5 .
```

Hopefully the merge succeeded.  Take note of the svn revision of your commit (let's say it was r3052 (on Google Code)), because we now need to go update our release branch's `branch-info.txt`.  Update the empty "Merges" section to look like this:

```
Merges:
/releases/1.5/@3000:3050 was merged (r3052) into /trunk/
-> The next merge into trunk will be r3050:????
```

The first line is pretty self-explanatory.  The second line is a note that will help you do the next merge correctly.  Notice that this "next merge" number is identical to the revision you specified to the right of the colon in your merge command.  This is important for reasons I'll talk about in the next subsection.

## Subsequent Merges

Before I get into the details, you need to understand the exact semantics of the revision range you specify in the merge command.  The key insight is to realize that an svn revision number can mean two different things, depending on context.  In one context, it can mean the _repository state_ at a particular revision.  I'll use `r3050` below to mean the repository state at revision 3050 (on Google Code).  In another context, it can mean the _change that occurred_ at a particular revision.  I'll use `c3050` below to mean the change that occurred when 3050 was committed.  This can be illustrated as a fence with posts:

```
r3049    r3050    r3051
 ||-------||-------||
 || c3050 || c3051 ||
 ||-------||-------||
```

This is highly relevant to the merge command, because the range you specify translates to a specific set of deltas.  Merging in `-r3050:3051` pulls in only one delta, namely `c3051`.  It does **not** pull in `c3050`.  To put it another way, you could think of the first number as exclusive and the second as inclusive.  This explains why in the previous section we wrote "`-> The next merge into trunk will be r3050:????`".  In our first merge, the final delta we got was `c3050`; specifying a starting point of `r3050` in the second merge will ensure that the next revision we get will be `c3051`.  Nothing skipped, and nothing merged twice is exactly what we want.

So for our second merge, if our last successful release branch build was at r3100 (on Google Code), we would merge into a clean updated copy of the trunk as:

```
svn merge -r3050:3100 https://google-web-toolkit.googlecode.com/svn/releases/1.5 .
```

Follow all the same instructions as above.  Record your merge into `branch-info.txt` and update the "next merge" line:

```
Merges:
/releases/1.5/@3000:3050 was merged (r3052) into /trunk/
/releases/1.5/@3050:3100 was merged (r3108) into /trunk/
-> The next merge into trunk will be r3100:????
```

# Someone screwed up and checked something into trunk which should have gone into the release branch

This is annoying, but not a reason for panic.  You need to merge the change from trunk into branch, but you also need to avoid merging the same change back into the trunk later.  There are several ways to handle this, here's a simple one:

  * Merge the trunk change into the branch.
  * Update the `branch-info.txt` to avoid merging that change.

Here's an example.  Assume we're in the state described above, and that someone committed the change we need into the trunk at r3110 (on Google Code).

We merge the trunk change into the branch:

```
svn merge -c3110 https://google-web-toolkit.googlecode.com/svn/trunk .
```

The `-c3110` is shorthand for `-r3109:3110`.  Let's say we commit the merge as r3115 (on Google Code).  We then update `branch-info.txt` to reflect the merge and the need to skip:

```
Merges:
/releases/1.5/@3000:3050 was merged (r3052) into /trunk/
/releases/1.5/@3050:3100 was merged (r3108) into /trunk/
/trunk@r3109:3110 was merged (r3115) into this branch
-> The next merge into trunk will be r3100:r3115 followed by r3115:????, skipping c3115
```

We skip c3115 because that's the change that's already in the trunk and we don't want to add it a second time.

# Philosophy

The procedures above are admittedly somewhat exacting.  However, the intent is to safeguard the integrity of the code base and prevent you from getting into a bad merge state, as well as provide an audit trail that can easily be debugged later if something has gone awry.  Merge problems can be extremely subtle and serious, and can be time consuming to detect, track down, and fix.  Some worst-case scenarios include unwittingly never merging a feature or fix into the trunk (causing a regression), merge errors not detected by the merge tools, and having to manually inspect every single change that occurred during a particularly bad merge.  The best course of action is to avoid them as much as possible by staying on the rails and recording your actions.