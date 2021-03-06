Writing a patch
===============

.. highlight:: console

Create a local working branch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start by creating a local git branch where you will commit your work::

    git checkout origin/develop -b myfeature

This will create a local git branch called ``myfeature`` that is associated
with the upstream ``develop`` branch for easy rebasing.

For new features and any invasive bug fixes (e.g. those requiring database
changes), the working branch should be based on ``origin/develop`` as shown.

For bug fixes that don't require invasive changes, then the working branch
should be based on the latest release branch. For example, if the latest
release shown on the `release download page
<http://beaker-project.org/releases/>`__ is Beaker 0.15, then local branches
to work on bug fixes should be created with a command like::

    git checkout origin/release-0.15 -b bz123456_fix_this_bug

Including the bug number in the branch name isn't required (since the branch
name is never published to anyone else), but it's a useful reference point
when working on multiple patches in parallel.


Testing your patch
~~~~~~~~~~~~~~~~~~

Beaker has a large and thorough suite of integration tests, including
many `Selenium/WebDriver <http://code.google.com/p/selenium/>`_ browser
tests. You should test your patch by running the test suite either
locally or in Beaker, or both.

In order to run the test suite locally, you must create the additional
test database in your local MySQL instance::

    mysql -uroot <<"EOF"
    CREATE DATABASE beaker_test;
    GRANT ALL ON beaker_test.* TO 'beaker'@'localhost' IDENTIFIED BY 'beaker';
    EOF

In addition, the necessary Selenium support is not yet available as an
RPM, so it is necessary to download `this jar
file <http://code.google.com/p/selenium/downloads/detail?name=selenium-server-standalone-2.35.0.jar&can=1&q=>`_
and save it as
``/usr/local/share/selenium/selenium-server-standalone-2.35.0.jar``.
(Yes, it must be that exact version at that exact path. The longer term
fix to make this step cleaner is to provide an appropriate RPM in the
Beaker repos)

While working on your patch, you would run the unit test for your new
feature/fix::

    cd IntegrationTests/
    ./run-tests.sh -sv bkr.inttest.path_to_my_new_test

Once the new or updated test is passing, you should also run the whole
suite to check for unintended side effects::

    ./run-tests.sh

The ``run-tests.sh`` script is a thin wrapper around
`nosetests <http://readthedocs.org/docs/nose/>`_ which sets up
``PYTHONPATH`` for running from a git checkout.

Once you've verified the patch passes the test suite, commit it to your
local branch::

    git commit

You can also run the Beaker test suite in Beaker (assuming you have
access to a working Beaker instance) using the
``/distribution/beaker/dogfood`` task. If your Beaker instance doesn't
already have a copy of this task, you can build it from Beaker's source
under the ``Tasks`` subdirectory. You can base your job on this `sample
dogfood job XML <../../sample-dogfood-job.xml>`_.

You can use ``Misc/rpmbuild.sh`` to build Beaker RPMs for testing::

    Misc/rpmbuild.sh -bb

or if you have Mock or Koji suitably configured, you can generate an
SRPM and build from that::

    Misc/rpmbuild.sh -bs

The ``Misc/rpmbuild.sh`` script will build from the HEAD commit
in git, so make sure you have committed your changes to your local
branch.

If the patch changes an existing feature or adds a new one,
then ideally the relevant documentation should be updated. Also note that
linking the documented feature to the release notes, and/or using the
'versionchanged/versionadded' directive where appropriate is encouraged.

Submitting your patch
~~~~~~~~~~~~~~~~~~~~~

The Beaker project uses the `Gerrit <http://code.google.com/p/gerrit/>`_
code review tool to manage patches. All patches are reviewed on Beaker's
Gerrit installation before being merged:

    `http://gerrit.beaker-project.org/ <http://gerrit.beaker-project.org>`_

New users can sign in using any OpenID account (preferably your `Fedora
OpenID <http://fedoraproject.org/wiki/OpenID>`_; avoid using Google
Accounts). Be sure to configure your e-mail addresses and SSH keys in
Gerrit after creating your account. Your SSH key is needed to
authenticate you when pushing patches using git. The e-mail address in
your git commits must also match one of the e-mail addresses you have
registered in Gerrit.

For convenience you can add the Gerrit server as a remote to your
``.git/config``::

    [remote "gerrit"]
    url = git+ssh://gerrit.beaker-project.org:29418/beaker

Once you're happy with the change and the test you have written for it,
push your local branch to Gerrit for review::

    git push gerrit myfeature:refs/for/develop

The destination branch in Gerrit should match the branch used as a basis for
the patch. For a bug fix targeting Beaker 0.15, the appropriate command would
look like::

    git push gerrit bz123456_fix_this_bug:refs/for/release-0.15

A new "change" in Gerrit will be created from your commit. Beaker
developers can then review and merge it as appropriate. See the `Gerrit
documentation <http://gerrit.googlecode.com/svn/documentation/2.2.1/index.html>`_
for more info.

If your patch fixes a bug, be sure to include a reference to the
Bugzilla number as a footer line like "Bug: 123456" in the commit
message (`example <http://git.beaker-project.org/c/b/c9bd4bf>`_).

To update the patch on an existing change, you can use
``git commit --amend``. You must ensure that the correct Change-Id
footer appears in your amended commit message. Refer to the Gerrit
`Change-Id <http://gerrit.googlecode.com/svn/documentation/2.2.1/user-changeid.html>`_
documentation for more details.

To avoid forgetting the Change-Id footer and accidentally creating a new
review instead of updating an existing one, it's useful to install this
hook which automatically adds an appropriate "Change-Id" entry to the
commit message when a patch is first committed locally::

    scp -p -P 29418 gerrit.beaker-project.org:hooks/commit-msg .git/hooks/


Reviewing a patch
~~~~~~~~~~~~~~~~~

For a change to make it through review and be merged into the
development branch for the next Beaker release, it needs to first be
marked in Gerrit as "+1 Verified" and have a "+2 Looks good to me,
approved" code review (only the core Beaker developers can grant the
latter).

The "+1 Verified" marker indicates one of the following:

-  If it's a bug fix that is reproduceable and testable, the new test
   case has been verified to fail before the fix, and pass after the
   fix.
-  If it's a bug fix that is not amenable to an automated test, the
   patch has been verified to fix the bug through some other means (such
   as trying it out manually).
-  If it's a new feature, the feature has been verified to work as
   described.
-  If it's a code change, the test suite has been verified to pass in
   full.
-  If it's a docs change, the docs have been verified to build correctly
   and look right.
-  On some rare occasions (for example, fixing a typo in a comment or
   README), it may simply indicate that the patch has been determined
   not to run a risk of breaking the application or documentation.

The "+2 Approved" code review marker should only be granted when all the
following criteria are met:

-  The patch is targetting the right branch (develop for new features and
   invasive bug fixes, latest release branch for non-invasive bug fixes)
-  All significant review comments have been addressed, with the aim of
   ensuring the Beaker code remains maintainable rather than
   degenerating over time.
-  Whenever practical, automated tests have been added to ensure the bug
   fix or new feature works as expected.
-  The code is commented appropriately (for example, explanations or
   issue tracker references are included for any obscure workarounds).
-  The documentation (including docstrings) has been updated
   appropriately
-  A release note has been added as described in the `What's New
   source <http://git.beaker-project.org/cgit/beaker/tree/documentation/whats-new/index.rst?h=develop>`_
   for new features, bug fixes that may break existing workarounds, and
   any changes that require manual steps from system administrators when
   upgrading an existing installation.
-  The commit message is correctly formatted with a short summary line
   and any additional continuation lines separated from the summary by a
   blank line.
-  For changes driven by a Bugzilla entry, the correct "Bug: NNNNNN"
   reference is present in the commit message (as described above in
   "Submitting your patch").
-  For functional and testing changes (that is, changes affecting code, not
   just documentation), another core developer has already granted
   "+1 Looks good to me, but someone else must approve" based on the
   above criteria (this criterion may occasionally be waived based on
   core developer availability).

Reviewers should also be looking for "missing updates": changes which
*should* have been made, but are not part of the current patch. For
example, if a new attribute is added for Jobs, then the Job detail page
should probably be updated to display that attribute as well. Another
example would be that if a patch changes the repo layout, then the
description of that layout in the README file should also be updated.

There's no simple guideline to help identify "what's missing" in cases
that aren't automatically detected through failing tests: it's something
that can only come from experience with Beaker and its code. To minimise
such cases, it is often desirable to add a test case that ensures the
two components are kept in sync, rather than relying on developers to
remember to update both places (assuming the duplication can't be
eliminated entirely by changing the implementation). That way, the
missing updates should be picked up automatically as a failure in the
test suite, rather than requiring the patch creator or reviewer to
notice that additional changes are needed.


Exceptions to the review process
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Core developers are permitted to bypass the review process by setting the
review on their own patches in at least the following circumstances:

- when a previously approved patch needs to be rebased to get Gerrit to merge
  it, but no actual changes were needed as part of the rebase (Gerrit is
  configured to rebase automatically, but the web UI sometimes gets confused
  and hides the submit button even though the rebase would work automatically)
- when minor fixes have been made to a previously approved documentation
  patch (documentation patch reviews are mainly aimed at overall structure
  and picking up omissions and technical errors. Fixing a typo or grammar
  error doesn't require restarting the entire review process)
- updating the git submodules for the beaker-project.org documentation (this
  may be pushed directly to git, bypassing Gerrit entirely)
- design proposal updates (design proposals should generally be discussed on
  the development mailing list rather than in a Gerrit review, although the
  latter can be useful for line-by-line commenting on specific details)
- technical road map updates (the overall technical road map is only updated
  by, or at the direction of, the Beaker Development Lead, rather than using
  the regular change review process)
- any changes to the beaker-administrivia repo (these scripts are just used
  to help with issue management and status tracking, and don't directly
  impact the actual functional code, tests or documentation)

As other exceptional cases are identified, they will also be noted here.
