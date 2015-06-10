## Status Values
The status values indicate the progress of the Issue toward being implemented.
The status does not imply anything about the importance or eventual fate of the issue. The
meanings of the key status values are:

  * New - untriaged.
  * Accepted - acknowledged as a something the core GWT team should address.
  * PatchesWelcome - acknowledged as a desirable change, but one which the core GWT team is unable to address.
  * Started - the assigned engineer has begun working on the Issue.
  * ReviewPending - a patch for the issue is complete, and pending peer-review; fix is not yet committed.
  * FixedNotReleased - coding on the Issue is complete and committed, but the revision containing the fix has not yet been put into a packaged release. Milestone is set indicating the release expected to include the fix.
  * NeedsInfo - the Issue requires additional information from the reporter or a commenter. No further action will be taken until the additional info is provided.
  * Fixed - fixed, and available in the packaged release indicated in Milestone.
  * NotPlanned - the Issue will receive no further attention.
  * Invalid - bug is erroneous, incomplete, ambiguous, etc.
  * Duplicate - bug is a duplicate of an existing bug.
  * CannotReproduce - The issue could not be reproduced at all.

The other status values are self-explanatory.  Again, note: a bug being marked
as Started, FixedNotReleased, etc. does not imply it will be in the next
build.  Only the Milestone indicates information pertaining to releases.

## Types
This field is used to denote whether an Issue is a Bug or a Request for
Enhancement (read: Feature).  It can also denote non-code bugs (such as
documentation issues.)

## Priorities
The Priority field is a measure of how important (and therefore likely) it is
that an Issue be resolved in the next release.  It is not a measure
of the severity of an Issue.  For example, most Compiler bugs will be marked
as Critical since a strong effort is generally made to make sure such Issues
do not carry over to the next release. However, a hosted mode crash bug is
high severity but might very well be Medium or Low priority if its occurrence
is rare, thus allowing project leadership to defer the bug to a future
release.

In a nutshell: Priority indicates whether the Issue should be a blocker for
the next release, not the relative importance or severity of a bug. Not every
issue has a priority, only those that need to be called out.

| **Priority** | **Meaning** |
|:-------------|:------------|
| Critical     | The issue will block the release;  that is, the corresponding milestone will NOT ship until the issues is resolved |
| High         | Very strong preference to resolve in the indicated release; will most likely block a release unless already late in a cycle |


## Milestones
Milestones correspond directly to packaged releases, with an optional suffix denoting pre-release builds. The digits in a release number are separated with underscores, and the suffix with a dash. This allows searches for all issues fixed in any milestone leading to a particular release (e.g. by searching for Milestone:2\_2), or for particular prerelease (e.g. Milestone:2\_2-M1).

Issues are attached to a Milestone if and only if they were (or will be)
present in the packaged binary corresponding to that Milestone.  Put another
way, if an Issue does not have a Milestone set, it will not ship in any
currently-planned release.  Conversely, users can use the Issue Tracker UI to
search for a Milestone they are interested in, and be confident that the
results of that search are an accurate description of the contents of the
corresponding release (remembering that "Priority" is an indication of the
likelihood of its successfull inclusion).