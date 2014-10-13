Customised Git Flow
===================

[github.com/expobrain/git-flow][expobrain-git-flow]


Introduction
------------

We are following an customised version of the [Git Flow][git-flow] branching
model paradigm to handle the development, bugfixing and deployment of your
repositories. The key concepts of our branching model are:

- the *dev* branch contains the most updated and unstable version of the source
  code
- the *staging* branch contains code related to completed features of bugfixes
- the *master* branch contains only stable code
- any feature or bugfix are implemented by a separated branch from *dev*
- any completed feature or bugfix is promoted to a *staging* branch
- any stable feature of bugfix is promoted to a *master* branch

Some limitation on the deploy task are in place for safety:

- deploy to dev environment is possible only from dev, feature and bugfix
  branches
- deploy to staging environment is possible only from *staging* branch
- deploy to production environment is possible only from *master* branch

There are other extra rules about branch management and deploy limitations but
they will be explained later in details.


Notes about commit’s comments
-----------------------------

Commit’s comments are really important, they are important to you to keep track
about what’s the scope of the commit and they are important also to other
members of the team on reading the repository’s history and relate a commit or
a set of commits to a particular feature or bugfix.

For this reason try to keep your comments clear and self-explanatory about
what’s the content of the commit. Also on merging back feature, bugfix or
hotfix branches put a reference of the related ticket into the commit’s
comment, some bug-tracking systems have standard format for ticket numbers (for
example [Codebase][codebase] uses `[touch: <ticket-no>]` to link a commit to a
ticket).

Putting the ticket number into a merge’s comment also allows us to temporarily
revert a feature which is not yet ready to be promoted to staging or
production, for instance a ticket which is part of a set of tickets that forms
a single feature only when completed together.


Branching from dev
==================

Creation of the branch
----------------------

Branching from *dev* branch must be performed every time we start a new feature
or bugfix. To start a new branch:

    $ git checkout dev  // ensure you are in branch dev before branching
    $ git checkout -b <my-branch>  // create and switch into a new branch

From now on all the work should be done in the relate branch keeping it
isolated from the parent branch. Because this is a feature/bugfix branch and
it’s implicitly not shared with other on the team you are safe to do any kind
of modification on the code base, even modifications which breaks the build,
without interfering with anyone else.


Branch maintenance
------------------

The lifespan of the branch can be very short or very long depending by the
complexity of the task. This mean the branch can diverge from the parent branch
because other members of the team committed their work on the parent branch,
possibly rendering your code base incompatible; also it’s dangerous to keep
your work (especially when spans one or more days) only on your machine,
introducing the possibility to lose the work done in case of hardware failure
or the machine get stolen.


Rebase
------

To maintain your code base update with the parent branch rebase your branch
periodically. You chose when apply a rebase by the state of your code, a good
rule of thumb is to rebase every time your code reach an acceptable level of
stability, or when new features or bugfix are merged into the parent branch;
you can even rebase just before resuming your work on the branch. This is up to
you but you must always rebase your work before merging back into the parent
branch (it will be explained in details later in this document).

To rebase against the parent branch be sure you had committed all the changes
and execute within your branch:

    $ git fetch
    $ git rebase origin/dev

The rebase process re-apply all the commits in the current branch on top of the
last commit on the parent branch. The rebase process can stop in case of a
conflict which can be resolved by:

    $ git mergetool  // cycle through every conflict to let you fix them one by one
    $ git rebase --continue  // continue the rebase process after conflict resolution

During conflict resolution you can abort the rebase by:

    $ git rebase --abort

which revert your branch to the state prior the rebase command.


Push to remote repo
-------------------

Another important habit to remember is to keep a copy of your long running
branch into the remote repo. To do that is really simple:

    $ git push --set-upstream origin <mybranch>

the first time you push your branch into the remote repo, and:

    $ git push

the subsequent times. If you had rebased your branch against the parent branch
you’ll need to force the push to the remote repo:

    $ git push --force

You can push your branch even before rebasing from the parent branch as an
extra backup of your work.


Merging back to the parent branch
---------------------------------

Once the feature or bugfix is complete and we are happy about the current state
of the code base we are ready to merge back to the parent branch.

Before of that you should rebase your commits on top of the tip of the parent
branch:

    $ git fetch  // fetch all the commits and refs from the remote repo
    $ git rebase origin/dev  // rebase on top of the remote dev

The first command fetch all the commits and references (tags and branches) from
the remote repository and updates your local copy, it’s needed to receive all
the modifications done by other team members in the *dev* branch. The second
command rebases your branch on top of the remote state of the *dev* branch,
that is the prefix origin/ reference the remote copy of *dev* instead of the
local one.

Once you code base is rebased and the feature or bugfix is tested again it’s
time to merge back into the parent branch:

    $ git checkout dev
    $ git pull  // be sure dev is up-to-date
    $ git merge --no-ff <my-branch>  // merge without fast-forward

By disabling the fast-forward options we are able to keep all the history
related to our branch outside the history of *dev* branch and to explicitly
mark in the *dev*’s history when the feature or bugfix was merged back.

The final step involves some cleanup of the repo:

    $ git branch -d <my-branch>  // delete the local copy of my-branch
    $ git push origin :<my-branch>  // delete the remote copy of my-branch

Where the first command is mandatory to remove your branch for your local copy
of the repository, the second command is necessary only if you had pushed your
branch into the remote repository otherwise you can skip it.


Staging branch
--------------

The *staging* branch is an intermediate branch to be used as a final test of a
set of feature or bugfixes before they are promoted to production branch.

The only modification allowed on this branch are bugfixes on features which are
under current testing; this limits the number of modifications introduced in
the branch and reduces at minmum the possibility of introducing new bugs.


Merging from dev
----------------

As stated before only features or bugfixes ready for production should be
merged from *dev* branch into *staging* branch. This means that before merging
we need to review the list of commits which will be merged into the branch:

    $ git checkout staging
    $ git pull  // ensure local copy is updated
    $ git log staging..origin/dev --oneline  // list all the commits in dev which are not in staging

The last command will generate a list of commits which will be merged into
*staging* branch; here you can see how much is important to write meaningful
comments into the commits, the comments help us to understand what will be
merged and what possibly we want to exclude from the branch.

Once ready we merge the *dev* branch into *staging* in the same way as we do
with feature or bugfix branches by merging without fast-forward:

    $ git merge --no-ff origin/dev


Bugfixing
---------

During the QA process modification of the code can arise because of bugs or
detail’s polishing; these modification should be done only on the *staging*
branch by following the same rules explained above about the feature or bugfix
branch lifespan.

Once the bugfixing and the polishing is done we deploy again to staging
server and iterate on the QA process with potentially another round of
bugfixing until the build reach a state where it’s ready to be deployed on
production.

There’s no merge back to the *dev* branch from *staging* because any
modification done in this branch is related to a specific release only so we
want to merge back the changes to *dev* only when the release is successfully
deployed to production’s server.


Master branch
-------------

The *master* branch is the branch which contains only builds which are
successfully deployed to the production’s server; code which is not
successfully deployed into the serve should not leave any commit in this
branch.


Merging from staging
--------------------

Merging from *staging* branch follows the same rules as explained before, that
is by merging without fast-forward:

    $ git checkout master
    $ git pull
    $ git merge origin/staging --no-ff


Deploying
---------

After a successful deploy tag the current state of the branch with the release
number of this build:

    $ git tag <major>.<minor>.<hotfix>

and push your tag into the remote repo

    $ git push --tags

Tags are really important, they marks a commit as a specific build number of
the project allowing you to find and deploy a previous build or to branch of a
release to apply a hotfix.


Hotfix
------

Sometimes an hotfix is necessary to fix a bug on code currently live on
production or code currently going into production. The hotfix should done in a
separate branch from *master* like with feature and bugfix branches:

    $ git checkout <tag-name>  // checkout the correct build
    $ git checkout -b <hotfix>

and when the work is done merge it back and tag it:

    $ git checkout master
    $ git merge --no-ff <hotfix>
    $ git tag <major>.<minor>.<hotfix>

Remember to increase the hotfix number in the release version


Merging back to dev
-------------------

Every time a new build is tagged in the *master* branch we need to merge it back
into the *dev* branch to be sure that the *dev* branch has all the production
code and hotfixes:

    $ git checkout dev
    $ git pull  // be sure current dev is up-to-date with the remote repository
    $ git merge --no-ff <tag-name>  // merge from tag name
    $ git push  // push changes

That’s it.


[expobrain-git-flow]: https://github.com/expobrain/git-flow
[git-flow]: http://nvie.com/posts/a-successful-git-branching-model/
[codebase]: https://www.codebasehq.com/
