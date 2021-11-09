---
author: Ethan Rowe
title: Better Git it in Your Soul
github_issue_number: 26
tags:
- git
date: 2007-12-24
---

*The article title "Better Git it in Your Soul" refers to the raucous, foot-stomping first track on the classic album [Mingus Ah Um](http://en.wikipedia.org/wiki/Mingus_Ah_Um) by Charles Mingus. The title is appropriate for the subject of version control with Git, as distributed version control as envisioned through Git represents a paradigm shift that must be embraced and understood in a fundamental way in order to shine.*

The free software ecosystem abounds with good choices for version control software. Of particular interest is the emergence of distributed version control systems, which offer a fundamentally different approach to change management than do the popular [Subversion](http://subversion.apache.org/) and its venerable predecessor, [CVS](http://ximbiot.com/cvs/wiki/).

Choice is a wonderful thing, yet it brings a near-inevitable wringing of hands in its wake: how does an engineering team/company/guru choose between so many options? How can you be sure of choosing the right one? Perhaps you only recently moved from CVS to Subversion; does the prospect of moving **again** provoke groans, and force consideration of the choice between what is good and what is easy?

At End Point, we only relatively recently began using Subversion when, for a variety of reasons, we looked into the choices more deeply. It became clear that, for all of Subversion's improvements over CVS, it was not the choice for the future. After careful consideration of the offerings out there (carried out primarily by our CTO, [Jon Jensen](/team/jon-jensen)), we determined that distributed systems were the best choice, and that [Git](http://git-scm.com/) is the winner in that category.

Git is not the only choice for distributed version control, by any means. [Mercurial](http://mercurial.selenic.com/wiki/) (abbreviated, wisely, as "Hg") offers a similar feature set and approach; the two are really quite close to each other. We determined that Git is simply the more mature project. There are other options still. However, rather than consider them all here, we'll focus instead on what we like about Git and the benefits it offers over Subversion.

### Distributed Development

Distributed development is at the heart of Git. In 2005, licensing concerns surrounding the use of [BitKeeper](http://www.bitkeeper.com/) forced [Linus Torvalds](http://en.wikipedia.org/wiki/Linus_Torvalds) to move Linux kernel development to a new version control system. Git is the result, started by Torvalds himself and later growing beyond his and the kernel development community's immediate needs into a full-fledged project in its own right. The Linux kernel development project, having proceeded in a distributed fashion already, needed a system that would continue this path, allowing for easy branching and merging across local and remote sources. Git's fundamental design reflects this, and the result is an incredibly fast, flexible, powerful system that opens up a staggering set of workflow options quite beyond the possibilities of any centralized version control system (such as Subversion or CVS).

#### How Distributed Development is Achieved

*This section gets a bit technical; if your interest in this subject is more at the level of workflow, management, etc., you can safely skip down to the "Distributed Development in Practice" section.*

Git "tracks content, not files". All files and directories managed in Git are reduced to basic Git "objects", the identity of each being determined by a SHA1 hash of its content. A file "X" added to Git, therefore, is not stored as "file X" within the Git repository; the file itself is stored as a "file" object identified by a SHA1 hash of the file "X"'s contents along with some Git headers about the data (identifying the Git object as a "file" object, including the file permissions, etc.). The name "X" is not found anywhere within this blob that represents "X". However, the Git object representing the directory in which file "X" lives will contain a reference to the "X" blob (via the SHA1 hash ID) and the name "X" for that reference. This is rather like Unix filesystems: a file on disk is just a file on disk, with no outer-world identity other than the local names given to references to that file on disk from directories that choose to reference it. The names are "local" because they are only meaningful within their respective directory.

Consider what this means: when file X in directory Y is changed, Git does not represent the event as "file Y/X changed thus: ...". Instead, a Git object representing the new state of file X is added to the Git repository (unless this is a state we've already seen for file X, in which case no new object is necessary thanks to the ID-by-content scheme described above). Additionally, a Git object representing the new state of directory Y is added, in which directory Y now references the new object created for X; this means that a change in file Y/X necessitates a change in Y. A change in Y would necessitate a change in its parent directory, to reference the new version of Y, and so on, all the way up to the top of the repository.

The end result is a great set of trees. Any change to the repository contents at any level results in a change to the entire tree. A "commit" in Git, then, is (at its simplest) an object that refers to the top-most tree object (by its SHA1 ID) as it was at the time of the commit. A commit object also may reference 0 to *n* parent commit objects, which when traced backwards reveal the commit history of the overall project. For workflow purposes, Git requires each commit object and its parent references in order to provide historical consistency. However, any given commit is a full snapshot of the project, and the previous commits are **not** needed in order to determine the actual contents of the repository at a given time. The commit object is, per usual, identified by the SHA1 hash of its contents, meaning that the name of the commit object is dependent upon its parent references, the state of the tree referenced by the commit, and some other details (the commit message, the timestamp, etc.).

Git's references bring this all together to enable distributed development. Refs may be simple, local refs, like the traditional *HEAD* reference that points to the "current" revision. Refs may also be remote, pointing to a branch within an entirely different repository that may be on a different machine. While the "commit" command is always local to the repository, commits can be pushed and pulled to and from remote references. Because objects are always named via SHA1 hashes of their contents, determining merge paths and the like between remote repositories is not difficult; either the Git repository has a certain Git object or it doesn't. Determining what changed in a given commit is a matter of comparing the tree object of that revision with that of the previous commit and finding objects that appear in one tree but not the other, or with different reference names. Branches in Git are merely references to a line of commits, and merges can be represented in Git as a commit with multiple parent commits (one commit per merged branch).

The simple genius of the design is quite a delight to behold, if one's leanings are sufficiently nerdy (as is apparently the case for the author). The internals are presented visually to great effect in the excellent article "[Git for Computer Scientists](http://eagain.net/articles/git-for-computer-scientists/)".

#### Distributed Development in Practice

Fear not! For users new to Git, the workflow can closely match that of Subversion or CVS, with the primary difference being that "working copies" are replaced with clone repositories (and their own working copies). A single central Git repository can act as the "master" repository, the canonical representation of your project and its history. Individual users clone that repository (via *git clone*), meaning that each user gets a new, standalone Git repository containing a remote reference to the central repository. Each user's commits are local to the user's clone repository until the user pushes changes out to the central repository (via *git push*), which is the point at which some conflict resolution comes into play and the user is forced to get the latest updates from the master repository (a "pull" via *git pull*). Therefore, a commit could be considered a two-stage process: commit, then push. While it isn't necessary to push immediately after a commit (you can go as long as you want without pushing/pulling), this is rather akin to accumulating large sets of changes in a Subversion working copy without doing an update and commit; with each unpushed change, you're increasing the pain of the eventual conflict resolution.

A Subversion or CVS user at this point would wonder why this is beneficial, given that all we've done is introduce more steps into the regular workflow. However, you do not need to be committing to multiple remote repositories in order to see distributed development in action. A clone repository is a full repository in is own right. You can pursue branched development within that repository. Branching is cheap in Git, so it's easy to use branches for different lines of development. One branch could be used for some longer-running work that you might not expect to push out for a few days, while another branch could be used for quick fixes and adjustments that get pushed out quickly. Experimental work can take place in yet another branch. And so on.

This hopefully illustrates some of the benefits of the Git model. However, the opportunities for collaboration are vastly expanded. Consider: in the above example, there's nothing stopping you from cloning one of your "working copy" repositories, rather than cloning the central master repository. For larger teams of engineers working on many concurrent projects, this aspect of Git can provide huge advantages. However, upon getting comfortable with Git, smaller teams (even just a pair of engineers) can use this method of collaboration for projects of any size, accumulating a set of changes in one repository without touching the "master" repository at all, then pushing the changes out to the master once everything is in place. There is no need to create and manage branches for this kind of work within the master repository; the master can remain entirely clean from such things.

Linus Torvalds illustrates some of the differences inherent in Git's approach within [this email](http://lwn.net/Articles/246381/) discussing practices in Linux kernel development. As he points out, Git places a greater focus on *social* organization for control and project management; for many projects, particularly smaller ones, a single master repository may suffice, but for larger projects, there may be several centralized, specialized repositories, all of which providing source material for a single public-facing canonical repository. Such is the case of kernel development, in which networking development works with one repository while cryptography works with a different one, and Torvalds himself responsible for merging changes from both into his master repository. The point is: the version control system can be applied in any fashion appropriate to the social needs of the project, and is infinitely more flexible than a centralized version control system as a result. Git grows and changes with your project; Subversion or CVS force a particular workflow and give you few options otherwise.

### Other Advantages of Git

Git is blazingly fast compared with Subversion for large sets of files. Let's be honest: Subversion is painfully slow with even modestly-sized directories. Git is not. (Though it still may bog down on big sets of large binaries.)

Despite using a snapshot model for representing data, Git is quite efficient in its use of disk space. When running in a Unix environment, Git will try to use hard links whereever it can to minimize space consumed. The snapshot model in Git is also normalized to a significant degree, though it does fall short of perfect normalization (perfect normalization would require abstracting out the Git object data from the Git object headers); while Git creates a lot of references within commits and trees, the big chunks of data that come from userland files are minimized to a reasonable extent (all Git objects are zlib-compressed to save space).

Branching and merging are a fundamental aspect of Git's design; they **have** to be in order to support distributed development. Branching in CVS is a painful exercise; while CVS advanced the state-of-the-art in its own time, CVS' design was simply not up to the task of real branching and merging. While claims to have solved the branching problem have been made in reference to Subversion, such claims are simply false: the branching tools may be improved in comparison with CVS, but Subversion still operates in the same revision modeling paradigm as CVS and is therefore fundamentally ill-suited for the task. That paradigm is broken when branching and merging are considered. Branching and merging remain an *exceptional* activity. In Git, branching is an *everyday* activity. To refer to this difference as a paradigm shift is not an exaggeration.

Any one of these advantages taken alone may not seem all that exciting, particularly if you don't actually use version control software, or if you work on projects of small scope, minimal complexity, and little collaboration. However, they all matter. Performance matters: on a project involving multiple engineers over the course of several months, full checkouts can be a regular occurence, and the speed of those checkouts affects productivity (not to mention job satisfaction). Branching and merging may be thought of as things that "this project won't need", but such thinking simply reflects the old paradigm in which branching is a specialized activity to be reserved for subprojects of significant scope/complexity or for marking milestones within a given project. Upon consideration of how human beings work and collaborate, and the trial-and-error nature of much of software engineering, it becomes obvious that branching and merging can be an invaluable tool in one's toolkit.

### In Conclusion

Subversion offers some real, hard-won improvements over CVS. Many people have worked very hard over many years to achieve this. This contribution to the free software ecosystem should not be overlooked or denigrated. That said, there is simply no technological reason to use Subversion for new projects, period.

Git offers a better way forward. Guides such as [Git for SVN Users](http://git-scm.com/course/svn.html) can get Subversion users started gracefully. Even if you and your team don't immediately use the full power of Git, chances are high that you *will* use it over time as people gain familiarity with the tools and the project evolves. The power Git offers is so fundamental to the Git design itself that it is practically impossible to not use it upon working with Git for any significant length of time. In any case, any meaningful thing you might do in Subversion or CVS can be done in Git. The reverse cannot be said: Git is far, far more flexible than any centralized system.

When first embarking with Git, it is critical to recognize these fundamental differences in design and ability. While Git can be productively used in a workflow similar to that of centralized version control systems, its scope goes far beyond and the basic design of Git really needs to be absorbed and understood by involved engineers in order to add the most value.

Better use Git. Better Git it in your soul.