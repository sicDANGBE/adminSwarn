# Export de projet

_Généré le 2025-12-29T22:46:30+01:00_

## .git/COMMIT_EDITMSG

```text
a

```

## .git/FETCH_HEAD

```text
baffdc9b2df085ac01f9c0420213a40f6da944ff		branch 'main' of https://github.com/sicDANGBE/adminSwarn

```

## .git/HEAD

```text
ref: refs/heads/main

```

## .git/ORIG_HEAD

```text
b4f74b85225b9ef3a4019de98593c4b2da875297

```

## .git/config

```text
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = https://github.com/sicDANGBE/adminSwarn.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
	gk-last-accessed = 2025-12-27T07:41:23.748Z

```

## .git/description

```text
Unnamed repository; edit this file 'description' to name the repository.

```

## .git/hooks/applypatch-msg.sample

```text
#!/bin/sh
#
# An example hook script to check the commit log message taken by
# applypatch from an e-mail message.
#
# The hook should exit with non-zero status after issuing an
# appropriate message if it wants to stop the commit.  The hook is
# allowed to edit the commit message file.
#
# To enable this hook, rename this file to "applypatch-msg".

. git-sh-setup
commitmsg="$(git rev-parse --git-path hooks/commit-msg)"
test -x "$commitmsg" && exec "$commitmsg" ${1+"$@"}
:

```

## .git/hooks/commit-msg.sample

```text
#!/bin/sh
#
# An example hook script to check the commit log message.
# Called by "git commit" with one argument, the name of the file
# that has the commit message.  The hook should exit with non-zero
# status after issuing an appropriate message if it wants to stop the
# commit.  The hook is allowed to edit the commit message file.
#
# To enable this hook, rename this file to "commit-msg".

# Uncomment the below to add a Signed-off-by line to the message.
# Doing this in a hook is a bad idea in general, but the prepare-commit-msg
# hook is more suited to it.
#
# SOB=$(git var GIT_AUTHOR_IDENT | sed -n 's/^\(.*>\).*$/Signed-off-by: \1/p')
# grep -qs "^$SOB" "$1" || echo "$SOB" >> "$1"

# This example catches duplicate Signed-off-by lines.

test "" = "$(grep '^Signed-off-by: ' "$1" |
	 sort | uniq -c | sed -e '/^[ 	]*1[ 	]/d')" || {
	echo >&2 Duplicate Signed-off-by lines.
	exit 1
}

```

## .git/hooks/fsmonitor-watchman.sample

```text
#!/usr/bin/perl

use strict;
use warnings;
use IPC::Open2;

# An example hook script to integrate Watchman
# (https://facebook.github.io/watchman/) with git to speed up detecting
# new and modified files.
#
# The hook is passed a version (currently 2) and last update token
# formatted as a string and outputs to stdout a new update token and
# all files that have been modified since the update token. Paths must
# be relative to the root of the working tree and separated by a single NUL.
#
# To enable this hook, rename this file to "query-watchman" and set
# 'git config core.fsmonitor .git/hooks/query-watchman'
#
my ($version, $last_update_token) = @ARGV;

# Uncomment for debugging
# print STDERR "$0 $version $last_update_token\n";

# Check the hook interface version
if ($version ne 2) {
	die "Unsupported query-fsmonitor hook version '$version'.\n" .
	    "Falling back to scanning...\n";
}

my $git_work_tree = get_working_dir();

my $retry = 1;

my $json_pkg;
eval {
	require JSON::XS;
	$json_pkg = "JSON::XS";
	1;
} or do {
	require JSON::PP;
	$json_pkg = "JSON::PP";
};

launch_watchman();

sub launch_watchman {
	my $o = watchman_query();
	if (is_work_tree_watched($o)) {
		output_result($o->{clock}, @{$o->{files}});
	}
}

sub output_result {
	my ($clockid, @files) = @_;

	# Uncomment for debugging watchman output
	# open (my $fh, ">", ".git/watchman-output.out");
	# binmode $fh, ":utf8";
	# print $fh "$clockid\n@files\n";
	# close $fh;

	binmode STDOUT, ":utf8";
	print $clockid;
	print "\0";
	local $, = "\0";
	print @files;
}

sub watchman_clock {
	my $response = qx/watchman clock "$git_work_tree"/;
	die "Failed to get clock id on '$git_work_tree'.\n" .
		"Falling back to scanning...\n" if $? != 0;

	return $json_pkg->new->utf8->decode($response);
}

sub watchman_query {
	my $pid = open2(\*CHLD_OUT, \*CHLD_IN, 'watchman -j --no-pretty')
	or die "open2() failed: $!\n" .
	"Falling back to scanning...\n";

	# In the query expression below we're asking for names of files that
	# changed since $last_update_token but not from the .git folder.
	#
	# To accomplish this, we're using the "since" generator to use the
	# recency index to select candidate nodes and "fields" to limit the
	# output to file names only. Then we're using the "expression" term to
	# further constrain the results.
	my $last_update_line = "";
	if (substr($last_update_token, 0, 1) eq "c") {
		$last_update_token = "\"$last_update_token\"";
		$last_update_line = qq[\n"since": $last_update_token,];
	}
	my $query = <<"	END";
		["query", "$git_work_tree", {$last_update_line
			"fields": ["name"],
			"expression": ["not", ["dirname", ".git"]]
		}]
	END

	# Uncomment for debugging the watchman query
	# open (my $fh, ">", ".git/watchman-query.json");
	# print $fh $query;
	# close $fh;

	print CHLD_IN $query;
	close CHLD_IN;
	my $response = do {local $/; <CHLD_OUT>};

	# Uncomment for debugging the watch response
	# open ($fh, ">", ".git/watchman-response.json");
	# print $fh $response;
	# close $fh;

	die "Watchman: command returned no output.\n" .
	"Falling back to scanning...\n" if $response eq "";
	die "Watchman: command returned invalid output: $response\n" .
	"Falling back to scanning...\n" unless $response =~ /^\{/;

	return $json_pkg->new->utf8->decode($response);
}

sub is_work_tree_watched {
	my ($output) = @_;
	my $error = $output->{error};
	if ($retry > 0 and $error and $error =~ m/unable to resolve root .* directory (.*) is not watched/) {
		$retry--;
		my $response = qx/watchman watch "$git_work_tree"/;
		die "Failed to make watchman watch '$git_work_tree'.\n" .
		    "Falling back to scanning...\n" if $? != 0;
		$output = $json_pkg->new->utf8->decode($response);
		$error = $output->{error};
		die "Watchman: $error.\n" .
		"Falling back to scanning...\n" if $error;

		# Uncomment for debugging watchman output
		# open (my $fh, ">", ".git/watchman-output.out");
		# close $fh;

		# Watchman will always return all files on the first query so
		# return the fast "everything is dirty" flag to git and do the
		# Watchman query just to get it over with now so we won't pay
		# the cost in git to look up each individual file.
		my $o = watchman_clock();
		$error = $output->{error};

		die "Watchman: $error.\n" .
		"Falling back to scanning...\n" if $error;

		output_result($o->{clock}, ("/"));
		$last_update_token = $o->{clock};

		eval { launch_watchman() };
		return 0;
	}

	die "Watchman: $error.\n" .
	"Falling back to scanning...\n" if $error;

	return 1;
}

sub get_working_dir {
	my $working_dir;
	if ($^O =~ 'msys' || $^O =~ 'cygwin') {
		$working_dir = Win32::GetCwd();
		$working_dir =~ tr/\\/\//;
	} else {
		require Cwd;
		$working_dir = Cwd::cwd();
	}

	return $working_dir;
}

```

## .git/hooks/post-update.sample

```text
#!/bin/sh
#
# An example hook script to prepare a packed repository for use over
# dumb transports.
#
# To enable this hook, rename this file to "post-update".

exec git update-server-info

```

## .git/hooks/pre-applypatch.sample

```text
#!/bin/sh
#
# An example hook script to verify what is about to be committed
# by applypatch from an e-mail message.
#
# The hook should exit with non-zero status after issuing an
# appropriate message if it wants to stop the commit.
#
# To enable this hook, rename this file to "pre-applypatch".

. git-sh-setup
precommit="$(git rev-parse --git-path hooks/pre-commit)"
test -x "$precommit" && exec "$precommit" ${1+"$@"}
:

```

## .git/hooks/pre-commit.sample

```text
#!/bin/sh
#
# An example hook script to verify what is about to be committed.
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.
#
# To enable this hook, rename this file to "pre-commit".

if git rev-parse --verify HEAD >/dev/null 2>&1
then
	against=HEAD
else
	# Initial commit: diff against an empty tree object
	against=$(git hash-object -t tree /dev/null)
fi

# If you want to allow non-ASCII filenames set this variable to true.
allownonascii=$(git config --type=bool hooks.allownonascii)

# Redirect output to stderr.
exec 1>&2

# Cross platform projects tend to avoid non-ASCII filenames; prevent
# them from being added to the repository. We exploit the fact that the
# printable range starts at the space character and ends with tilde.
if [ "$allownonascii" != "true" ] &&
	# Note that the use of brackets around a tr range is ok here, (it's
	# even required, for portability to Solaris 10's /usr/bin/tr), since
	# the square bracket bytes happen to fall in the designated range.
	test $(git diff --cached --name-only --diff-filter=A -z $against |
	  LC_ALL=C tr -d '[ -~]\0' | wc -c) != 0
then
	cat <<\EOF
Error: Attempt to add a non-ASCII file name.

This can cause problems if you want to work with people on other platforms.

To be portable it is advisable to rename the file.

If you know what you are doing you can disable this check using:

  git config hooks.allownonascii true
EOF
	exit 1
fi

# If there are whitespace errors, print the offending file names and fail.
exec git diff-index --check --cached $against --

```

## .git/hooks/pre-merge-commit.sample

```text
#!/bin/sh
#
# An example hook script to verify what is about to be committed.
# Called by "git merge" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message to
# stderr if it wants to stop the merge commit.
#
# To enable this hook, rename this file to "pre-merge-commit".

. git-sh-setup
test -x "$GIT_DIR/hooks/pre-commit" &&
        exec "$GIT_DIR/hooks/pre-commit"
:

```

## .git/hooks/pre-push.sample

```text
#!/bin/sh

# An example hook script to verify what is about to be pushed.  Called by "git
# push" after it has checked the remote status, but before anything has been
# pushed.  If this script exits with a non-zero status nothing will be pushed.
#
# This hook is called with the following parameters:
#
# $1 -- Name of the remote to which the push is being done
# $2 -- URL to which the push is being done
#
# If pushing without using a named remote those arguments will be equal.
#
# Information about the commits which are being pushed is supplied as lines to
# the standard input in the form:
#
#   <local ref> <local oid> <remote ref> <remote oid>
#
# This sample shows how to prevent push of commits where the log message starts
# with "WIP" (work in progress).

remote="$1"
url="$2"

zero=$(git hash-object --stdin </dev/null | tr '[0-9a-f]' '0')

while read local_ref local_oid remote_ref remote_oid
do
	if test "$local_oid" = "$zero"
	then
		# Handle delete
		:
	else
		if test "$remote_oid" = "$zero"
		then
			# New branch, examine all commits
			range="$local_oid"
		else
			# Update to existing branch, examine new commits
			range="$remote_oid..$local_oid"
		fi

		# Check for WIP commit
		commit=$(git rev-list -n 1 --grep '^WIP' "$range")
		if test -n "$commit"
		then
			echo >&2 "Found WIP commit in $local_ref, not pushing"
			exit 1
		fi
	fi
done

exit 0

```

## .git/hooks/pre-rebase.sample

```text
#!/bin/sh
#
# Copyright (c) 2006, 2008 Junio C Hamano
#
# The "pre-rebase" hook is run just before "git rebase" starts doing
# its job, and can prevent the command from running by exiting with
# non-zero status.
#
# The hook is called with the following parameters:
#
# $1 -- the upstream the series was forked from.
# $2 -- the branch being rebased (or empty when rebasing the current branch).
#
# This sample shows how to prevent topic branches that are already
# merged to 'next' branch from getting rebased, because allowing it
# would result in rebasing already published history.

publish=next
basebranch="$1"
if test "$#" = 2
then
	topic="refs/heads/$2"
else
	topic=`git symbolic-ref HEAD` ||
	exit 0 ;# we do not interrupt rebasing detached HEAD
fi

case "$topic" in
refs/heads/??/*)
	;;
*)
	exit 0 ;# we do not interrupt others.
	;;
esac

# Now we are dealing with a topic branch being rebased
# on top of master.  Is it OK to rebase it?

# Does the topic really exist?
git show-ref -q "$topic" || {
	echo >&2 "No such branch $topic"
	exit 1
}

# Is topic fully merged to master?
not_in_master=`git rev-list --pretty=oneline ^master "$topic"`
if test -z "$not_in_master"
then
	echo >&2 "$topic is fully merged to master; better remove it."
	exit 1 ;# we could allow it, but there is no point.
fi

# Is topic ever merged to next?  If so you should not be rebasing it.
only_next_1=`git rev-list ^master "^$topic" ${publish} | sort`
only_next_2=`git rev-list ^master           ${publish} | sort`
if test "$only_next_1" = "$only_next_2"
then
	not_in_topic=`git rev-list "^$topic" master`
	if test -z "$not_in_topic"
	then
		echo >&2 "$topic is already up to date with master"
		exit 1 ;# we could allow it, but there is no point.
	else
		exit 0
	fi
else
	not_in_next=`git rev-list --pretty=oneline ^${publish} "$topic"`
	/usr/bin/perl -e '
		my $topic = $ARGV[0];
		my $msg = "* $topic has commits already merged to public branch:\n";
		my (%not_in_next) = map {
			/^([0-9a-f]+) /;
			($1 => 1);
		} split(/\n/, $ARGV[1]);
		for my $elem (map {
				/^([0-9a-f]+) (.*)$/;
				[$1 => $2];
			} split(/\n/, $ARGV[2])) {
			if (!exists $not_in_next{$elem->[0]}) {
				if ($msg) {
					print STDERR $msg;
					undef $msg;
				}
				print STDERR " $elem->[1]\n";
			}
		}
	' "$topic" "$not_in_next" "$not_in_master"
	exit 1
fi

<<\DOC_END

This sample hook safeguards topic branches that have been
published from being rewound.

The workflow assumed here is:

 * Once a topic branch forks from "master", "master" is never
   merged into it again (either directly or indirectly).

 * Once a topic branch is fully cooked and merged into "master",
   it is deleted.  If you need to build on top of it to correct
   earlier mistakes, a new topic branch is created by forking at
   the tip of the "master".  This is not strictly necessary, but
   it makes it easier to keep your history simple.

 * Whenever you need to test or publish your changes to topic
   branches, merge them into "next" branch.

The script, being an example, hardcodes the publish branch name
to be "next", but it is trivial to make it configurable via
$GIT_DIR/config mechanism.

With this workflow, you would want to know:

(1) ... if a topic branch has ever been merged to "next".  Young
    topic branches can have stupid mistakes you would rather
    clean up before publishing, and things that have not been
    merged into other branches can be easily rebased without
    affecting other people.  But once it is published, you would
    not want to rewind it.

(2) ... if a topic branch has been fully merged to "master".
    Then you can delete it.  More importantly, you should not
    build on top of it -- other people may already want to
    change things related to the topic as patches against your
    "master", so if you need further changes, it is better to
    fork the topic (perhaps with the same name) afresh from the
    tip of "master".

Let's look at this example:

		   o---o---o---o---o---o---o---o---o---o "next"
		  /       /           /           /
		 /   a---a---b A     /           /
		/   /               /           /
	       /   /   c---c---c---c B         /
	      /   /   /             \         /
	     /   /   /   b---b C     \       /
	    /   /   /   /             \     /
    ---o---o---o---o---o---o---o---o---o---o---o "master"


A, B and C are topic branches.

 * A has one fix since it was merged up to "next".

 * B has finished.  It has been fully merged up to "master" and "next",
   and is ready to be deleted.

 * C has not merged to "next" at all.

We would want to allow C to be rebased, refuse A, and encourage
B to be deleted.

To compute (1):

	git rev-list ^master ^topic next
	git rev-list ^master        next

	if these match, topic has not merged in next at all.

To compute (2):

	git rev-list master..topic

	if this is empty, it is fully merged to "master".

DOC_END

```

## .git/hooks/pre-receive.sample

```text
#!/bin/sh
#
# An example hook script to make use of push options.
# The example simply echoes all push options that start with 'echoback='
# and rejects all pushes when the "reject" push option is used.
#
# To enable this hook, rename this file to "pre-receive".

if test -n "$GIT_PUSH_OPTION_COUNT"
then
	i=0
	while test "$i" -lt "$GIT_PUSH_OPTION_COUNT"
	do
		eval "value=\$GIT_PUSH_OPTION_$i"
		case "$value" in
		echoback=*)
			echo "echo from the pre-receive-hook: ${value#*=}" >&2
			;;
		reject)
			exit 1
		esac
		i=$((i + 1))
	done
fi

```

## .git/hooks/prepare-commit-msg.sample

```text
#!/bin/sh
#
# An example hook script to prepare the commit log message.
# Called by "git commit" with the name of the file that has the
# commit message, followed by the description of the commit
# message's source.  The hook's purpose is to edit the commit
# message file.  If the hook fails with a non-zero status,
# the commit is aborted.
#
# To enable this hook, rename this file to "prepare-commit-msg".

# This hook includes three examples. The first one removes the
# "# Please enter the commit message..." help message.
#
# The second includes the output of "git diff --name-status -r"
# into the message, just before the "git status" output.  It is
# commented because it doesn't cope with --amend or with squashed
# commits.
#
# The third example adds a Signed-off-by line to the message, that can
# still be edited.  This is rarely a good idea.

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2
SHA1=$3

/usr/bin/perl -i.bak -ne 'print unless(m/^. Please enter the commit message/..m/^#$/)' "$COMMIT_MSG_FILE"

# case "$COMMIT_SOURCE,$SHA1" in
#  ,|template,)
#    /usr/bin/perl -i.bak -pe '
#       print "\n" . `git diff --cached --name-status -r`
# 	 if /^#/ && $first++ == 0' "$COMMIT_MSG_FILE" ;;
#  *) ;;
# esac

# SOB=$(git var GIT_COMMITTER_IDENT | sed -n 's/^\(.*>\).*$/Signed-off-by: \1/p')
# git interpret-trailers --in-place --trailer "$SOB" "$COMMIT_MSG_FILE"
# if test -z "$COMMIT_SOURCE"
# then
#   /usr/bin/perl -i.bak -pe 'print "\n" if !$first_line++' "$COMMIT_MSG_FILE"
# fi

```

## .git/hooks/push-to-checkout.sample

```text
#!/bin/sh

# An example hook script to update a checked-out tree on a git push.
#
# This hook is invoked by git-receive-pack(1) when it reacts to git
# push and updates reference(s) in its repository, and when the push
# tries to update the branch that is currently checked out and the
# receive.denyCurrentBranch configuration variable is set to
# updateInstead.
#
# By default, such a push is refused if the working tree and the index
# of the remote repository has any difference from the currently
# checked out commit; when both the working tree and the index match
# the current commit, they are updated to match the newly pushed tip
# of the branch. This hook is to be used to override the default
# behaviour; however the code below reimplements the default behaviour
# as a starting point for convenient modification.
#
# The hook receives the commit with which the tip of the current
# branch is going to be updated:
commit=$1

# It can exit with a non-zero status to refuse the push (when it does
# so, it must not modify the index or the working tree).
die () {
	echo >&2 "$*"
	exit 1
}

# Or it can make any necessary changes to the working tree and to the
# index to bring them to the desired state when the tip of the current
# branch is updated to the new commit, and exit with a zero status.
#
# For example, the hook can simply run git read-tree -u -m HEAD "$1"
# in order to emulate git fetch that is run in the reverse direction
# with git push, as the two-tree form of git read-tree -u -m is
# essentially the same as git switch or git checkout that switches
# branches while keeping the local changes in the working tree that do
# not interfere with the difference between the branches.

# The below is a more-or-less exact translation to shell of the C code
# for the default behaviour for git's push-to-checkout hook defined in
# the push_to_deploy() function in builtin/receive-pack.c.
#
# Note that the hook will be executed from the repository directory,
# not from the working tree, so if you want to perform operations on
# the working tree, you will have to adapt your code accordingly, e.g.
# by adding "cd .." or using relative paths.

if ! git update-index -q --ignore-submodules --refresh
then
	die "Up-to-date check failed"
fi

if ! git diff-files --quiet --ignore-submodules --
then
	die "Working directory has unstaged changes"
fi

# This is a rough translation of:
#
#   head_has_history() ? "HEAD" : EMPTY_TREE_SHA1_HEX
if git cat-file -e HEAD 2>/dev/null
then
	head=HEAD
else
	head=$(git hash-object -t tree --stdin </dev/null)
fi

if ! git diff-index --quiet --cached --ignore-submodules $head --
then
	die "Working directory has staged changes"
fi

if ! git read-tree -u -m "$commit"
then
	die "Could not update working tree to new HEAD"
fi

```

## .git/hooks/sendemail-validate.sample

```text
#!/bin/sh

# An example hook script to validate a patch (and/or patch series) before
# sending it via email.
#
# The hook should exit with non-zero status after issuing an appropriate
# message if it wants to prevent the email(s) from being sent.
#
# To enable this hook, rename this file to "sendemail-validate".
#
# By default, it will only check that the patch(es) can be applied on top of
# the default upstream branch without conflicts in a secondary worktree. After
# validation (successful or not) of the last patch of a series, the worktree
# will be deleted.
#
# The following config variables can be set to change the default remote and
# remote ref that are used to apply the patches against:
#
#   sendemail.validateRemote (default: origin)
#   sendemail.validateRemoteRef (default: HEAD)
#
# Replace the TODO placeholders with appropriate checks according to your
# needs.

validate_cover_letter () {
	file="$1"
	# TODO: Replace with appropriate checks (e.g. spell checking).
	true
}

validate_patch () {
	file="$1"
	# Ensure that the patch applies without conflicts.
	git am -3 "$file" || return
	# TODO: Replace with appropriate checks for this patch
	# (e.g. checkpatch.pl).
	true
}

validate_series () {
	# TODO: Replace with appropriate checks for the whole series
	# (e.g. quick build, coding style checks, etc.).
	true
}

# main -------------------------------------------------------------------------

if test "$GIT_SENDEMAIL_FILE_COUNTER" = 1
then
	remote=$(git config --default origin --get sendemail.validateRemote) &&
	ref=$(git config --default HEAD --get sendemail.validateRemoteRef) &&
	worktree=$(mktemp --tmpdir -d sendemail-validate.XXXXXXX) &&
	git worktree add -fd --checkout "$worktree" "refs/remotes/$remote/$ref" &&
	git config --replace-all sendemail.validateWorktree "$worktree"
else
	worktree=$(git config --get sendemail.validateWorktree)
fi || {
	echo "sendemail-validate: error: failed to prepare worktree" >&2
	exit 1
}

unset GIT_DIR GIT_WORK_TREE
cd "$worktree" &&

if grep -q "^diff --git " "$1"
then
	validate_patch "$1"
else
	validate_cover_letter "$1"
fi &&

if test "$GIT_SENDEMAIL_FILE_COUNTER" = "$GIT_SENDEMAIL_FILE_TOTAL"
then
	git config --unset-all sendemail.validateWorktree &&
	trap 'git worktree remove -ff "$worktree"' EXIT &&
	validate_series
fi

```

## .git/hooks/update.sample

```text
#!/bin/sh
#
# An example hook script to block unannotated tags from entering.
# Called by "git receive-pack" with arguments: refname sha1-old sha1-new
#
# To enable this hook, rename this file to "update".
#
# Config
# ------
# hooks.allowunannotated
#   This boolean sets whether unannotated tags will be allowed into the
#   repository.  By default they won't be.
# hooks.allowdeletetag
#   This boolean sets whether deleting tags will be allowed in the
#   repository.  By default they won't be.
# hooks.allowmodifytag
#   This boolean sets whether a tag may be modified after creation. By default
#   it won't be.
# hooks.allowdeletebranch
#   This boolean sets whether deleting branches will be allowed in the
#   repository.  By default they won't be.
# hooks.denycreatebranch
#   This boolean sets whether remotely creating branches will be denied
#   in the repository.  By default this is allowed.
#

# --- Command line
refname="$1"
oldrev="$2"
newrev="$3"

# --- Safety check
if [ -z "$GIT_DIR" ]; then
	echo "Don't run this script from the command line." >&2
	echo " (if you want, you could supply GIT_DIR then run" >&2
	echo "  $0 <ref> <oldrev> <newrev>)" >&2
	exit 1
fi

if [ -z "$refname" -o -z "$oldrev" -o -z "$newrev" ]; then
	echo "usage: $0 <ref> <oldrev> <newrev>" >&2
	exit 1
fi

# --- Config
allowunannotated=$(git config --type=bool hooks.allowunannotated)
allowdeletebranch=$(git config --type=bool hooks.allowdeletebranch)
denycreatebranch=$(git config --type=bool hooks.denycreatebranch)
allowdeletetag=$(git config --type=bool hooks.allowdeletetag)
allowmodifytag=$(git config --type=bool hooks.allowmodifytag)

# check for no description
projectdesc=$(sed -e '1q' "$GIT_DIR/description")
case "$projectdesc" in
"Unnamed repository"* | "")
	echo "*** Project description file hasn't been set" >&2
	exit 1
	;;
esac

# --- Check types
# if $newrev is 0000...0000, it's a commit to delete a ref.
zero=$(git hash-object --stdin </dev/null | tr '[0-9a-f]' '0')
if [ "$newrev" = "$zero" ]; then
	newrev_type=delete
else
	newrev_type=$(git cat-file -t $newrev)
fi

case "$refname","$newrev_type" in
	refs/tags/*,commit)
		# un-annotated tag
		short_refname=${refname##refs/tags/}
		if [ "$allowunannotated" != "true" ]; then
			echo "*** The un-annotated tag, $short_refname, is not allowed in this repository" >&2
			echo "*** Use 'git tag [ -a | -s ]' for tags you want to propagate." >&2
			exit 1
		fi
		;;
	refs/tags/*,delete)
		# delete tag
		if [ "$allowdeletetag" != "true" ]; then
			echo "*** Deleting a tag is not allowed in this repository" >&2
			exit 1
		fi
		;;
	refs/tags/*,tag)
		# annotated tag
		if [ "$allowmodifytag" != "true" ] && git rev-parse $refname > /dev/null 2>&1
		then
			echo "*** Tag '$refname' already exists." >&2
			echo "*** Modifying a tag is not allowed in this repository." >&2
			exit 1
		fi
		;;
	refs/heads/*,commit)
		# branch
		if [ "$oldrev" = "$zero" -a "$denycreatebranch" = "true" ]; then
			echo "*** Creating a branch is not allowed in this repository" >&2
			exit 1
		fi
		;;
	refs/heads/*,delete)
		# delete branch
		if [ "$allowdeletebranch" != "true" ]; then
			echo "*** Deleting a branch is not allowed in this repository" >&2
			exit 1
		fi
		;;
	refs/remotes/*,commit)
		# tracking branch
		;;
	refs/remotes/*,delete)
		# delete tracking branch
		if [ "$allowdeletebranch" != "true" ]; then
			echo "*** Deleting a tracking branch is not allowed in this repository" >&2
			exit 1
		fi
		;;
	*)
		# Anything else (is there anything else?)
		echo "*** Update hook: unknown type of update to ref $refname of type $newrev_type" >&2
		exit 1
		;;
esac

# --- Finished
exit 0

```

## .git/index

> Fichier binaire non inclus (4569 octets)

## .git/info/exclude

```text
# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
# For a project mostly in C, the following would be a good set of
# exclude patterns (uncomment them if you want to use them):
# *.[oa]
# *~

```

## .git/logs/HEAD

```text
0000000000000000000000000000000000000000 baffdc9b2df085ac01f9c0420213a40f6da944ff sicDANGBE <dansoug@gmail.com> 1764885915 +0100	commit (initial): first commit
baffdc9b2df085ac01f9c0420213a40f6da944ff 0000000000000000000000000000000000000000 sicDANGBE <dansoug@gmail.com> 1764885921 +0100	Branch: renamed refs/heads/master to refs/heads/main
0000000000000000000000000000000000000000 baffdc9b2df085ac01f9c0420213a40f6da944ff sicDANGBE <dansoug@gmail.com> 1764885921 +0100	Branch: renamed refs/heads/master to refs/heads/main
baffdc9b2df085ac01f9c0420213a40f6da944ff b4f74b85225b9ef3a4019de98593c4b2da875297 sicDANGBE <dansoug@gmail.com> 1765820519 +0100	commit: cluster fonctionnel à l'exception de rabbitmq
b4f74b85225b9ef3a4019de98593c4b2da875297 cb9babef9a99f668dfedcd1bdef2ae8e4224a37c sicDANGBE <dansoug@gmail.com> 1766054457 +0100	commit: maj chat version
cb9babef9a99f668dfedcd1bdef2ae8e4224a37c 31484b05904cad755c238021fd6f87f529738622 sicDANGBE <dansoug@gmail.com> 1766054494 +0100	commit: remove project_*
31484b05904cad755c238021fd6f87f529738622 3f2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0 sicDANGBE <dansoug@gmail.com> 1766054521 +0100	commit: a

```

## .git/logs/refs/heads/main

```text
0000000000000000000000000000000000000000 baffdc9b2df085ac01f9c0420213a40f6da944ff sicDANGBE <dansoug@gmail.com> 1764885915 +0100	commit (initial): first commit
baffdc9b2df085ac01f9c0420213a40f6da944ff baffdc9b2df085ac01f9c0420213a40f6da944ff sicDANGBE <dansoug@gmail.com> 1764885921 +0100	Branch: renamed refs/heads/master to refs/heads/main
baffdc9b2df085ac01f9c0420213a40f6da944ff b4f74b85225b9ef3a4019de98593c4b2da875297 sicDANGBE <dansoug@gmail.com> 1765820519 +0100	commit: cluster fonctionnel à l'exception de rabbitmq
b4f74b85225b9ef3a4019de98593c4b2da875297 cb9babef9a99f668dfedcd1bdef2ae8e4224a37c sicDANGBE <dansoug@gmail.com> 1766054457 +0100	commit: maj chat version
cb9babef9a99f668dfedcd1bdef2ae8e4224a37c 31484b05904cad755c238021fd6f87f529738622 sicDANGBE <dansoug@gmail.com> 1766054494 +0100	commit: remove project_*
31484b05904cad755c238021fd6f87f529738622 3f2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0 sicDANGBE <dansoug@gmail.com> 1766054521 +0100	commit: a

```

## .git/logs/refs/remotes/origin/main

```text
0000000000000000000000000000000000000000 baffdc9b2df085ac01f9c0420213a40f6da944ff sicDANGBE <dansoug@gmail.com> 1764885950 +0100	update by push
baffdc9b2df085ac01f9c0420213a40f6da944ff b4f74b85225b9ef3a4019de98593c4b2da875297 sicDANGBE <dansoug@gmail.com> 1765820521 +0100	update by push
b4f74b85225b9ef3a4019de98593c4b2da875297 cb9babef9a99f668dfedcd1bdef2ae8e4224a37c sicDANGBE <dansoug@gmail.com> 1766054462 +0100	update by push
cb9babef9a99f668dfedcd1bdef2ae8e4224a37c 3f2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0 sicDANGBE <dansoug@gmail.com> 1766054525 +0100	update by push

```

## .git/objects/01/d2167d3d10068dc2e103cc760a9924a34960dd

> Fichier binaire non inclus (69 octets)

## .git/objects/04/3bf9fef54622a82b0c1489680b71878ee87b26

> Fichier binaire non inclus (1275 octets)

## .git/objects/10/7d589fc91036af8fba1ba129901be91a31897c

> Fichier binaire non inclus (210 octets)

## .git/objects/12/dd915be4a8210d5e1f76057b21a4d05af18278

> Fichier binaire non inclus (195 octets)

## .git/objects/13/4cc55cc85ba228ff3170946b66003ef438fb53

> Fichier binaire non inclus (1737 octets)

## .git/objects/13/a733caa16cec86e457f57dbaf9b8c58385247e

> Fichier binaire non inclus (1284 octets)

## .git/objects/14/ca67e6e6b48fc9890bf8d9708b997f19627940

> Fichier binaire non inclus (2756 octets)

## .git/objects/18/d034bdec206eba68b4493753af53947030b372

> Fichier binaire non inclus (79 octets)

## .git/objects/1c/1473c234147fe7fc247d23fa080183f873340f

> Fichier binaire non inclus (53 octets)

## .git/objects/1d/bdc784eee7f8adc4af6b772c03ebc34b52641b

> Fichier binaire non inclus (434 octets)

## .git/objects/20/54facaec7935a72e976aebfd6605f5e326e514

> Fichier binaire non inclus (693 octets)

## .git/objects/23/b57a36608a8df344086ad7e4b606034537cf12

> Fichier binaire non inclus (53 octets)

## .git/objects/26/535a0c9d10d5f68d0ef58a108717721a32f60e

> Fichier binaire non inclus (610 octets)

## .git/objects/2c/66c08e7d25653c7d562db99229e37aeda0c365

> Fichier binaire non inclus (53 octets)

## .git/objects/2d/77948267ca47c8eee91713a792b82efbe209de

> Fichier binaire non inclus (217 octets)

## .git/objects/2d/844f814c96306db61a7c566a5c2ea2d93e33e9

> Fichier binaire non inclus (225 octets)

## .git/objects/2f/e8b10dd366256fcb894488579d0d219e648ed6

> Fichier binaire non inclus (682 octets)

## .git/objects/30/2920dfd5650225d34cb7f6db2bdd8cd7da4fbf

> Fichier binaire non inclus (67 octets)

## .git/objects/30/a331ae73eef7bb48e0325edb589bced8056792

> Fichier binaire non inclus (60 octets)

## .git/objects/31/484b05904cad755c238021fd6f87f529738622

> Fichier binaire non inclus (166 octets)

## .git/objects/32/909a6b02edc8ac9be3e2fa054bfc8e18123961

> Fichier binaire non inclus (53 octets)

## .git/objects/35/c80e9a5b06ea9878e8dcffd7cbb64089ba0c83

> Fichier binaire non inclus (566 octets)

## .git/objects/36/03809970396a1fa88991d76975b78cd2a8b053

> Fichier binaire non inclus (1879 octets)

## .git/objects/36/9f29c368049f01a11563ef346b98af2702a221

> Fichier binaire non inclus (53 octets)

## .git/objects/37/640b1bf6239e541e2747193080d79460008cf4

> Fichier binaire non inclus (32 octets)

## .git/objects/3b/342248b6e896d0ed8b1dd20adeb2ec35bc4fec

> Fichier binaire non inclus (107 octets)

## .git/objects/3b/4ca10adc45a5be5f487c8246f3ac180d643717

> Fichier binaire non inclus (76 octets)

## .git/objects/3c/d98bd44d5819a990ef6b6925a10d5d4ed1614a

> Fichier binaire non inclus (977 octets)

## .git/objects/3d/b6b4716de113f5dbf8468ea2e41bd40177dfa9

> Fichier binaire non inclus (42844 octets)

## .git/objects/3e/32478c54689abc143e1802215a81fc6c795e13

> Fichier binaire non inclus (47 octets)

## .git/objects/3f/275713e446966701409619f1c5528b64e85221

> Fichier binaire non inclus (366 octets)

## .git/objects/3f/2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0

> Fichier binaire non inclus (155 octets)

## .git/objects/3f/fdc8b7a539862d9eaa85544d6d323d687c28d3

> Fichier binaire non inclus (79 octets)

## .git/objects/43/9e2595364aa6224f96f8fea9ec40297a2bf4c7

> Fichier binaire non inclus (531 octets)

## .git/objects/46/5665e6fe80703c84b1303751ba339298fb884a

> Fichier binaire non inclus (1850 octets)

## .git/objects/4a/7c295e715efc972d9ed4de7c0a25028f5e1c3b

> Fichier binaire non inclus (1117 octets)

## .git/objects/4b/1bbe276df1c05ea6efcac02cec07c6fe5907c2

> Fichier binaire non inclus (746 octets)

## .git/objects/50/e2d1641e7762cf47166a7afccaacb993949448

> Fichier binaire non inclus (24 octets)

## .git/objects/51/3d92feb277350e20774e6cec81e830fe2b4be6

> Fichier binaire non inclus (80 octets)

## .git/objects/53/94e6388554f4a9370de79aca4101438c29d813

> Fichier binaire non inclus (113 octets)

## .git/objects/56/a69a15619c856a37fef233d278b4dc930fce3b

> Fichier binaire non inclus (53 octets)

## .git/objects/58/682fefefd32a7170683511b9bf7f0d11b2a809

> Fichier binaire non inclus (52 octets)

## .git/objects/59/328ca54a045502fba049aafa1852e66bb5f819

> Fichier binaire non inclus (611 octets)

## .git/objects/5c/4877258f52a852c61950b075199a351d78804d

> Fichier binaire non inclus (706 octets)

## .git/objects/61/53a090823205974b35493ef2fc7ef88507e0ef

> Fichier binaire non inclus (210 octets)

## .git/objects/66/2cdad636335aa7ddf87ecc104425ae2a03645b

> Fichier binaire non inclus (163 octets)

## .git/objects/69/e0c1e58e9d8eef6940b3f034e916dff2b9b5ed

> Fichier binaire non inclus (53 octets)

## .git/objects/6a/fc057e641164c8f51ae71ffb807f658fd8d4de

> Fichier binaire non inclus (113 octets)

## .git/objects/6c/3a5652a769d42a705a88a57b3e4c3fa4e9160e

> Fichier binaire non inclus (279 octets)

## .git/objects/7a/469c898b6c29c5529444b2a460cded43955950

```text
x�1�0�����R!�C���g�D,l�YHy��c\��)fC��e��s�C�����]$s�6���_H�5�ܲ�N#K��N���O�����z���L�({oc5�RX��Y��?&�(�
```

## .git/objects/83/6b0fa16df3e85cc2deea25e9dacaeb8b887659

> Fichier binaire non inclus (278 octets)

## .git/objects/8c/b126ddeb5a8a33247c28d4fd65e53ab077d60c

> Fichier binaire non inclus (244 octets)

## .git/objects/8f/4ec334a94058e5cd8a2c34e28102f10f847860

> Fichier binaire non inclus (557 octets)

## .git/objects/92/ad2e70272378effe63b116ed39542ed3273ad7

> Fichier binaire non inclus (1000 octets)

## .git/objects/96/16032d6011eb0cda6fce84acf48c78ec8505de

> Fichier binaire non inclus (1508 octets)

## .git/objects/97/42a5926e1be55a42e870e00305824e2b13ffd5

> Fichier binaire non inclus (1114 octets)

## .git/objects/97/d5407c91f933f0e6e7ffdb77224c4ab0a97364

> Fichier binaire non inclus (244 octets)

## .git/objects/9a/66f582cbe0ce3baeb420da975502068b24e4ab

> Fichier binaire non inclus (244 octets)

## .git/objects/9c/2f2408502613931cbb8aa1c1f85efbe0af7e6b

> Fichier binaire non inclus (80 octets)

## .git/objects/9e/f2e6dadb2ad2f1dc58adf45d6a894ba3474b87

> Fichier binaire non inclus (53 octets)

## .git/objects/a0/801ca44c8761704da73cd617292f1b55b03df8

> Fichier binaire non inclus (433 octets)

## .git/objects/a2/5ca07857e74c16aa08409574d02a04b7aee99f

> Fichier binaire non inclus (211 octets)

## .git/objects/a6/c8114721bb11d3a98a15eb99989c8dd4a8bd5b

> Fichier binaire non inclus (113 octets)

## .git/objects/ac/8640438f29a90bfc27df380a9bdbfef7e405e8

> Fichier binaire non inclus (2015 octets)

## .git/objects/ad/3cedbf5943c12399732301b12e853deb88267b

> Fichier binaire non inclus (98 octets)

## .git/objects/b1/92683d230eac9793518b58be6ca5398abd6e60

> Fichier binaire non inclus (53 octets)

## .git/objects/b3/107607c4998ca8746787a6d21009e795dee3c5

> Fichier binaire non inclus (894 octets)

## .git/objects/b3/a7ab66fd51da17c193ed7a15ba22771183cd7c

> Fichier binaire non inclus (53 octets)

## .git/objects/b4/f74b85225b9ef3a4019de98593c4b2da875297

> Fichier binaire non inclus (193 octets)

## .git/objects/b7/9df636fb4d1a1b11336ff0fc3e7855af93fea9

> Fichier binaire non inclus (415 octets)

## .git/objects/ba/ef302c97d16c79b319d281043571ee94014230

> Fichier binaire non inclus (216 octets)

## .git/objects/ba/ffdc9b2df085ac01f9c0420213a40f6da944ff

> Fichier binaire non inclus (130 octets)

## .git/objects/bc/d9d8c1d40153407dbbdc4d703fc44ce55de36d

> Fichier binaire non inclus (1747 octets)

## .git/objects/be/757d3459959944bbb1537ec4ca355a165ac7d0

> Fichier binaire non inclus (2975 octets)

## .git/objects/bf/7735b0afbb274ffa01336e92fb3b5d6cfad46e

> Fichier binaire non inclus (478 octets)

## .git/objects/c1/89664be7eef83178050a13fc0797ab0a50971d

> Fichier binaire non inclus (2961 octets)

## .git/objects/c7/f1b2b54d4c9a0afd2f170c747bd16f6280f6d4

> Fichier binaire non inclus (763 octets)

## .git/objects/cb/9babef9a99f668dfedcd1bdef2ae8e4224a37c

```text
x��Mj�@@����,k���4$t�;Hc9q�x�=���+d���WZ�K��of�%��'*jXlT1e�Ir�	CRbcQw����sdM��k�y�!O���ca�IR����{�����>���gx�d���r�TY~�J�0��3��8 �G}�u{�*?P���϶}i����E]
```

## .git/objects/cc/50ea877b82eff68f6f1a93895ed4cb0670db3d

> Fichier binaire non inclus (80 octets)

## .git/objects/ce/a31b562c2f2b8318b01e8ee5df5caa52a50188

```text
x���n�0�w�S�9����e�vـ�ڦ#-���R����9�b��M��I�D~���t�c��?�[��Wk8P7b��R�����>�+�}i�e���L��g�Z(.G���fH�4�����朤�lZ�����6l����O��GGu����[_$�b��O-�a25	�վ�Yh���
�1I�j��L�G�䄞���m,!����H�"�R,�b���'�^�3��9�����\J���հj;��؃t���Yz�R�(gV%���ວG�S`��c�mc�;��5J3ҙh��M1֘�1bT���
�����OCqv�y.�d��6��n�B���2��(xN����]P�񀇇3`�n���Ԗ�w��!f�.�uP٩���ņ��a]d�J��Rt!K�����ݾ���v��9�]���j���I�Gb�������N]�%���*��Ꭶ�����7��{������7�2�E�
```

## .git/objects/d2/208d333d9003c158229765ee6203b4afe16603

> Fichier binaire non inclus (1802 octets)

## .git/objects/d2/60905788171669c3bbbc8b68c8349700cc4ac8

> Fichier binaire non inclus (53 octets)

## .git/objects/d4/0cf7f44e7c20ef0b41948b2faa84215effa7da

> Fichier binaire non inclus (1424 octets)

## .git/objects/d4/2d0bb1bce69db01fd85be6c014649939e7583f

```text
x%��
�PE[��m�P1���E���X#<�ś��'��XOܞ{�i�6P\�]����eҼ�4�R�{�{4�M=Ga~�Sc��R!y�e&���#��ư��|��R�q�_!��Te�c�E����ɵ�!pˬ���1�C�.�uڍ�fB�D���n>�
```

## .git/objects/d5/50410fa79954a4d6a62e23a5d6431a14fa0851

> Fichier binaire non inclus (131 octets)

## .git/objects/d7/0f80bc342ea8e3e75964586ed312b4c3e3a9f1

> Fichier binaire non inclus (584 octets)

## .git/objects/da/bf50e0d613b431ad27422ee424607a8cbadc77

> Fichier binaire non inclus (125 octets)

## .git/objects/da/cbfd100534ad0f0b9fe48b1f39d8c28c6f3695

> Fichier binaire non inclus (530 octets)

## .git/objects/db/824122210b5ac33b1f44d89fe652fba5115399

> Fichier binaire non inclus (79 octets)

## .git/objects/dc/8dbb8bee9e0ca3103db0641ebb26a2461ff8ab

> Fichier binaire non inclus (53 octets)

## .git/objects/df/8fc9e1de87bdabbed02bb5fe1709f26ec2c394

> Fichier binaire non inclus (53 octets)

## .git/objects/e1/c411fefdf9f4530d65597c5b98e3b5bb023dee

> Fichier binaire non inclus (76 octets)

## .git/objects/e4/3e39d095f25c76733e3168a4b8fec4d5a22361

> Fichier binaire non inclus (97 octets)

## .git/objects/e4/9cd7abf024e83d1e07e882cf7d6caa6b2f4682

> Fichier binaire non inclus (1272 octets)

## .git/objects/ea/5b21d1f97231962cce183c4de9fce15870075b

> Fichier binaire non inclus (245 octets)

## .git/objects/ed/bf3b870d9e46e5f7dc113970b1dd523ee4e7d4

> Fichier binaire non inclus (195 octets)

## .git/objects/f7/5944e8e80d3243882751be31084ac16eeb5290

> Fichier binaire non inclus (1695 octets)

## .git/objects/f9/565c084436b011278bf3461b07574d3a5eb644

> Fichier binaire non inclus (20126 octets)

## .git/objects/fa/33a4be7bdede970da452b912dd072a56793df5

> Fichier binaire non inclus (709 octets)

## .git/objects/fa/648d77f788a63c178b93dea39c0842dbe528d8

> Fichier binaire non inclus (2994 octets)

## .git/objects/fc/989621e098a2b316d7eb93f8936bc56ceb06ab

> Fichier binaire non inclus (989 octets)

## .git/objects/fc/e88d47c4f8cf4b5dc15c0bb571be962ccebe98

> Fichier binaire non inclus (1053 octets)

## .git/refs/heads/main

```text
3f2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0

```

## .git/refs/remotes/origin/main

```text
3f2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0

```

## .gitignore

```text
project_export.*
```

## deploy.yml

```yaml
---
- name: Déploiement de la Stack d'Administration (Manager)
  hosts: ovh.core
  become: yes
  roles:
    - deploy
```

## group_vars/all/secret.yml

```yaml
# github_token: "AUQKCEL5H6TFEO6QQFTICVLJIPVEA"
github_token: AUQKCEI33FWTVOQQOPODVUDJIP2ZY

postgres_password: 
grafana_admin_password:

user_mail: dansoug@gmail.com
```

## infra/runners/custom-image/Dockerfile

```text
# On part de l'image officielle GitHub Runner (Base Ubuntu)
FROM ghcr.io/actions/actions-runner:latest

# On passe root temporairement pour l'installation
USER root

# Désactivation des invites interactives pour éviter les erreurs de build
ENV DEBIAN_FRONTEND=noninteractive

# 1. Mise à jour système et installation dépendances minimales
# On supprime curl/git/etc à la fin si on veut être ultra-strict, 
# mais ici on en a besoin pour le checkout du code et helm.
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    git \
    unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 2. Installation de Kubectl (Version Stable v1.28.x pour matcher K3s)
# SECURITÉ : On vérifie le SHA256 du binaire téléchargé
RUN curl -LO "https://dl.k8s.io/release/v1.28.5/bin/linux/amd64/kubectl" && \
    curl -LO "https://dl.k8s.io/release/v1.28.5/bin/linux/amd64/kubectl.sha256" && \
    echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check && \
    chmod +x kubectl && \
    mv kubectl /usr/local/bin/ && \
    rm kubectl.sha256

# 3. Installation de Helm (Script officiel)
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 4. DURCISSEMENT (HARDENING)
# Suppression de sudo pour empêcher toute élévation de privilèges
RUN rm -f /usr/bin/sudo

# On repasse sur l'utilisateur standard 'runner' (UID 1001)
# C'est CRITIQUE : le conteneur ne tournera jamais en root
USER runner
```

## infra/runners/deployment.yaml

```yaml
# infra/runners/deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: actions-runner-system

---
# 1. Compte de Service (L'identité du Runner)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-runner-sa
  namespace: actions-runner-system

---
# 2. Rôle (Les permissions : Admin du cluster pour pouvoir tout déployer)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deploy-runner-admin
subjects:
  - kind: ServiceAccount
    name: deploy-runner-sa
    namespace: actions-runner-system
roleRef:
  kind: ClusterRole
  name: cluster-admin # On lui donne les pleins pouvoirs pour gérer les apps
  apiGroup: rbac.authorization.k8s.io

---
# 3. Le Runner lui-même
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-runner-klaro
  namespace: actions-runner-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-runner-klaro
  template:
    metadata:
      labels:
        app: github-runner-klaro
    spec:
      serviceAccountName: deploy-runner-sa
      containers:
      - name: runner
        image: spadmdck/k8s-deploy-runner:v1
        imagePullPolicy: Always
        env:
          - name: GITHUB_REPOSITORY
            value: "sicDANGBE/klaro" # Ton repo d'application
          - name: GITHUB_TOKEN
            value: "AUQKCEL5H6TFEO6QQFTICVLJIPVEA" 
        
        command: ["/bin/bash", "-c"]
        args:
          - |
            # Enregistrement (si pas déjà fait)
            ./config.sh --url https://github.com/sicDANGBE/klaro --token ${GITHUB_TOKEN} --unattended --replace --name k8s-runner-01 --labels k8s-deploy
            # Lancement
            ./run.sh
```

## inventory.yml

```yaml
all:
  # --- VARIABLES GLOBALES (Appliquées à TOUT le monde) ---
  vars:
    sysadmin_user: supadmin
    ansible_user: supadmin
    ansible_port: 49281
    ansible_python_interpreter: /usr/bin/python3
    ansible_become: yes
    local_public_key: "~/.ssh/id_rsa.pub"
    main_domain: dgsynthex.online
    
    # Manager Swarm dynamique (pour compatibilité rôles existants)
    manager_host: "{{ groups['manager'][0] | default('') }}" 

  children:
    swarm_nodes:
      children:
        manager:
          hosts:
            ovh.core:
              ansible_host: vps-940a692a.vps.ovh.net
        workers:
          hosts:
            ovh.worker.01:
              ansible_host: vps-255dab72.vps.ovh.net
            ovh.worker.02:
              ansible_host: vps-9e3ed523.vps.ovh.net
        # Les vars spécifiques à Docker/Swarm peuvent rester ici si besoin
        vars:
          docker_hub_user: spadmdck
          docker_hub_path: "Irmapil-1989"

    k8s_cluster:
      children:
        server:
          hosts:
            ovh.core:
        agents:
          hosts:
            ovh.worker.01:
            ovh.worker.02:
```

## pb/audit_security.yml

```yaml
---
- name: Audit de Sécurité et Configuration
  hosts: swarm_nodes
  become: yes
  gather_facts: yes
  vars:
    report_path: "./reports"

  tasks:
    # --- 1. ETAT DU SYSTEME ---
    - name: Vérification de l'OS et Kernel
      command: uname -a
      register: sys_kernel

    - name: Liste des utilisateurs critiques (passwd)
      shell: grep -E '^(root|supadmin|dockremap|debian)' /etc/passwd
      register: sys_users

    # --- 2. DOCKER (Le point critique actuel) ---
    - name: Lecture du daemon.json
      command: cat /etc/docker/daemon.json
      register: docker_conf
      ignore_errors: yes

    - name: Vérification des fichiers de mapping (subuid)
      command: cat /etc/subuid
      register: subuid_file
      ignore_errors: yes

    - name: Statut du Service Docker
      command: systemctl status docker --no-pager
      register: docker_status
      ignore_errors: yes

    - name: Logs récents de Docker (si erreur)
      shell: journalctl -u docker -n 20 --no-pager
      register: docker_logs
      ignore_errors: yes

    - name: Vérification des permissions dossier Docker
      shell: ls -ld /var/lib/docker
      register: docker_perm
      ignore_errors: yes

    # --- 3. RESEAU & SECURITE ---
    - name: Règles NFTables actives
      command: nft list ruleset
      register: nft_rules
      ignore_errors: yes

    - name: Ports en écoute (SS)
      shell: ss -tulnp
      register: net_ports

    - name: Configuration SSH active (sans commentaires)
      shell: grep -vE '^#|^$' /etc/ssh/sshd_config
      register: ssh_conf

    # --- 4. GENERATION DU RAPPORT LOCAL ---
    
    - name: Création du dossier de rapports local
      delegate_to: localhost
      become: no
      file:
        path: "{{ report_path }}"
        state: directory

    - name: Écriture du rapport localement
      delegate_to: localhost
      become: no
      copy:
        dest: "{{ report_path }}/report_{{ inventory_hostname }}.txt"
        content: |
          =============================================
          RAPPORT D'AUDIT : {{ inventory_hostname }}
          Date : {{ ansible_date_time.iso8601 }}
          IP : {{ ansible_default_ipv4.address }}
          =============================================

          [1] NOYAU & OS
          ---------------------------------------------
          {{ sys_kernel.stdout }}

          [2] UTILISATEURS CLES
          ---------------------------------------------
          {{ sys_users.stdout }}

          [3] DOCKER : CONFIGURATION
          ---------------------------------------------
          >> daemon.json :
          {{ docker_conf.stdout | default('Fichier introuvable') }}

          >> subuid mapping :
          {{ subuid_file.stdout | default('Fichier introuvable') }}
          
          >> Permissions /var/lib/docker :
          {{ docker_perm.stdout | default('Dossier introuvable') }}

          [4] DOCKER : SANTE
          ---------------------------------------------
          >> Status SystemD :
          {{ docker_status.stdout | default('Service introuvable') }}

          >> Derniers Logs (Erreurs potentielles) :
          {{ docker_logs.stdout }}

          [5] PARE-FEU (NFTABLES)
          ---------------------------------------------
          {{ nft_rules.stdout | default('Aucune règle chargée') }}

          [6] PORTS OUVERTS
          ---------------------------------------------
          {{ net_ports.stdout }}

          [7] CONFIG SSH
          ---------------------------------------------
          {{ ssh_conf.stdout }}
```

## pb/configure_firewall_swarm.yml

```yaml
---
- name: Configuration et Validation du Pare-feu (Swarm + Web)
  hosts: swarm_nodes
  become: yes
  vars:
    # Récupération dynamique des IPs
    cluster_ips: "{{ groups['swarm_nodes'] | map('extract', hostvars, ['ansible_default_ipv4', 'address']) | join(', ') }}"
    # On définit explicitement qui est le Manager pour le ciblage des ports Web
    manager_host: "ovh.core" 

  tasks:
    # --- 1. CONFIGURATION DU PARE-FEU ---
    
    - name: Mise à jour de /etc/nftables.conf
      copy:
        dest: /etc/nftables.conf
        validate: '/usr/sbin/nft -c -f %s'
        content: |
          #!/usr/sbin/nft -f
          flush ruleset
          
          # Définition des IPs de confiance (le cluster)
          define SWARM_NODES = { {{ cluster_ips }} }

          table inet filter {
            chain input {
              type filter hook input priority 0; policy drop;

              # Base
              iifname "lo" accept
              ct state established,related accept
              ip protocol icmp accept
              ip6 nexthdr icmpv6 accept

              # SSH Admin (Port variable Ansible)
              tcp dport {{ ansible_port }} accept

              # ACCES WEB (HTTP/HTTPS)
              # Uniquement sur le Manager, car Traefik est en "mode: host" sur ce noeud
              {% if inventory_hostname == manager_host %}
              tcp dport { 80, 443 } accept
              {% endif %}

              # SWARM (Only from Cluster Nodes)
              ip saddr $SWARM_NODES tcp dport { 2377, 7946 } accept
              ip saddr $SWARM_NODES udp dport { 7946, 4789 } accept
              
              # Protocol ESP (Encryption Overlay)
              ip protocol esp accept
            }
            chain forward {
              type filter hook forward priority 0; policy drop;
              ct state established,related accept
              
              # DOCKER LOCAL
              iifname "docker0" accept
              oifname "docker0" accept

              # DOCKER SWARM OVERLAY
              # Nécessaire pour la communication inter-conteneurs et le routing mesh
              iifname "docker_gwbridge" accept
              oifname "docker_gwbridge" accept
            }
            chain output {
              type filter hook output priority 0; policy accept;
            }
          }
        mode: '0755'
      notify: Reload Nftables

    - name: Application immédiate du pare-feu
      meta: flush_handlers

    # --- 2. VALIDATION SSH (Canari local) ---

    - name: Vérification que SSH est toujours vivant
      wait_for_connection:
        timeout: 10
    
    # --- 3. VALIDATION CROISÉE SWARM (Le vrai test) ---

    - name: Installation de netcat (nécessaire pour le test)
      apt:
        pkg: netcat-openbsd
        state: present

    # ETAPE A : On ouvre une fausse écoute sur le Manager (Port 2377)
    # Option -k pour garder le port ouvert pour tous les workers
    - name: Démarrage simulation Swarm sur le Manager
      command: "timeout 25s nc -lk -p 2377"
      async: 30
      poll: 0
      when: inventory_hostname == manager_host
      register: swarm_listener

    # ETAPE B : Les Workers essaient de toucher le Manager
    - name: Test de connexion Swarm (Worker -> Manager)
      wait_for:
        host: "{{ hostvars[manager_host]['ansible_default_ipv4']['address'] }}"
        port: 2377
        timeout: 10
        state: started
      when: inventory_hostname != manager_host

    # ETAPE C : Nettoyage
    - name: Arrêt de la simulation sur le Manager
      shell: "pkill -f 'nc -lk -p 2377'"
      ignore_errors: yes
      when: inventory_hostname == manager_host

    - name: Succès
      debug:
        msg: "Validation RÉUSSIE : Pare-feu configuré (Web sur Manager, Swarm interne OK)."
      run_once: true

  handlers:
    - name: Reload Nftables
      service: name=nftables state=reloaded
```

## pb/init_swarm.yml

```yaml
---
# --- ETAPE 1 : INSTALLATION DES PRÉREQUIS (Sur tous les noeuds) ---
- name: Installation du SDK Python Docker
  hosts: swarm_nodes
  become: yes
  tasks:
    - name: Installation de python3-docker
      apt:
        pkg:
          - python3-docker # Le SDK officiel via APT (Recommandé sur Debian 12)
          - python3-pip    # Toujours utile
        state: present
        update_cache: yes

# --- ETAPE 2 : INITIALISATION DU MANAGER ---
- name: Initialisation du Manager (ovh.core)
  hosts: ovh.core
  become: yes
  tasks:
    - name: Initialiser le Swarm
      community.docker.docker_swarm:
        state: present
        # advertise_addr est crucial pour que les workers sachent qui contacter
        advertise_addr: "{{ ansible_default_ipv4.address }}"
      register: swarm_info

    - name: Récupération du Token Worker
      set_fact:
        worker_token: "{{ swarm_info.swarm_facts.JoinTokens.Worker }}"

# --- ETAPE 3 : JONCTION DES WORKERS ---
- name: Jonction des Workers au Cluster
  hosts: ovh.worker.01, ovh.worker.02
  become: yes
  vars:
    # On récupère l'IP et le Token depuis les "facts" du Manager
    manager_ip: "{{ hostvars['ovh.core']['ansible_default_ipv4']['address'] }}"
    token: "{{ hostvars['ovh.core']['worker_token'] }}"
  tasks:
    - name: Rejoindre le Swarm en tant que Worker
      community.docker.docker_swarm:
        state: join
        join_token: "{{ token }}"
        remote_addrs: [ "{{ manager_ip }}:2377" ]
```

## pb/reports/report_ovh.core.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.core
Date : 2025-12-02T18:31:36Z
IP : 151.80.147.175
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-940a692a 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:38 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18193 (dockerd)
      Tasks: 11
     Memory: 44.3M
        CPU: 7.426s
     CGroup: /system.slice/docker.service
             └─18193 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.818570238Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=cef5a1a49839 ep=gateway_ingress-sbox net=docker_gwbridge nid=dbf523416848
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.829682237Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=85050cbf4cea ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665421496Z" level=info msg="worker qmpr8hijiu95mafd88xinwgen was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665928330Z" level=info msg="worker kohp6n6zegz6r1hnkcuhsoew1 was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682129891Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682266493Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683205218Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683286093Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:26:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:26:03.253409850Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:31:03.453096669Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:02 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:02.640447053Z" level=info msg="dispatcher starting" module=dispatcher node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.154768701Z" level=info msg="manager selected by agent for new session: { }" module=node/agent node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.154978494Z" level=info msg="waiting 0s before registering session" module=node/agent node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.247533103Z" level=info msg="worker vlflk1nflcz8dj3ok7opa91vm was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.250978922Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.147.175 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.147.175 network-control-plane-mtu=1500 remote-addr-list="[]"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.251185482Z" level=info msg="New memberlist node - Node:vps-940a692a will use memberlist nodeID:535ca91ded85 with config:&{NodeID:535ca91ded85 Hostname:vps-940a692a BindAddr:0.0.0.0 AdvertiseAddr:151.80.147.175 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.251163822Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.252069002Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.252253772Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.256792551Z" level=info msg="cluster update event" module=dispatcher node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.818570238Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=cef5a1a49839 ep=gateway_ingress-sbox net=docker_gwbridge nid=dbf523416848
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.829682237Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=85050cbf4cea ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665421496Z" level=info msg="worker qmpr8hijiu95mafd88xinwgen was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665928330Z" level=info msg="worker kohp6n6zegz6r1hnkcuhsoew1 was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682129891Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682266493Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683205218Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683286093Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:26:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:26:03.253409850Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:31:03.453096669Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q       Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0                  0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=11))
udp   UNCONN 0      0               127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=20))
udp   UNCONN 0      0            127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=18))
udp   UNCONN 0      0      151.80.147.175%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=477,fd=10))
udp   UNCONN 0      0                  0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                     [::]:5355          [::]:*    users:(("systemd-resolve",pid=450,fd=13))
udp   UNCONN 0      0                        *:7946             *:*    users:(("dockerd",pid=18193,fd=34))      
tcp   LISTEN 0      4096            127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=21))
tcp   LISTEN 0      4096               0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=12))
tcp   LISTEN 0      4096         127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=19))
tcp   LISTEN 0      128                0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=13523,fd=3))          
tcp   LISTEN 0      4096                     *:7946             *:*    users:(("dockerd",pid=18193,fd=32))      
tcp   LISTEN 0      4096                     *:2377             *:*    users:(("dockerd",pid=18193,fd=27))      
tcp   LISTEN 0      4096                  [::]:5355          [::]:*    users:(("systemd-resolve",pid=450,fd=14))
tcp   LISTEN 0      128                   [::]:49281         [::]:*    users:(("sshd",pid=13523,fd=4))          

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/reports/report_ovh.worker.01.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.worker.01
Date : 2025-12-02T18:31:36Z
IP : 151.80.144.195
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-255dab72 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:37 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18092 (dockerd)
      Tasks: 11
     Memory: 36.9M
        CPU: 5.247s
     CGroup: /system.slice/docker.service
             └─18092 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.691644130Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=60ddb6db40e2 ep=gateway_ingress-sbox net=docker_gwbridge nid=c0f4c74b9ddd
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.703322155Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=a3f1990d8df0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:10.153756722Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:23:35.682745850Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:25:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:25:35.681935285Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:26:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:26:05.680283289Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:29:35.681878714Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.680737396Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.682496041Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:35.686172637Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.671442892Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=qmpr8hijiu95mafd88xinwgen
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.677499893Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.144.195 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.144.195 network-control-plane-mtu=1500 remote-addr-list="[151.80.147.175]"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.677752901Z" level=info msg="New memberlist node - Node:vps-255dab72 will use memberlist nodeID:ed3a6b16b0ce with config:&{NodeID:ed3a6b16b0ce Hostname:vps-255dab72 BindAddr:0.0.0.0 AdvertiseAddr:151.80.144.195 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679137537Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679262362Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679970038Z" level=info msg="The new bootstrap node list is:[151.80.147.175]"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.683252555Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.683323656Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.855151671Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.855283187Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.691644130Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=60ddb6db40e2 ep=gateway_ingress-sbox net=docker_gwbridge nid=c0f4c74b9ddd
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.703322155Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=a3f1990d8df0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:10.153756722Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:23:35.682745850Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:25:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:25:35.681935285Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:26:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:26:05.680283289Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:29:35.681878714Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.680737396Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.682496041Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:35.686172637Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q       Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0               127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=20))
udp   UNCONN 0      0            127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=18))
udp   UNCONN 0      0      151.80.144.195%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=478,fd=10))
udp   UNCONN 0      0                  0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                  0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=11))
udp   UNCONN 0      0                        *:7946             *:*    users:(("dockerd",pid=18092,fd=28))      
udp   UNCONN 0      0                     [::]:5355          [::]:*    users:(("systemd-resolve",pid=433,fd=13))
tcp   LISTEN 0      4096         127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=19))
tcp   LISTEN 0      4096            127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=21))
tcp   LISTEN 0      128                0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=13712,fd=3))          
tcp   LISTEN 0      4096               0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=12))
tcp   LISTEN 0      4096                     *:7946             *:*    users:(("dockerd",pid=18092,fd=27))      
tcp   LISTEN 0      128                   [::]:49281         [::]:*    users:(("sshd",pid=13712,fd=4))          
tcp   LISTEN 0      4096                  [::]:5355          [::]:*    users:(("systemd-resolve",pid=433,fd=14))

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/reports/report_ovh.worker.02.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.worker.02
Date : 2025-12-02T18:31:36Z
IP : 151.80.146.40
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-9e3ed523 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:38 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18407 (dockerd)
      Tasks: 11
     Memory: 37.7M
        CPU: 5.456s
     CGroup: /system.slice/docker.service
             └─18407 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.709409551Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=d4bb76314650 ep=gateway_ingress-sbox net=docker_gwbridge nid=fc4e028e84b8
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.721517407Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=e0a32fdf5ba0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:10.169551923Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:23:35.681905808Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:25:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:25:35.682903222Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:26:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:26:05.680074323Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:29:35.681648880Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.680853183Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.682300403Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:35.685523484Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.671076421Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=kohp6n6zegz6r1hnkcuhsoew1
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.677328290Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.146.40 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.146.40 network-control-plane-mtu=1500 remote-addr-list="[151.80.147.175]"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.677606879Z" level=info msg="New memberlist node - Node:vps-9e3ed523 will use memberlist nodeID:65de0ea72bfb with config:&{NodeID:65de0ea72bfb Hostname:vps-9e3ed523 BindAddr:0.0.0.0 AdvertiseAddr:151.80.146.40 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.679531154Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.679680501Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.680708194Z" level=info msg="The new bootstrap node list is:[151.80.147.175]"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683716227Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683762334Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683789072Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683815623Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.709409551Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=d4bb76314650 ep=gateway_ingress-sbox net=docker_gwbridge nid=fc4e028e84b8
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.721517407Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=e0a32fdf5ba0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:10.169551923Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:23:35.681905808Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:25:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:25:35.682903222Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:26:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:26:05.680074323Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:29:35.681648880Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.680853183Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.682300403Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:35.685523484Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q      Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0              127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=20))
udp   UNCONN 0      0           127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=18))
udp   UNCONN 0      0      151.80.146.40%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=480,fd=10))
udp   UNCONN 0      0                 0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                 0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=11))
udp   UNCONN 0      0                    [::]:5355          [::]:*    users:(("systemd-resolve",pid=452,fd=13))
udp   UNCONN 0      0                       *:7946             *:*    users:(("dockerd",pid=18407,fd=28))      
tcp   LISTEN 0      128               0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=14030,fd=3))          
tcp   LISTEN 0      4096           127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=21))
tcp   LISTEN 0      4096        127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=19))
tcp   LISTEN 0      4096              0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=12))
tcp   LISTEN 0      128                  [::]:49281         [::]:*    users:(("sshd",pid=14030,fd=4))          
tcp   LISTEN 0      4096                 [::]:5355          [::]:*    users:(("systemd-resolve",pid=452,fd=14))
tcp   LISTEN 0      4096                    *:7946             *:*    users:(("dockerd",pid=18407,fd=27))      

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/secure_servers.yml

```yaml
---
- name: Sécurisation des serveurs OVH Debian 12 (Mode Paranoïaque)
  hosts: swarm_nodes
  become: yes
  vars:
    sysadmin_user: "supadmin"
    sysadmin_password: "!" 
    new_ssh_port: 49281
    local_public_key: "~/.ssh/id_rsa.pub"
    
  tasks:
    # --- 1. CONFIGURATION SYSTEME & UTILISATEUR ---
    
    - name: Mise à jour du cache APT et upgrade
      apt:
        update_cache: yes
        upgrade: dist
        autoremove: yes

    - name: Création de l'utilisateur {{ sysadmin_user }}
      user:
        name: "{{ sysadmin_user }}"
        password: "{{ sysadmin_password }}"
        groups: sudo
        shell: /bin/bash
        append: yes
        state: present

    - name: Copie de la clé SSH pour {{ sysadmin_user }}
      authorized_key:
        user: "{{ sysadmin_user }}"
        state: present
        key: "{{ lookup('file', local_public_key) }}"

    - name: Sudo sans mot de passe pour {{ sysadmin_user }}
      copy:
        dest: "/etc/sudoers.d/{{ sysadmin_user }}"
        content: "{{ sysadmin_user }} ALL=(ALL) NOPASSWD: ALL"
        mode: '0440'
        validate: 'visudo -cf %s'

    - name: Création user dockremap (anticipation Docker)
      user:
        name: dockremap
        system: yes
        shell: /usr/sbin/nologin
        create_home: no
        state: present

    - name: Installation des paquets de sécurité
      apt:
        pkg:
          - nftables
          - fail2ban
          - sudo
          - curl
        state: present

    # --- 2. PARE-FEU (CRITIQUE) ---
    
    - name: Configuration NFTables (Double Porte)
      copy:
        dest: /etc/nftables.conf
        content: |
          #!/usr/sbin/nft -f
          flush ruleset

          table inet filter {
            chain input {
              type filter hook input priority 0; policy drop;
              iifname "lo" accept
              ct state established,related accept
              ip protocol icmp accept
              ip6 nexthdr icmpv6 accept

              # 1. On ouvre le NOUVEAU port
              tcp dport {{ new_ssh_port }} accept
              
              # 2. On garde l'ANCIEN port ouvert (Sécurité anti-coupure)
              tcp dport 22 accept
            }
            chain forward {
              type filter hook forward priority 0; policy drop;
            }
            chain output {
              type filter hook output priority 0; policy accept;
            }
          }
        mode: '0755'

    # SECURITE : On force le reload MAINTENANT, pas à la fin du playbook
    - name: Rechargement immédiat de NFTables
      service:
        name: nftables
        state: reloaded
        enabled: yes

    # --- 3. CONFIGURATION SSH (CRITIQUE) ---

    - name: Durcissement SSHD (Changement de port)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^#?Port', line: "Port {{ new_ssh_port }}" }
        - { regexp: '^#?PermitRootLogin', line: "PermitRootLogin no" }
        - { regexp: '^#?PasswordAuthentication', line: "PasswordAuthentication no" }
        - { regexp: '^#?PubkeyAuthentication', line: "PubkeyAuthentication yes" }
        - { regexp: '^#?X11Forwarding', line: "X11Forwarding no" }
        # On s'assure que SSH n'inclut pas d'autres fichiers qui écraseraient nos configs
        - { regexp: '^#?Include', line: "#Include /etc/ssh/sshd_config.d/*.conf" }

    # SECURITE : On vérifie la syntaxe avant de redémarrer
    - name: Vérification de la configuration SSHD
      command: sshd -t
      changed_when: false

    # On redémarre SSH. La session Ansible actuelle ne devrait pas couper,
    # car elle est "established" dans le pare-feu et SSH ne tue pas les processus fils actifs.
    - name: Redémarrage immédiat de SSH
      service:
        name: ssh
        state: restarted

    # --- 4. FAIL2BAN ---

    - name: Protection SSH avec Fail2Ban
      copy:
        dest: /etc/fail2ban/jail.d/ssh_custom.conf
        content: |
          [sshd]
          enabled = true
          port = {{ new_ssh_port }}
          bantime = 1h
          findtime = 10m
          maxretry = 5
      notify: Restart Fail2Ban

  handlers:
    - name: Restart Fail2Ban
      service: name=fail2ban state=restarted
```

## project_export.log

```text
[2025-12-29 22:46:30] Source  : .
[2025-12-29 22:46:30] Sortie  : project_export.md
[2025-12-29 22:46:30] Fichiers trouvés (avant filtre): 177
[2025-12-29 22:46:30] Fichiers à concaténer (après filtre): 176 (exclus auto:1 dir:0 file:0)
[2025-12-29 22:46:30] Concatène [1] .git/COMMIT_EDITMSG (size=2)
[2025-12-29 22:46:30] Concatène [2] .git/FETCH_HEAD (size=99)
[2025-12-29 22:46:30] Concatène [3] .git/HEAD (size=21)
[2025-12-29 22:46:30] Concatène [4] .git/ORIG_HEAD (size=41)
[2025-12-29 22:46:30] Concatène [5] .git/config (size=309)
[2025-12-29 22:46:30] Concatène [6] .git/description (size=73)
[2025-12-29 22:46:30] Concatène [7] .git/hooks/applypatch-msg.sample (size=478)
[2025-12-29 22:46:30] Concatène [8] .git/hooks/commit-msg.sample (size=896)
[2025-12-29 22:46:30] Concatène [9] .git/hooks/fsmonitor-watchman.sample (size=4726)
[2025-12-29 22:46:30] Concatène [10] .git/hooks/post-update.sample (size=189)
[2025-12-29 22:46:30] Concatène [11] .git/hooks/pre-applypatch.sample (size=424)
[2025-12-29 22:46:30] Concatène [12] .git/hooks/pre-commit.sample (size=1643)
[2025-12-29 22:46:30] Concatène [13] .git/hooks/pre-merge-commit.sample (size=416)
[2025-12-29 22:46:30] Concatène [14] .git/hooks/pre-push.sample (size=1374)
[2025-12-29 22:46:30] Concatène [15] .git/hooks/pre-rebase.sample (size=4898)
[2025-12-29 22:46:30] Concatène [16] .git/hooks/pre-receive.sample (size=544)
[2025-12-29 22:46:30] Concatène [17] .git/hooks/prepare-commit-msg.sample (size=1492)
[2025-12-29 22:46:30] Concatène [18] .git/hooks/push-to-checkout.sample (size=2783)
[2025-12-29 22:46:30] Concatène [19] .git/hooks/sendemail-validate.sample (size=2308)
[2025-12-29 22:46:30] Concatène [20] .git/hooks/update.sample (size=3650)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/index — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [22] .git/info/exclude (size=240)
[2025-12-29 22:46:30] Concatène [23] .git/logs/HEAD (size=1155)
[2025-12-29 22:46:30] Concatène [24] .git/logs/refs/heads/main (size=973)
[2025-12-29 22:46:30] Concatène [25] .git/logs/refs/remotes/origin/main (size=576)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/01/d2167d3d10068dc2e103cc760a9924a34960dd — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/04/3bf9fef54622a82b0c1489680b71878ee87b26 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/10/7d589fc91036af8fba1ba129901be91a31897c — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/12/dd915be4a8210d5e1f76057b21a4d05af18278 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/13/4cc55cc85ba228ff3170946b66003ef438fb53 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/13/a733caa16cec86e457f57dbaf9b8c58385247e — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/14/ca67e6e6b48fc9890bf8d9708b997f19627940 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/18/d034bdec206eba68b4493753af53947030b372 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/1c/1473c234147fe7fc247d23fa080183f873340f — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/1d/bdc784eee7f8adc4af6b772c03ebc34b52641b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/20/54facaec7935a72e976aebfd6605f5e326e514 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/23/b57a36608a8df344086ad7e4b606034537cf12 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/26/535a0c9d10d5f68d0ef58a108717721a32f60e — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/2c/66c08e7d25653c7d562db99229e37aeda0c365 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/2d/77948267ca47c8eee91713a792b82efbe209de — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/2d/844f814c96306db61a7c566a5c2ea2d93e33e9 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/2f/e8b10dd366256fcb894488579d0d219e648ed6 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/30/2920dfd5650225d34cb7f6db2bdd8cd7da4fbf — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/30/a331ae73eef7bb48e0325edb589bced8056792 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/31/484b05904cad755c238021fd6f87f529738622 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/32/909a6b02edc8ac9be3e2fa054bfc8e18123961 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/35/c80e9a5b06ea9878e8dcffd7cbb64089ba0c83 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/36/03809970396a1fa88991d76975b78cd2a8b053 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/36/9f29c368049f01a11563ef346b98af2702a221 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/37/640b1bf6239e541e2747193080d79460008cf4 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3b/342248b6e896d0ed8b1dd20adeb2ec35bc4fec — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3b/4ca10adc45a5be5f487c8246f3ac180d643717 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3c/d98bd44d5819a990ef6b6925a10d5d4ed1614a — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3d/b6b4716de113f5dbf8468ea2e41bd40177dfa9 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3e/32478c54689abc143e1802215a81fc6c795e13 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3f/275713e446966701409619f1c5528b64e85221 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3f/2af84dbad3bfcbb57afce2aae3fd74e9e0d3c0 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/3f/fdc8b7a539862d9eaa85544d6d323d687c28d3 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/43/9e2595364aa6224f96f8fea9ec40297a2bf4c7 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/46/5665e6fe80703c84b1303751ba339298fb884a — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/4a/7c295e715efc972d9ed4de7c0a25028f5e1c3b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/4b/1bbe276df1c05ea6efcac02cec07c6fe5907c2 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/50/e2d1641e7762cf47166a7afccaacb993949448 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/51/3d92feb277350e20774e6cec81e830fe2b4be6 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/53/94e6388554f4a9370de79aca4101438c29d813 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/56/a69a15619c856a37fef233d278b4dc930fce3b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/58/682fefefd32a7170683511b9bf7f0d11b2a809 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/59/328ca54a045502fba049aafa1852e66bb5f819 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/5c/4877258f52a852c61950b075199a351d78804d — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/61/53a090823205974b35493ef2fc7ef88507e0ef — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/66/2cdad636335aa7ddf87ecc104425ae2a03645b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/69/e0c1e58e9d8eef6940b3f034e916dff2b9b5ed — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/6a/fc057e641164c8f51ae71ffb807f658fd8d4de — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/6c/3a5652a769d42a705a88a57b3e4c3fa4e9160e — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [75] .git/objects/7a/469c898b6c29c5529444b2a460cded43955950 (size=122)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/83/6b0fa16df3e85cc2deea25e9dacaeb8b887659 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/8c/b126ddeb5a8a33247c28d4fd65e53ab077d60c — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/8f/4ec334a94058e5cd8a2c34e28102f10f847860 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/92/ad2e70272378effe63b116ed39542ed3273ad7 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/96/16032d6011eb0cda6fce84acf48c78ec8505de — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/97/42a5926e1be55a42e870e00305824e2b13ffd5 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/97/d5407c91f933f0e6e7ffdb77224c4ab0a97364 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/9a/66f582cbe0ce3baeb420da975502068b24e4ab — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/9c/2f2408502613931cbb8aa1c1f85efbe0af7e6b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/9e/f2e6dadb2ad2f1dc58adf45d6a894ba3474b87 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/a0/801ca44c8761704da73cd617292f1b55b03df8 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/a2/5ca07857e74c16aa08409574d02a04b7aee99f — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/a6/c8114721bb11d3a98a15eb99989c8dd4a8bd5b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ac/8640438f29a90bfc27df380a9bdbfef7e405e8 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ad/3cedbf5943c12399732301b12e853deb88267b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/b1/92683d230eac9793518b58be6ca5398abd6e60 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/b3/107607c4998ca8746787a6d21009e795dee3c5 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/b3/a7ab66fd51da17c193ed7a15ba22771183cd7c — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/b4/f74b85225b9ef3a4019de98593c4b2da875297 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/b7/9df636fb4d1a1b11336ff0fc3e7855af93fea9 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ba/ef302c97d16c79b319d281043571ee94014230 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ba/ffdc9b2df085ac01f9c0420213a40f6da944ff — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/bc/d9d8c1d40153407dbbdc4d703fc44ce55de36d — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/be/757d3459959944bbb1537ec4ca355a165ac7d0 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/bf/7735b0afbb274ffa01336e92fb3b5d6cfad46e — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/c1/89664be7eef83178050a13fc0797ab0a50971d — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/c7/f1b2b54d4c9a0afd2f170c747bd16f6280f6d4 — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [103] .git/objects/cb/9babef9a99f668dfedcd1bdef2ae8e4224a37c (size=168)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/cc/50ea877b82eff68f6f1a93895ed4cb0670db3d — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [105] .git/objects/ce/a31b562c2f2b8318b01e8ee5df5caa52a50188 (size=497)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/d2/208d333d9003c158229765ee6203b4afe16603 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/d2/60905788171669c3bbbc8b68c8349700cc4ac8 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/d4/0cf7f44e7c20ef0b41948b2faa84215effa7da — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [109] .git/objects/d4/2d0bb1bce69db01fd85be6c014649939e7583f (size=160)
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/d5/50410fa79954a4d6a62e23a5d6431a14fa0851 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/d7/0f80bc342ea8e3e75964586ed312b4c3e3a9f1 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/da/bf50e0d613b431ad27422ee424607a8cbadc77 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/da/cbfd100534ad0f0b9fe48b1f39d8c28c6f3695 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/db/824122210b5ac33b1f44d89fe652fba5115399 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/dc/8dbb8bee9e0ca3103db0641ebb26a2461ff8ab — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/df/8fc9e1de87bdabbed02bb5fe1709f26ec2c394 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/e1/c411fefdf9f4530d65597c5b98e3b5bb023dee — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/e4/3e39d095f25c76733e3168a4b8fec4d5a22361 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/e4/9cd7abf024e83d1e07e882cf7d6caa6b2f4682 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ea/5b21d1f97231962cce183c4de9fce15870075b — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/ed/bf3b870d9e46e5f7dc113970b1dd523ee4e7d4 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/f7/5944e8e80d3243882751be31084ac16eeb5290 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/f9/565c084436b011278bf3461b07574d3a5eb644 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/fa/33a4be7bdede970da452b912dd072a56793df5 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/fa/648d77f788a63c178b93dea39c0842dbe528d8 — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/fc/989621e098a2b316d7eb93f8936bc56ceb06ab — référencé mais non inclus
[2025-12-29 22:46:30] ℹ️  Binaire : .git/objects/fc/e88d47c4f8cf4b5dc15c0bb571be962ccebe98 — référencé mais non inclus
[2025-12-29 22:46:30] Concatène [128] .git/refs/heads/main (size=41)
[2025-12-29 22:46:30] Concatène [129] .git/refs/remotes/origin/main (size=41)
[2025-12-29 22:46:30] Concatène [130] .gitignore (size=16)
[2025-12-29 22:46:30] Concatène [131] deploy.yml (size=117)
[2025-12-29 22:46:30] Concatène [132] group_vars/all/secret.yml (size=166)
[2025-12-29 22:46:30] Concatène [133] infra/runners/custom-image/Dockerfile (size=1513)
[2025-12-29 22:46:30] Concatène [134] infra/runners/deployment.yaml (size=1667)
[2025-12-29 22:46:30] Concatène [135] inventory.yml (size=1132)
[2025-12-29 22:46:30] Concatène [136] pb/audit_security.yml (size=3561)
[2025-12-29 22:46:30] Concatène [137] pb/configure_firewall_swarm.yml (size=3897)
[2025-12-29 22:46:30] Concatène [138] pb/init_swarm.yml (size=1470)
[2025-12-29 22:46:30] Concatène [139] pb/reports/report_ovh.core.txt (size=10872)
[2025-12-29 22:46:30] Concatène [140] pb/reports/report_ovh.worker.01.txt (size=10677)
[2025-12-29 22:46:30] Concatène [141] pb/reports/report_ovh.worker.02.txt (size=10658)
[2025-12-29 22:46:30] Concatène [142] pb/secure_servers.yml (size=4293)

```

## roles/app_k8s/tasks/main.yml

```yaml
---
# roles/app_k8s/tasks/main.yml

# -----------------------------------------------------------
# 0. FIX DNS (CRITIQUE : Réparation post-install K3s)
# -----------------------------------------------------------
- name: "FIX DNS : Configuration de resolv.conf (Google/Cloudflare)"
  copy:
    dest: /etc/resolv.conf
    content: |
      nameserver 1.1.1.1
      nameserver 8.8.8.8
      nameserver 213.186.33.99
    owner: root
    group: root
    mode: '0644'

# -----------------------------------------------------------
# 1. INSTALLATION HELM (Via Script Officiel)
# -----------------------------------------------------------

- name: Téléchargement du script d'installation Helm (v3)
  get_url:
    # C'est bien get-helm-3, la v4 n'existe pas encore
    url: https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    dest: /tmp/get_helm.sh
    mode: '0700'

- name: Exécution du script d'installation Helm
  command: /tmp/get_helm.sh
  args:
    # Idempotence : Si ce fichier existe, on ne relance pas l'install
    creates: /usr/local/bin/helm
  environment:
    # Optionnel : permet d'éviter des erreurs de vérification GPG dans certains cas
    HELM_INSTALL_DIR: /usr/local/bin

- name: Nettoyage du script d'installation
  file:
    path: /tmp/get_helm.sh
    state: absent

# -----------------------------------------------------------
# 2. DÉPENDANCES PYTHON (Pour les modules Ansible K8s)
# -----------------------------------------------------------

- name: Installation librairies Python pour K8s
  apt:
    pkg:
      - python3-kubernetes
      - python3-jsonpatch
      - python3-yaml
    state: present
    update_cache: yes

# -----------------------------------------------------------
# 3. CRÉATION NAMESPACES
# -----------------------------------------------------------

- name: Création des Namespaces K8s
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    name: "{{ item }}"
    api_version: v1
    kind: Namespace
    state: present
  loop:
    - portainer
    - apps
    - monitoring

# -----------------------------------------------------------
# 4. DÉPLOIEMENT DES APPS
# -----------------------------------------------------------

- module_defaults:
    group/kubernetes.core.helm:
      kubeconfig: /etc/rancher/k3s/k3s.yaml
    group/kubernetes.core.k8s:
      kubeconfig: /etc/rancher/k3s/k3s.yaml
  block:
    - include_tasks: traefik.yml
    - include_tasks: portainer.yml
    - include_tasks: rabbitmq.yml
    - include_tasks: n8n.yml
    - include_tasks: nodered.yml
    - include_tasks: mqtt.yml
    - include_tasks: monitoring.yml
    - include_tasks: postgres.yml
```

## roles/app_k8s/tasks/monitoring.yml

```yaml
---
- name: Ajout Repo Prometheus Community
  kubernetes.core.helm_repository:
    name: prometheus-community
    repo_url: https://prometheus-community.github.io/helm-charts

- name: Déploiement Stack Monitoring (Prometheus + Grafana)
  kubernetes.core.helm:
    name: monitoring
    chart_ref: prometheus-community/kube-prometheus-stack
    release_namespace: monitoring
    values:
      grafana:
        enabled: true
        adminPassword: "{{ grafana_admin_password }}" # Utilise Ansible Vault
        ingress:
          enabled: true
          ingressClassName: "traefik"
          hosts:
            - "grafana.{{ main_domain }}"
          annotations:
            traefik.ingress.kubernetes.io/router.entrypoints: websecure
            traefik.ingress.kubernetes.io/router.tls: "true"
            traefik.ingress.kubernetes.io/router.tls.certresolver: "le"
        persistence:
          enabled: true
          storageClass: "{{ storage_class }}"
          size: 5Gi

        additionalDataSources:
          - name: Loki
            type: loki
            url: http://loki-gateway.monitoring.svc.cluster.local
            access: proxy
            jsonData: 
              maxLines: 1000
      
      prometheus:
        prometheusSpec:
          # IMPORTANT : Permet de scraper les ServiceMonitors de TOUS les namespaces (ex: PostgreSQL dans sgcb)
          serviceMonitorSelectorNilUsesHelmValues: false
          podMonitorSelectorNilUsesHelmValues: false
          ruleSelectorNilUsesHelmValues: false
          # Configuration de la rétention
          retention: 15d
          retentionSize: "8GB" # Sécurité pour ne pas saturer le disque
          resources:
            requests:
              memory: "512Mi"
              cpu: "200m"
            limits:
              memory: "2Gi"
          storageSpec:
            volumeClaimTemplate:
              spec:
                storageClassName: "{{ storage_class }}"
                accessModes: ["ReadWriteOnce"]
                resources:
                  requests:
                    storage: 10Gi

        alertmanager:
        config:
          global:
            resolve_timeout: 5m
          route:
            group_by: ['alertname', 'job']
            receiver: "{{user_mail}}" # À remplacer par un webhook (Slack/Discord/Mail)

        
- name: Déploiement Loki
  kubernetes.core.helm:
    name: loki
    chart_ref: grafana/loki
    release_namespace: monitoring
    chart_version: 5.41.0
    values:
      loki:
        auth_enabled: false
        commonConfig:
          replication_factor: 1
        
        # Configuration du stockage S3
        storage:
          type: s3
          s3:
            endpoint: "{{ s3_endpoint }}" # ex: s3.gra.io.cloud.ovh.net
            region: "{{ s3_region }}"     # ex: gra
            bucketnames: "{{ loki_bucket_name }}"
            access_key_id: "{{ s3_access_key }}"
            secret_access_key: "{{ s3_secret_key }}"
            s3forcepathstyle: true

        schemaConfig:
          configs:
            - from: "2024-01-01"
              store: tsdb
              object_store: s3
              schema: v13
              index:
                prefix: index_
                period: 24h

      # Persistence locale pour le cache d'index
      deploymentMode: SingleBinary
      singleBinary:
        replicas: 1
        persistence:
          enabled: true
          size: 5Gi
          storageClass: "{{ storage_class }}"

- name: Déploiement Promtail
  kubernetes.core.helm:
    name: promtail
    chart_ref: grafana/promtail
    release_namespace: monitoring
    values:
      config:
        clients:
          - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

## roles/app_k8s/tasks/mqtt.yml

```yaml
---
# 1. On ajoute un dépôt qui CONTIENT vraiment Mosquitto
- name: Ajout Repo Helm bdclark (Mosquitto)
  kubernetes.core.helm_repository:
    name: bdclark
    repo_url: https://bdclark.github.io/helm-charts
    force_update: yes

# 2. Mise à jour cache (Sécurité)
- name: Force mise à jour du cache Helm
  command: /usr/local/bin/helm repo update
  environment:
    KUBECONFIG: /etc/rancher/k3s/k3s.yaml
  changed_when: false

# 3. Déploiement
- name: Déploiement Mosquitto (Broker MQTT)
  kubernetes.core.helm:
    name: mosquitto
    chart_ref: bdclark/mosquitto
    release_namespace: apps
    create_namespace: true
    values:
      # Authentification (Attention, ce chart utilise une syntaxe différente de Bitnami)
      auth:
        enabled: true
        # On crée un utilisateur admin/secret
        # En prod, utilise des secrets k8s existants
        users:
          - username: "mqtt_user"
            password: "mqtt_password"
      
      # Exposition Service
      service:
        type: LoadBalancer
        ports:
          mqtt:
            port: 1883
            nodePort: 31883 # Optionnel, pour fixer le port si besoin
      
      # Persistance
      persistence:
        enabled: true
        storageClass: "{{ storage_class }}"
        size: 1Gi
```

## roles/app_k8s/tasks/n8n.yml

```yaml
---
# On utilise le chart '8gears' ou similaire qui est populaire pour n8n
- name: Ajout Repo Helm n8n
  kubernetes.core.helm_repository:
    name: cnrancher
    repo_url: https://charts.rancher.io

# Note : Il n'y a pas de Chart "Officiel" n8n parfait, souvent on le fait en YAML pur.
# Mais pour rester dans l'esprit Helm, on va utiliser une définition Deployment simple via le module k8s
# car les charts n8n tiers sont souvent instables.

- name: Déploiement n8n (Deployment + Service + Ingress)
  kubernetes.core.k8s:
    state: present
    namespace: apps
    definition:
      apiVersion: v1
      kind: List
      items:
        - apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: n8n-data
            namespace: apps
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "{{ storage_class }}"
            resources:
              requests:
                storage: 5Gi

        - apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: n8n
            namespace: apps
            labels:
              app: n8n
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: n8n
            template:
              metadata:
                labels:
                  app: n8n
              spec:
                containers:
                  - name: n8n
                    image: n8nio/n8n:latest
                    ports:
                      - containerPort: 5678
                    env:
                      - name: N8N_HOST
                        value: "n8n.{{ main_domain }}"
                      - name: WEBHOOK_URL
                        value: "https://n8n.{{ main_domain }}/"
                    volumeMounts:
                      - name: data
                        mountPath: /home/node/.n8n
                volumes:
                  - name: data
                    persistentVolumeClaim:
                      claimName: n8n-data

        - apiVersion: v1
          kind: Service
          metadata:
            name: n8n
            namespace: apps
          spec:
            selector:
              app: n8n
            ports:
              - port: 80
                targetPort: 5678

        - apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: n8n
            namespace: apps
            annotations:
              traefik.ingress.kubernetes.io/router.entrypoints: websecure
              traefik.ingress.kubernetes.io/router.tls: "true"
              traefik.ingress.kubernetes.io/router.tls.certresolver: "le"
          spec:
            rules:
              - host: "n8n.{{ main_domain }}"
                http:
                  paths:
                    - path: /
                      pathType: Prefix
                      backend:
                        service:
                          name: n8n
                          port:
                            number: 80
```

## roles/app_k8s/tasks/nodered.yml

```yaml
---
# roles/app_k8s/tasks/nodered.yml

# 1. DÉFINITION DES VARIABLES (Pour faciliter la comparaison)
- name: "Définition de la version cible Node-RED"
  set_fact:
    nodered_target_image: "nodered/node-red:4.1.2-22"

# 2. INTROSPECTION (On demande au cluster : 'Tu en es où ?')
- name: "Vérification de l'état actuel de Node-RED"
  kubernetes.core.k8s_info:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    api_version: apps/v1
    kind: Deployment
    name: nodered
    namespace: apps
  register: nodered_status

# 3. CALCUL DU BESOIN DE DÉPLOIEMENT
# On déploie SI :
# - Le déploiement n'existe pas (resources est vide)
# - OU L'image actuelle n'est pas celle qu'on veut
# - OU Le déploiement n'est pas "Ready" (pour réparer un état cassé)
- name: "Analyse de la nécessité de déployer"
  set_fact:
    deploy_needed: >-
      {{
        (nodered_status.resources | length == 0) or
        (nodered_status.resources[0].spec.template.spec.containers[0].image != nodered_target_image) or
        (nodered_status.resources[0].status.readyReplicas | default(0) | int < nodered_status.resources[0].spec.replicas | default(1) | int)
      }}

- name: "Debug : Raison du déploiement"
  debug:
    msg: "Déploiement requis : {{ deploy_needed }}. Image actuelle : {{ nodered_status.resources[0].spec.template.spec.containers[0].image | default('Inconnue') }}"
  when: deploy_needed

# 4. DÉPLOIEMENT CONDITIONNEL
- name: Déploiement Node-RED (Smart Update)
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml # Explicite pour éviter les erreurs de contexte
    state: present
    namespace: apps
    definition:
      apiVersion: v1
      kind: List
      items:
        # --- PERSISTANCE ---
        - apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: nodered-data
            namespace: apps
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "{{ storage_class }}"
            resources:
              requests:
                storage: 5Gi

        # --- DEPLOYMENT ---
        - apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: nodered
            namespace: apps
            labels:
              app: nodered
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: nodered
            template:
              metadata:
                labels:
                  app: nodered
              spec:
                securityContext:
                  fsGroup: 1000
                  runAsUser: 1000
                  runAsGroup: 1000
                containers:
                  - name: nodered
                    image: "{{ nodered_target_image }}" # On utilise la variable définie plus haut
                    imagePullPolicy: Always
                    ports:
                      - containerPort: 1880
                    env:
                      - name: TZ
                        value: "Europe/Paris"
                    volumeMounts:
                      - name: data
                        mountPath: /data
                volumes:
                  - name: data
                    persistentVolumeClaim:
                      claimName: nodered-data

        # --- SERVICE ---
        - apiVersion: v1
          kind: Service
          metadata:
            name: nodered
            namespace: apps
          spec:
            selector:
              app: nodered
            ports:
              - port: 80
                targetPort: 1880

        # --- INGRESS ---
        - apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: nodered
            namespace: apps
            annotations:
              traefik.ingress.kubernetes.io/router.entrypoints: websecure
              traefik.ingress.kubernetes.io/router.tls: "true"
              traefik.ingress.kubernetes.io/router.tls.certresolver: "le"
          spec:
            rules:
              - host: "red.{{ main_domain }}"
                http:
                  paths:
                    - path: /
                      pathType: Prefix
                      backend:
                        service:
                          name: nodered
                          port:
                            number: 80
  # C'EST ICI QUE LA MAGIE OPÈRE :
  when: deploy_needed
```

## roles/app_k8s/tasks/ollama.yml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-storage
  namespace: ocr-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi # Suffisant pour plusieurs modèles Vision (Llama 3.2, Moondream, etc.)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ocr-system
spec:
  replicas: 1 # Tu pourras scaler via HPA plus tard
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
        env:
        - name: OLLAMA_HOST
          value: "0.0.0.0"
        - name: OLLAMA_MODELS
          value: "/root/.ollama/models"
        resources:
          limits:
            cpu: "4"    # Ajuste selon tes nodes OVH
            memory: "8Gi"
          requests:
            cpu: "2"
            memory: "4Gi"
        volumeMounts:
        - name: ollama-models
          mountPath: /root/.ollama/models
      volumes:
      - name: ollama-models
        persistentVolumeClaim:
          claimName: ollama-storage
---
apiVersion: v1
kind: Service
metadata:
  name: ollama-service
  namespace: ocr-system
spec:
  selector:
    app: ollama
  ports:
  - protocol: TCP
    port: 11434
    targetPort: 11434
  type: ClusterIP
```

## roles/app_k8s/tasks/portainer.yml

```yaml
- name: Ajout Repo Helm Portainer
  kubernetes.core.helm_repository:
    name: portainer
    repo_url: https://portainer.github.io/k8s/

- name: Déploiement Portainer CE
  kubernetes.core.helm:
    name: portainer
    chart_ref: portainer/portainer
    release_namespace: portainer
    create_namespace: true
    values:
      ingress:
        enabled: true
        ingressClassName: "traefik"
        annotations:
          # HTTPS automatique via Traefik (CertResolver à configurer si prod)
          traefik.ingress.kubernetes.io/router.entrypoints: websecure
          traefik.ingress.kubernetes.io/router.tls: "true"
          traefik.ingress.kubernetes.io/router.tls.certresolver: "le"
        hosts:
          - host: "portainer.{{ main_domain }}"
            paths:
              - path: "/"
      persistence:
        enabled: true
        size: 10Gi
        storageClass: "{{ storage_class }}"
```

## roles/app_k8s/tasks/postgres.yml

```yaml
---
- name: Ajout du dépôt Helm Bitnami
  kubernetes.core.helm_repository:
    name: bitnami
    repo_url: https://charts.bitnami.com/bitnami

- name: Déploiement de PostgreSQL (Idempotent)
  kubernetes.core.helm:
    name: postgres
    chart_ref: bitnami/postgresql
    release_namespace: sgcb
    create_namespace: yes
    chart_version: 18.2.0
    wait: yes # Assure l'idempotence en attendant que les ressources soient prêtes
    state: present
    values:
      global:
        storageClass: "local-path" # Défaut sur k3s, à adapter selon ton S3/partage
      
      auth:
        database: "ma_base_saas"
        username: "admin_user"
        password: "{{ postgres_password }}" # Utilise Ansible Vault pour ceci
      
      # Configuration pour ton Prometheus
      metrics:
        enabled: true
        service:
          type: ClusterIP
        # Si tu utilises Prometheus Operator (courant avec k3s/kube-prometheus-stack)
        serviceMonitor:
          enabled: true
          namespace: monitoring # Namespace où se trouve ton Prometheus
          interval: 30s

      primary:
        persistence:
          enabled: true
          size: 8Gi
        # Configuration réseau pour l'accessibilité
        service:
          type: ClusterIP
          ports:
            postgresql: 5432

# postgres-postgresql.sgcb.svc.cluster.local <= pour être attaqué depuis l'extérieur
```

## roles/app_k8s/tasks/rabbitmq.yml

```yaml
---
- name: Ajout Repo Helm Bitnami
  kubernetes.core.helm_repository:
    name: bitnami
    repo_url: https://charts.bitnami.com/bitnami

- name: Déploiement RabbitMQ Cluster
  kubernetes.core.helm:
    name: rabbitmq
    chart_ref: bitnami/rabbitmq
    release_namespace: apps
    chart_version: "16.0.14"

    values:

      global:
        security:
          allowInsecureImages: true

      metrics:
        enabled: true
        plugins: "rabbitmq_prometheus"
        serviceMonitor:
          enabled: true
          namespace: "apps"
          interval: 30s
          labels:
            release: "monitoring"

      image:
        registry: docker.io
        repository: bitnamilegacy/rabbitmq
        tag: 4.1.3-debian-12-r1

      replicaCount: 3 # 1 par noeud pour la haute dispo

      auth:
        username: admin
        password: "cdosiuhqsdoifuhqsdoiufh" # A mettre dans un vault idéalement
        erlangCookie: "cdosi_uhq_sd_o_i_f_u_h__q__s_doi_gqmsdofguqsdhfpmuihufh"

      ingress:
        enabled: true
        hostname: "rabbitmq.{{ main_domain }}"
        ingressClassName: "traefik"
        annotations:
          traefik.ingress.kubernetes.io/router.entrypoints: websecure
          traefik.ingress.kubernetes.io/router.tls: "true"
          traefik.ingress.kubernetes.io/router.tls.certresolver: "le"

      persistence:
        enabled: true
        storageClass: "{{ storage_class }}"
        size: 8Gi
```

## roles/app_k8s/tasks/traefik.yml

```yaml

```

## roles/app_k8s/vars/main.yml

```yaml

# Versions des Charts Helm (À jour Fin 2025)
helm_versions:
  portainer: "2.21.5"
  rabbitmq: "12.0.0" # Bitnami
  n8n: "0.14.0"      # OCI ou Community
  mosquitto: "4.0.0" # Bitnami
  monitoring: "45.0.0" # Kube-Prometheus-Stack

# Persistance
storage_class: "local-path" # Le stockage par défaut de K3s
```

## roles/apps/tasks/main.yml

```yaml
---
# 1. Lecture Dynamique (Source de vérité)
- name: Lecture UID dockremap effectif
  shell: 'grep "^dockremap:" /etc/subuid | cut -d: -f2'
  register: remap_uid_out
  changed_when: false

- name: Lecture GID dockremap effectif
  shell: 'grep "^dockremap:" /etc/subgid | cut -d: -f2'
  register: remap_gid_out
  changed_when: false

- set_fact:
    remap_uid: "{{ remap_uid_out.stdout }}"
    remap_gid: "{{ remap_gid_out.stdout }}"

# 2. Réseau Overlay
- name: Réseau Traefik Public
  community.docker.docker_network:
    name: traefik-public
    driver: overlay
    attachable: true
    state: present

# -------------------------------------------------------
# GESTION DES DOSSIERS (SÉPARATION MANAGER / WORKERS)
# -------------------------------------------------------

# 1. DOSSIERS MANAGER UNIQUEMENT
# Ces services sont contraints sur "node.role == manager"
# Inutile de polluer les workers avec ces dossiers.
- name: Création dossiers Apps (Manager uniquement)
  file:
    path: "/srv/docker/{{ item }}"
    state: directory
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0755'
  loop:
    - traefik
    - traefik/acme
    - portainer
    - portainer/data
    - monitoring             # Parent
    - monitoring/prometheus
    - monitoring/grafana
    - monitoring/loki
  # Pas de delegate_to ici, ça s'exécute sur le manager (ovh.core)

# 2. DOSSIERS GLOBAUX (Promtail)
# Promtail tourne en mode "global" (sur tous les noeuds).
# Il a besoin de son dossier de config partout.
- name: Création dossiers Promtail (Cluster-wide)
  file:
    path: "/srv/docker/{{ item[1] }}"
    state: directory
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0755'
  # On ne boucle QUE sur les dossiers nécessaires à Promtail
  loop: "{{ groups['swarm_nodes'] | product(['monitoring', 'monitoring/promtail']) | list }}"
  delegate_to: "{{ item[0] }}"
  vars:
    ansible_become: yes

# 4. ACL Socket
- name: ACL Docker Socket
  acl:
    path: /var/run/docker.sock
    entity: "{{ remap_uid }}"
    etype: user
    permissions: rw
    state: present

- name: Login Docker Hub (Manager)
  community.docker.docker_login:
    username: "{{ docker_hub_user | default(omit) }}"
    password: "{{ docker_hub_path | default(omit) }}"
  when: docker_hub_user is defined

# 5. Déploiement des Stacks

# --- TRAEFIK ---
- name: Configuration Traefik (Template)
  template:
    src: traefik.yaml.j2
    dest: /srv/docker/traefik/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Traefik
  community.docker.docker_stack:
    state: present
    name: traefik
    compose:
      - /srv/docker/traefik/docker-compose.yml
    prune: yes
    with_registry_auth: yes

# --- PORTAINER ---
- name: Configuration Portainer (Template)
  template:
    src: portainer.yaml.j2
    dest: /srv/docker/portainer/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Portainer
  community.docker.docker_stack:
    state: present
    name: portainer
    compose:
      - /srv/docker/portainer/docker-compose.yml
    prune: yes
    with_registry_auth: yes

# --- MONITORING ---
# A. Config Prometheus
- name: Configuration Prometheus (prometheus.yml)
  template:
    src: prometheus.yml.j2
    dest: /srv/docker/monitoring/prometheus/prometheus.yml
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0644'

# B. Config Promtail
- name: Configuration Promtail Cluster-wide (promtail.yaml)
  template:
    src: promtail.yaml.j2
    dest: /srv/docker/monitoring/promtail/config.yaml
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0644'
  loop: "{{ groups['swarm_nodes'] }}"
  delegate_to: "{{ item }}"

# C. Compose Monitoring
- name: Configuration Monitoring (Template)
  template:
    src: monitoring.yaml.j2
    dest: /srv/docker/monitoring/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Monitoring
  community.docker.docker_stack:
    state: present
    name: monitoring
    compose:
      - /srv/docker/monitoring/docker-compose.yml
    prune: yes
    with_registry_auth: yes
```

## roles/apps/templates/monitoring.yaml.j2

```text
version: '3.8'

services:
  # --- STOCKAGE DES LOGS ---
  loki:
    image: grafana/loki:3.0.0
    user: "0:0"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

  # --- COLLECTEUR DE LOGS (Remplace le plugin Docker) ---
  promtail:
    image: grafana/promtail:2.9.0
    user: "0:0" # Root requis pour lire /var/lib/docker
    command: -config.file=/etc/promtail/config.yaml
    volumes:
      - /srv/docker/monitoring/promtail/config.yaml:/etc/promtail/config.yaml:ro
      - /var/lib/docker/{{ remap_uid }}.{{ remap_gid }}/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - monitoring
    deploy:
      mode: global # UN PAR NOEUD (Agent)
      placement:
        constraints: [node.platform.os == linux]

  # --- STOCKAGE DES MÉTRIQUES ---
  prometheus:
    image: prom/prometheus:v2.55.1
    user: "0:0"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    volumes:
      # CORRECTION : On monte le FICHIER explicitement
      - /srv/docker/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      # On garde le dossier pour la data (si tu veux persister la TSDB, ajoute un volume ici)
      # - prometheus_data:/prometheus
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

  # --- COLLECTEUR DE MÉTRIQUES SYSTÈME ---
  node-exporter:
    image: prom/node-exporter:v1.8.0
    user: "0:0" # Root souvent nécessaire pour les accès /proc /sys complets
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/host/root'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro,rslave
    networks:
      - monitoring
    deploy:
      mode: global # UN PAR NOEUD
      placement:
        constraints: [node.platform.os == linux]

  # --- VISUALISATION ---
  grafana:
    image: grafana/grafana:main
    user: "0:0"
    volumes:
      - /srv/docker/monitoring/grafana:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://grafana.{{ main_domain }}
      - GF_SERVER_DOMAIN=grafana.{{ main_domain }}
      - GF_SERVER_SERVE_FROM_SUB_PATH=false
    networks:
      - monitoring
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.grafana.rule=Host(`grafana.{{ main_domain }}`)"
        - "traefik.http.routers.grafana.entrypoints=websecure"
        - "traefik.http.routers.grafana.tls.certresolver=myresolver"
        - "traefik.http.services.grafana.loadbalancer.server.port=3000"

networks:
  monitoring:
    driver: overlay
    driver_opts:
      com.docker.network.driver.mtu: "1300"
  traefik-public:
    external: true
```

## roles/apps/templates/portainer.yaml.j2

```text
version: '3.8'

services:
  agent:
    # MISE A JOUR VERSION
    image: portainer/agent:2.21.5
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    # MISE A JOUR VERSION
    image: portainer/portainer-ce:2.21.5
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    volumes:
      - /srv/docker/portainer/data:/data
    networks:
      - traefik-public
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.rule=Host(`portainer.{{ main_domain }}`)"
        - "traefik.http.routers.portainer.entrypoints=web"
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"
        - "traefik.http.routers.portainer.entrypoints=websecure"
        - "traefik.http.routers.portainer.tls.certresolver=myresolver"

networks:
  traefik-public:
    external: true
  agent_network:
    driver: overlay
    attachable: true
    driver_opts:
      com.docker.network.driver.mtu: "1300"
```

## roles/apps/templates/prometheus.yml.j2

```text
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Auto-découverte des Node Exporters via le DNS Docker Swarm
  - job_name: 'node-exporter'
    dns_sd_configs:
      - names:
          - 'tasks.node-exporter'
        type: 'A'
        port: 9100
```

## roles/apps/templates/promtail.yaml.j2

```text
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    # C'est ICI que la magie opère : on utilise "docker_sd_configs"
    # Cela permet à Promtail de demander à Docker : "Qui sont ces conteneurs ?"
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: status
            values: [running]

    # "relabel_configs" sert à transformer les infos techniques de Docker
    # en labels lisibles pour toi dans Grafana/Loki.
    relabel_configs:
      # 1. Construction du chemin du log via l'ID du conteneur
      - source_labels: ['__meta_docker_container_id']
        target_label: '__path__'
        replacement: '/var/lib/docker/containers/$1/$1-json.log'

      # 2. Récupération du nom de la STACK (ex: monitoring, traefik)
      - source_labels: ['__meta_docker_container_label_com_docker_stack_namespace']
        target_label: 'stack'

      # 3. Récupération du nom du SERVICE (ex: monitoring_prometheus)
      - source_labels: ['__meta_docker_container_label_com_docker_swarm_service_name']
        target_label: 'service'

      # 4. Récupération du nom du CONTENEUR (ex: monitoring_prometheus.1.xB2...)
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        replacement: '$1'
        target_label: 'container'

      # 5. Récupération de l'ID du NODE (ex: ovh.worker.01)
      # Utile pour debugger un serveur spécifique
      - source_labels: ['__meta_docker_node_name']
        target_label: 'node_name'
```

## roles/apps/templates/traefik.yaml.j2

```text
version: '3.8'

services:
  traefik:
    image: traefik:v3.3
    command:
      # NOUVELLE SYNTAXE V3
      - --providers.swarm=true
      - --providers.swarm.endpoint=unix:///var/run/docker.sock
      - --providers.swarm.exposedbydefault=false

      # --- CONFIGURATION HTTP (80) AVEC REDIRECTION HTTPS ---
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https

      # --- CONFIGURATION HTTPS (443) ---
      - --entrypoints.websecure.address=:443

      - --api.dashboard=true
      - --api.insecure=true
      - --accesslog=true
      - --metrics.prometheus=true
      - --metrics.prometheus.buckets=0.1,0.3,1.2,5.0
      - --accesslog=true
      - --log.level=INFO
      # Résolveur de certificats (à décommenter pour la prod)
      - --certificatesresolvers.myresolver.acme.email=admin@dgsynthex.online
      - --certificatesresolvers.myresolver.acme.storage=/acme/acme.json
      - --certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /srv/docker/traefik/acme:/acme
    networks:
      - traefik-public
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
      labels:
        # On active Traefik pour lui-même
        - "traefik.enable=true"
        
        # On définit la règle de routage (Ton domaine)
        - "traefik.http.routers.dashboard.rule=Host(`traefik.{{ main_domain }}`)"
        
        # MAGIE : On pointe vers le service interne de l'API (pas besoin de port)
        - "traefik.http.routers.dashboard.service=api@internal"
        
        # On écoute sur le port 80 (web) et 443 (websecure)
        - "traefik.http.routers.dashboard.entrypoints=web,websecure"
        
        # (Optionnel) Authentification basique pour sécuriser l'IHM plus tard
        # - "traefik.http.routers.dashboard.middlewares=auth"
        - "traefik.http.services.dashboard.loadbalancer.server.port=8080"

networks:
  traefik-public:
    external: true
```

## roles/clean/tasks/main.yml

```yaml
---
# -------------------------------------------------------
# ETAPE : NETTOYAGE NUCLÉAIRE (AUTOMATISÉ)
# -------------------------------------------------------

- name: Arrêt des services Docker (Socket & Service)
  service:
    name: "{{ item }}"
    state: stopped
  loop:
    - docker.socket
    - docker
  ignore_errors: true

- name: Kill de sécurité des processus (shim, runc, containerd)
  shell: |
    pkill -9 dockerd || true
    pkill -9 containerd || true
    pkill -9 containerd-shim || true
    pkill -9 runc || true
  ignore_errors: true
  changed_when: false

# --- LA CORRECTION EST ICI ---
# On boucle sur /proc/mounts pour trouver TOUT ce qui est lié à Docker
# (Network namespaces, Overlay2, Containers, Volumes...)
# On trie à l'envers (-r) pour démonter les enfants avant les parents.
- name: Démontage FORCÉ de tous les points de montage Docker (Run & Lib)
  shell: |
    grep -E '/var/run/docker|/var/lib/docker' /proc/mounts | awk '{print $2}' | sort -r | xargs -r umount -f -l
  changed_when: false
  ignore_errors: true
  register: umount_result

- name: Debug du démontage
  debug:
    msg: "Nettoyage des montages terminé."
  when: umount_result.rc == 0

- name: Démontage SPÉCIFIQUE des Network Namespaces (Anti-Zombie)
  shell: |
    # 1. On démonte explicitement chaque fichier de namespace réseau
    # C'est souvent là que ça bloque (le fameux '1-zv77tswb8w')
    find /var/run/docker/netns -type f 2>/dev/null | xargs -r -I {} umount -l {}
    
    # 2. On démonte le dossier netns lui-même s'il est monté
    umount -l /var/run/docker/netns 2>/dev/null || true

    # 3. On finit par le balayage large classique
    grep -E '/var/run/docker|/var/lib/docker' /proc/mounts | awk '{print $2}' | sort -r | xargs -r umount -f -l
  changed_when: false
  ignore_errors: true
  args:
    executable: /bin/bash

# -----------------------------

- name: Destruction du stockage interne Docker (/var/lib/docker)
  file:
    path: /var/lib/docker
    state: absent

- name: Nettoyage des dossiers runtimes (/var/run/docker)
  file:
    path: /var/run/docker
    state: absent

- name: Suppression des données persistantes SaaS (/srv/docker)
  file:
    path: /srv/docker
    state: absent

# On redémarre Docker pour qu'il régénère ses dossiers proprement
- name: Redémarrage de Docker (Vierge)
  service:
    name: docker
    state: started

- name: Préparation dossier racine /srv/docker
  file:
    path: /srv/docker
    state: directory
    mode: '0755'
```

## roles/common/handlers/main.yml

```yaml
---
- name: Restart SSH
  service: name=ssh state=restarted

- name: Reload NFTables
  service: name=nftables state=reloaded
  notify: Restart Docker

- name: Restart Docker
  service: name=docker state=restarted
```

## roles/common/tasks/main.yml

```yaml
---
- name: Mise à jour du système
  apt:
    update_cache: true
    upgrade: dist
    autoremove: true

- name: Installation des paquets de base
  apt:
    pkg:
      - acl
      - curl
      - sudo
      - nftables
      - fail2ban
      - python3-pip
      - python3-yaml
      - python3-jsondiff
    state: present

# --- GESTION UTILISATEURS ---

- name: Verrouillage de l'utilisateur 'debian' (Sécurité)
  user:
    name: debian
    password_lock: true
    shell: /usr/sbin/nologin
    # On ne supprime pas pour éviter de casser cloud-init

- name: Création utilisateur Admin {{ sysadmin_user }}
  user:
    name: "{{ sysadmin_user }}"
    groups: sudo
    shell: /bin/bash
    append: yes
    state: present

- name: Clé SSH pour {{ sysadmin_user }}
  authorized_key:
    user: "{{ sysadmin_user }}"
    state: present
    # On ajoute un default pour éviter le crash si la variable manque
    key: "{{ lookup('file', local_public_key | default('~/.ssh/id_rsa.pub')) }}"

- name: Sudo sans mot de passe
  copy:
    dest: "/etc/sudoers.d/{{ sysadmin_user }}"
    content: "{{ sysadmin_user }} ALL=(ALL) NOPASSWD: ALL"
    mode: '0440'
    validate: 'visudo -cf %s'

- name: Configuration Sysctl (Optimisations Docker & Loki)
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    # Réduit l'utilisation du SWAP (préserve les perfs)
    - { key: 'vm.swappiness', value: '10' }
    # Indispensable pour Loki / Elasticsearch / SonarQube
    - { key: 'vm.max_map_count', value: '262144' }
    # Augmente le nombre de fichiers inotify (Monitoring fichiers logs)
    - { key: 'fs.inotify.max_user_instances', value: '8192' }
    # Routing IPv4 (Requis Docker)
    - { key: 'net.ipv4.ip_forward', value: '1' }

# --- GESTION SUBUID / SUBGID (Séparation stricte) ---
# debian: 100000
# supadmin: 165536
# dockremap: 231072 (165536 + 65536)

- name: Configuration des SubUIDs
  copy:
    dest: /etc/subuid
    content: |
      debian:100000:65536
      {{ sysadmin_user }}:165536:65536
      dockremap:231072:65536
    mode: '0644'

- name: Configuration des SubGIDs
  copy:
    dest: /etc/subgid
    content: |
      debian:100000:65536
      {{ sysadmin_user }}:165536:65536
      dockremap:231072:65536
    mode: '0644'

# --- NFTABLES (Architecture Modulaire) ---

- name: Création du dossier de configuration modulaire NFTables
  file:
    path: /etc/nftables.d
    state: directory
    mode: '0755'

- name: Configuration Base NFTables (SSH uniquement)
  template:
    src: nftables_base.j2
    dest: /etc/nftables.conf
    mode: '0755'
    validate: '/usr/sbin/nft -c -f %s'
  notify: Reload NFTables

- name: Configuration Logging Rejets Silencieux (Fichier modulaire)
  template:
    src: nftables_logging.conf.j2
    dest: /etc/nftables.d/99-logging.conf
    mode: '0644'
  notify: Reload NFTables

- name: Activation et Démarrage de NFTables
  service:
    name: nftables
    state: started
    enabled: yes

# --- CONFIGURATION SSHD ---

- name: Configuration SSHD
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    validate: 'sshd -t -f %s' # Validation avant restart !
  loop:
    - { regexp: '^#?Port', line: "Port {{ ansible_port }}" }
    - { regexp: '^#?PermitRootLogin', line: "PermitRootLogin no" }
    - { regexp: '^#?PasswordAuthentication', line: "PasswordAuthentication no" }
  notify: Restart SSH

# --- FAIL2BAN ---

- name: Activation et Démarrage de Fail2Ban
  service:
    name: fail2ban
    state: started
    enabled: yes

# --- LOGGING NFTABLES ---

- name: Installation de Rsyslog
  apt:
    pkg: rsyslog
    state: present

- name: Configuration redirection logs NFTables
  copy:
    dest: /etc/rsyslog.d/10-nftables.conf
    content: |
      # Redirection de TOUS les logs commençant par [NFT- vers le fichier dédié
      # Cela capture [NFT-DROP], [NFT-FWD-DROP], [NFT-INPUT-DROP], etc.
      :msg, contains, "[NFT-" -/var/log/nftables.log
      & stop
    mode: '0644'
  notify: Restart Rsyslog

- name: Configuration Logrotate pour NFTables (Évite de saturer le disque)
  copy:
    dest: /etc/logrotate.d/nftables
    content: |
      /var/log/nftables.log {
          rotate 7
          daily
          missingok
          notifempty
          delaycompress
          compress
          postrotate
              /usr/lib/rsyslog/rsyslog-rotate
          endscript
      }
    mode: '0644'
```

## roles/common/templates/nftables_base.j2

```text
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    
    # --- TRAFIC DE CONFIANCE (Loopback) ---
    iifname "lo" accept
    ct state established,related accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # --- SSH ADMIN (Global) ---
    tcp dport {{ ansible_port }} accept
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }
}

# 1. On charge les règles spécifiques (Swarm, Apps...)
# Elles s'ajoutent à la suite des règles ci-dessus
include "/etc/nftables.d/*.conf"

# 2. LOGGING FINAL (Global)
# Cette règle est ajoutée TOUT À LA FIN de la chaîne input.
# Tout paquet qui n'a pas été accepté par le SSH ou par les includes (*) arrivera ici.
# On le logue dans /var/log/nftables.log (via rsyslog) avant qu'il soit jeté par la "policy drop".
# add rule inet filter input limit rate 10/minute log prefix "[NFT-DROP] " flags all

```

## roles/common/templates/nftables_logging.conf.j2

```text
# Configuration des logs pour les paquets rejetés (Silent Drops)
# Ce fichier est chargé en dernier (99-logging).

table inet filter {
    # On ajoute ces règles aux chaînes existantes sans les écraser

    chain input {
        # Log des paquets entrants rejetés avant la policy drop
        limit rate 10/minute log prefix "[NFT-INPUT-DROP] " flags all
    }

    chain forward {
        # Log des paquets traversants (Docker/Swarm) rejetés
        # C'est ici que tu verras si le pare-feu bloque Traefik <-> Grafana
        limit rate 20/minute log prefix "[NFT-FWD-DROP] " flags all
    }
}
```

## roles/docker/handlers/main.yml

```yaml
---
- name: Restart Docker
  service: name=docker state=restarted
```

## roles/docker/tasks/main.yml

```yaml
---
# --- 1. INSTALLATION & PRÉREQUIS ---


- name: Installation des dépendances système pour Docker
  apt:
    pkg:
      - ca-certificates
      - curl
      - gnupg
    state: present
    update_cache: true

- name: Création du dossier pour les clés GPG (si absent)
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Ajout de la clé GPG officielle Docker
  get_url:
    url: https://download.docker.com/linux/debian/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
    force: true

- name: Ajout du dépôt Docker stable
  shell: |
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null
  args:
    creates: /etc/apt/sources.list.d/docker.list

- name: Mise à jour du cache APT après ajout du dépôt
  apt:
    update_cache: true

- name: Installation Docker Engine & Outils
  apt:
    pkg:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
      - python3-docker   # Requis pour les modules Ansible Docker
      - iptables         # Requis pour la compatibilité nftables
    state: present

# --- 2. CONFIGURATION UTILISATEUR & SÉCURITÉ ---

- name: Création utilisateur système dockremap
  user:
    name: dockremap
    system: yes
    state: present
    # Note : Les subuids sont gérés par le rôle 'common', on ne touche pas ici.

- name: Configuration Daemon Docker (UserNS + Logs + No Live Restore)
  copy:
    dest: /etc/docker/daemon.json
    content: |
      {
        "userns-remap": "default",
        "log-driver": "json-file",
        "log-opts": { "max-size": "10m", "max-file": "3" },
        "no-new-privileges": true,
        "min-api-version": "1.24",
        "mtu": 1300
      }
    mode: '0644'
  register: docker_config

# --- 3. GESTION INTELLIGENTE DU CHANGEMENT D'UID ---
# Si on change le subuid ou active le remap, il faut nettoyer /var/lib/docker
# sinon Docker plante au démarrage à cause des mauvaises permissions.

- name: Vérification existence dossier Docker
  stat:
    path: /var/lib/docker
  register: docker_dir

# - name: Nettoyage Docker si changement d'architecture UserNS
#   file:
#     path: /var/lib/docker
#     state: absent
#   # On supprime uniquement si la config daemon.json a changé ET que le dossier existait déjà
#   when: docker_config.changed and docker_dir.stat.exists
#   notify: Restart Docker

# --- 4. COMPATIBILITÉ RÉSEAU & SWARM ---

- name: Configuration alternatives iptables-nft (Fix Swarm Debian 12)
  alternatives:
    name: iptables
    path: /usr/sbin/iptables-nft
  notify: Restart Docker

# --- 5. DÉMARRAGE & FINALISATION ---

- name: Démarrage et Activation de Docker
  service:
    name: docker
    state: started
    enabled: yes

- name: Ajout de l'Admin au groupe Docker
  user:
    name: "{{ sysadmin_user }}"
    groups: docker
    append: yes

- name: Fix MSS Clamping (Docker User Chain) - TCPMSS
  shell: |
    # Vérifie si la règle existe déjà pour éviter les doublons
    iptables -t filter -C DOCKER-USER -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200 2>/dev/null || \
    # Si elle n'existe pas, on l'insère en première position
    iptables -t filter -I DOCKER-USER -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200
  changed_when: false
  # On ignore les erreurs si iptables-legacy vs nft pose souci, mais sur Debian 12 ça passe via le wrapper
  ignore_errors: true

# --- 6. GESTION PERMISSIONS SOCKET (CRITIQUE POUR PORTAINER/TRAEFIK) ---

- name: Récupération de l'UID remappé (dockremap)
  shell: 'grep "^dockremap:" /etc/subuid | cut -d: -f2'
  register: dockremap_uid_check
  changed_when: false
  check_mode: no

- name: Application ACL sur le socket Docker pour le user remappé
  acl:
    path: /var/run/docker.sock
    entity: "{{ dockremap_uid_check.stdout }}"
    etype: user
    permissions: rw
    state: present
  when: dockremap_uid_check.stdout != ""
```

## roles/k8s/defaults/main.yml

```yaml
---
# roles/k8s/defaults/main.yml

# Version de K3s (Pinnée pour la stabilité)
k3s_version: "v1.28.5+k3s1"

# Interface réseau pour Flannel (Overlay)
k3s_flannel_iface: "ens3"
```

## roles/k8s/tasks/main.yml

```yaml
---
# roles/k8s/tasks/main.yml

# --- 1. PRÉPARATION SYSTÈME ---

- name: Désactivation du Swap (Requis pour K8s)
  shell: |
    swapoff -a
    sed -i '/swap/d' /etc/fstab
  when: ansible_facts['swaptotal_mb'] | default(0) > 0

- name: Chargement des modules noyau requis
  modprobe:
    name: "{{ item }}"
    state: present
  loop:
    - overlay
    - br_netfilter

- name: Configuration Sysctl pour K8s
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { key: 'net.bridge.bridge-nf-call-iptables', value: '1' }
    - { key: 'net.ipv4.ip_forward', value: '1' }
    - { key: 'net.bridge.bridge-nf-call-ip6tables', value: '1' }

# --- 2. GESTION FIREWALL ---

- name: Nettoyage des règles Firewall Swarm (Obsolètes)
  file:
    path: /etc/nftables.d/10-swarm.conf
    state: absent
  notify: Reload NFTables

- name: Déploiement règles NFTables K8s
  template:
    # CORRECTION ICI : Chemin relatif au rôle k8s (Ansible cherche auto dans roles/k8s/templates)
    src: nftables_k8s.conf.j2
    dest: /etc/nftables.d/20-k8s.conf
    mode: '0644'
  notify: Reload NFTables

- name: Application immédiate du Pare-feu
  meta: flush_handlers

# --- 3. INSTALLATION DU SERVER (MASTER) ---

- name: Installation K3s Server
  shell: >
    curl -sfL https://get.k3s.io | 
    INSTALL_K3S_VERSION="{{ k3s_version }}" 
    INSTALL_K3S_EXEC="server --node-ip {{ ansible_facts['default_ipv4']['address'] }} --flannel-iface {{ k3s_flannel_iface }} --disable-cloud-controller --write-kubeconfig-mode 644" 
    sh -
  args:
    creates: /var/lib/rancher/k3s/server/node-token
  when: "'server' in group_names"

- name: Attente de la création du fichier Token
  wait_for:
    path: /var/lib/rancher/k3s/server/node-token
    state: present
    timeout: 300
    msg: "Le fichier node-token n'a pas été créé à temps. Vérifiez 'systemctl status k3s'."
  when: "'server' in group_names"

- name: Récupération du Token du Cluster
  slurp:
    src: /var/lib/rancher/k3s/server/node-token
  register: k3s_token_b64
  when: "'server' in group_names"

- name: Sauvegarde du Token en local (Contrôleur)
  copy:
    content: "{{ k3s_token_b64['content'] | b64decode | trim }}"
    dest: "/tmp/k3s_cluster_token"
    mode: '0644'
  delegate_to: localhost
  become: false
  when: "'server' in group_names"

# --- 4. INSTALLATION DES AGENTS (WORKERS) ---

- name: Installation K3s Agent
  vars:
    master_host_name: "{{ groups['server'][0] }}"
    master_url: "{{ hostvars[master_host_name]['ansible_host'] }}"
    master_token: "{{ lookup('file', '/tmp/k3s_cluster_token') }}"
    my_ip: "{{ ansible_facts['default_ipv4']['address'] }}"
  shell: >
    curl -sfL https://get.k3s.io | 
    K3S_URL=https://{{ master_url }}:6443 
    K3S_TOKEN={{ master_token }} 
    INSTALL_K3S_VERSION="{{ k3s_version }}" 
    INSTALL_K3S_EXEC="agent --node-ip {{ my_ip }} --flannel-iface {{ k3s_flannel_iface }}" 
    sh -
  args:
    creates: /var/lib/rancher/k3s/agent/client-kubelet.crt
  when: "'agents' in group_names"

# --- 5. CONFORT ADMIN ---

- name: Création dossier .kube pour {{ sysadmin_user }}
  file:
    path: "/home/{{ sysadmin_user }}/.kube"
    state: directory
    owner: "{{ sysadmin_user }}"
    group: "{{ sysadmin_user }}"
    mode: '0750'
  when: "'server' in group_names"

- name: Copie de la config Kube pour l'utilisateur
  copy:
    src: /etc/rancher/k3s/k3s.yaml
    dest: "/home/{{ sysadmin_user }}/.kube/config"
    remote_src: yes
    owner: "{{ sysadmin_user }}"
    group: "{{ sysadmin_user }}"
    mode: '0600'
  when: "'server' in group_names"

- name: Ajout de l'alias et autocompletion
  blockinfile:
    path: "/home/{{ sysadmin_user }}/.bashrc"
    marker: "# {mark} ANSIBLE MANAGED BLOCK K8S"
    block: |
      alias k='kubectl'
      source <(kubectl completion bash)
      complete -o default -F __start_kubectl k
      export KUBECONFIG=/home/{{ sysadmin_user }}/.kube/config
  when: "'server' in group_names"

# --- 6. NETTOYAGE ---

- name: Suppression du token local (Sécurité)
  file:
    path: "/tmp/k3s_cluster_token"
    state: absent
  delegate_to: localhost
  become: false
  run_once: true

- name: Configuration Traefik ACME (Let's Encrypt)
  template:
    src: traefik-config.yaml.j2
    dest: /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
    mode: '0644'
  when: "'server' in group_names"
```

## roles/k8s/templates/nftables_k8s.conf.j2

```text
# /etc/nftables.d/20-k8s.conf
# Règles pour Kubernetes (K3s)

# 1. Liste des IPs Physiques (Tes serveurs OVH)
define K8S_NODES = { {{ groups['k8s_cluster'] | map('extract', hostvars, ['ansible_host']) | join(', ') }} }

# 2. Réseaux Internes K3s (Valeurs par défaut)
# Pods CIDR: 10.42.0.0/16
# Services CIDR: 10.43.0.0/16
define K8S_PODS = 10.42.0.0/16
define K8S_SVCS = 10.43.0.0/16

table inet filter {
  chain input {
    # --- API SERVER (6443) ---
    # Autoriser les Nœuds ET les Pods à parler au Master
    ip saddr $K8S_NODES tcp dport 6443 accept
    ip saddr $K8S_PODS tcp dport 6443 accept

    # --- KUBELET & METRICS (10250) ---
    ip saddr $K8S_NODES tcp dport 10250 accept
    ip saddr $K8S_PODS tcp dport 10250 accept

    # --- FLANNEL VXLAN (8472) ---
    # Réseau Overlay (Vital pour que les pods se parlent entre noeuds)
    ip saddr $K8S_NODES udp dport 8472 accept

    # --- ACCÈS CLIENTS (Traefik) ---
    # Uniquement sur le Master (ou partout si Traefik est en DaemonSet hostPort)
    # K3s par défaut expose Traefik sur les ports 80/443 du noeud
    tcp dport { 80, 443 } accept
  }

  chain forward {
    # --- ROUTAGE INTERNE K8S ---
    # On autorise tout le trafic qui vient des pods ou va vers les pods
    ip saddr $K8S_PODS accept
    ip daddr $K8S_PODS accept
    
    # On garde la confiance sur les interfaces CNI
    iifname "cni0" accept
    oifname "cni0" accept
    iifname "flannel.1" accept
    oifname "flannel.1" accept
  }
}
```

## roles/k8s/templates/traefik-config.yaml.j2

```text
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    # On active les logs pour debugger
    logs:
      general:
        level: INFO
      access:
        enabled: true

    # Configuration Let's Encrypt (ACME)
    additionalArguments:
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.email=admin@dgsynthex.online"
      - "--certificatesresolvers.le.acme.storage=/data/acme.json"
      # Utilise le serveur de staging pour tester (évite le ban), commente pour la prod
      # - "--certificatesresolvers.le.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"

    # Persistance du fichier acme.json (pour ne pas perdre les certs au reboot)
    persistence:
      enabled: true
      path: /data
      size: 128Mi
```

## roles/runner/tasks/main.yml

```yaml
---
- name: Création du dossier temporaire pour les manifestes
  file:
    path: /tmp/k8s-runner
    state: directory
    mode: '0700'

- name: Génération du manifeste Runner (avec Token)
  template:
    src: deployment.yaml.j2
    dest: /tmp/k8s-runner/deployment.yaml
    mode: '0600'

- name: Application du Runner dans le Cluster
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    src: /tmp/k8s-runner/deployment.yaml

- name: Nettoyage du fichier contenant le token
  file:
    path: /tmp/k8s-runner
    state: absent
```

## roles/runner/templates/deployment.yaml.j2

```text
# roles/runner/templates/deployment.yaml.j2
apiVersion: v1
kind: Namespace
metadata:
  name: actions-runner-system

---
# 1. Identité du Runner (ServiceAccount)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-runner-sa
  namespace: actions-runner-system

---
# 2. Permissions (Admin du Cluster)
# Nécessaire pour que le runner puisse exécuter "kubectl apply" partout
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deploy-runner-admin
subjects:
  - kind: ServiceAccount
    name: deploy-runner-sa
    namespace: actions-runner-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

---
# 3. Le Runner (Deployment)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-runner-klaro
  namespace: actions-runner-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-runner-klaro
  template:
    metadata:
      labels:
        app: github-runner-klaro
    spec:
      serviceAccountName: deploy-runner-sa
      containers:
      - name: runner
        # Ton image Hardened créée précédemment
        image: spadmdck/k8s-deploy-runner:v1
        imagePullPolicy: Always
        
        # Variables d'environnement pour l'enregistrement
        env:
          - name: GITHUB_REPOSITORY
            value: "sicDANGBE/klaro"
          - name: GITHUB_TOKEN
            value: "{{ github_token }}" # Injecté par Ansible
            
        # Ressources limitées pour éviter qu'un build ne tue le cluster
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "2Gi"

        # Script de démarrage (Identique à celui testé manuellement)
        command: ["/bin/bash", "-c"]
        args:
          - |
            # Configuration unique (si pas déjà configuré)
            if [ ! -f .runner ]; then
              ./config.sh --url https://github.com/saasMsDGH --token ${GITHUB_TOKEN} --unattended --replace --name org-runner-01 --labels k8s-deploy
            fi
            # Lancement
            ./run.sh
```

## roles/swarm/tasks/main.yml

```yaml
---
# -------------------------------------------------------
# ETAPE 1 : MANAGER
# -------------------------------------------------------

- name: Initialisation du Swarm (Manager)
  community.docker.docker_swarm:
    state: present
    advertise_addr: "{{ ansible_default_ipv4.address }}"
  register: swarm_init_result
  when: "'manager' in group_names"  # <--- CIBLAGE PAR GROUPE

- name: Récupération des Infos du Cluster (Token)
  community.docker.docker_swarm_info:
  register: swarm_facts
  delegate_to: "{{ groups['manager'][0] }}"
  run_once: true

# -------------------------------------------------------
# ETAPE 2 : PARE-FEU
# -------------------------------------------------------
# (Pas de changement ici, ça s'applique à tout le monde)
- name: Déploiement règles NFTables Swarm
  template:
    src: nftables_swarm.conf.j2
    dest: /etc/nftables.d/10-swarm.conf
    mode: '0644'
  notify: Reload NFTables

- name: Application immédiate du Pare-feu
  meta: flush_handlers

# -------------------------------------------------------
# ETAPE 3 : WORKERS - JONCTION ET NETTOYAGE
# -------------------------------------------------------

- name: Vérification état Swarm local du Worker
  community.docker.docker_host_info:
  register: worker_info
  when: "'workers' in group_names" # <--- CIBLAGE PAR GROUPE

- name: Quitter le mauvais Swarm
  command: docker swarm leave --force
  when:
    - "'workers' in group_names"
    - worker_info.host_info.Swarm.LocalNodeState == 'active'
    - worker_info.host_info.Swarm.Cluster.ID is defined
    - worker_info.host_info.Swarm.Cluster.ID != swarm_facts.swarm_facts.ID
  notify: Restart Docker

- name: Force restart Docker si nettoyage
  meta: flush_handlers

- name: Rejoindre le Swarm
  community.docker.docker_swarm:
    state: join
    join_token: "{{ swarm_facts.swarm_facts.JoinTokens.Worker }}"
    remote_addrs: [ "{{ hostvars[groups['manager'][0]]['ansible_default_ipv4']['address'] }}:2377" ]
  when:
    - "'workers' in group_names"
    - (worker_info.host_info.Swarm.LocalNodeState != 'active') or
      (worker_info.host_info.Swarm.Cluster.ID is defined and worker_info.host_info.Swarm.Cluster.ID != swarm_facts.swarm_facts.ID)

# -------------------------------------------------------
# ETAPE 6 : RÉSEAU GLOBAL (INFRASTRUCTURE)
# -------------------------------------------------------
- name: Création du réseau overlay 'traefik-public'
  community.docker.docker_network:
    name: traefik-public
    driver: overlay
    attachable: true # Important pour le debugging ou les conteneurs hors-stack
    driver_options:
      # encrypted: "yes"
      # FIX MTU CRITIQUE (OVH/OpenStack VXLAN)
      com.docker.network.driver.mtu: "1300"
  when: "'manager' in group_names"
  run_once: true

- name: Vérification MTU docker_gwbridge (Ingress)
  shell: docker network inspect docker_gwbridge --format '{{ '{{' }} index .Options "com.docker.network.driver.mtu" {{ '}}' }}'
  register: gwbridge_mtu
  ignore_errors: true
  changed_when: false
  when: "'manager' in group_names"

# # Cette tâche est un peu "brutale" mais nécessaire si le gwbridge est en 1500
# # Elle ne se lancera que si le MTU est mauvais.
# - name: Force Recréation docker_gwbridge et Ingress en MTU 1300
#   shell: |
#     # 1. On supprime d'abord le réseau Ingress (qui verrouille le gwbridge)
#     # Le "|| true" permet de continuer si le réseau n'existe pas ou bug
#     docker network rm ingress || true
    
#     # 2. Maintenant on peut supprimer le gwbridge
#     docker network rm docker_gwbridge || true
    
#     # 3. On recrée le gwbridge avec le bon MTU (1300)
#     docker network create \
#       --subnet 172.18.0.0/16 \
#       --gateway 172.18.0.1 \
#       --opt com.docker.network.bridge.name=docker_gwbridge \
#       --opt com.docker.network.bridge.enable_icc=false \
#       --opt com.docker.network.driver.mtu=1300 \
#       docker_gwbridge
      
#     # 4. CRITIQUE : On recrée immédiatement le réseau Ingress
#     # Sinon tes services ne pourront plus publier de ports
#     docker network create \
#       --driver overlay \
#       --ingress \
#       --opt com.docker.network.driver.mtu=1300 \
#       ingress
#   when: 
#     - "'manager' in group_names"
#     - gwbridge_mtu.stdout != "1300"
#     # On enlève la condition rc==0 pour forcer si besoin, le script gère les erreurs via || true
```

## roles/swarm/templates/nftables_swarm.conf.j2

```text
# Règles spécifiques Swarm
# Injectées dans la table "inet filter" définie dans nftables.conf

# Définition de la liste des IPs du cluster
define SWARM_NODES = { {{ groups['swarm_nodes'] | map('extract', hostvars, ['ansible_default_ipv4', 'address']) | join(', ') }} }

# Ajout à la chaîne INPUT existante
add rule inet filter input ip saddr $SWARM_NODES tcp dport { 2377, 7946 } accept
add rule inet filter input ip saddr $SWARM_NODES udp dport { 7946, 4789 } accept
add rule inet filter input ip protocol esp accept

# Ajout à la chaîne FORWARD existante (Pour les conteneurs et l'Overlay)
add rule inet filter forward iifname "docker0" accept
add rule inet filter forward oifname "docker0" accept
add rule inet filter forward iifname "docker_gwbridge" accept
add rule inet filter forward oifname "docker_gwbridge" accept

# Autoriser le trafic INTER-CONTENEURS sur les réseaux custom (Stacks)
# Docker nomme ces interfaces "br-<id_reseau>"
add rule inet filter forward iifname "br-*" accept
add rule inet filter forward oifname "br-*" accept

# Web Public (Uniquement sur le Manager)
{% if inventory_hostname == manager_host %}
add rule inet filter input tcp dport { 80, 443 } accept
{% endif %}
```

## site.yml

```yaml
---
# -----------------------------------------------------------------------
# PLAY 1 : INFRASTRUCTURE BASE & SWARM (Legacy / Nettoyage)
# Cible : swarm_nodes (Tous les serveurs)
# -----------------------------------------------------------------------
- name: "Déploiement Infrastructure Swarm & Système"
  hosts: swarm_nodes
  become: true
  
  # Les handlers sont propres à ce Play
  handlers:
    - name: Restart Rsyslog
      service: name=rsyslog state=restarted

    - name: Reload NFTables
      service: name=nftables state=reloaded
      
    - name: Restart Docker
      service: name=docker state=restarted
      
    - name: Restart SSH
      service: name=ssh state=restarted

  roles:
    # 1. Nettoyage (A lancer AVANT d'installer k8s)
    # Commande : ansible-playbook site.yml --tags "clean"
    - role: clean
      tags: ["clean"]

    # 2. Configuration Système (Utilisateurs, Firewall Base, Fail2Ban)
    # Commande : ansible-playbook site.yml --tags "common"
    - role: common
      tags: ["common", "system"]

    # 3. Moteur Docker (Requis pour Swarm, optionnel pour k8s qui utilise containerd)
    # Commande : ansible-playbook site.yml --tags "docker"
    - role: docker
      tags: ["docker", "engine"]

    # 4. Orchestration Swarm (Incompatible avec k8s sur le même nœud)
    # Commande : ansible-playbook site.yml --tags "swarm"
    - role: swarm
      tags: ["swarm", "network"]

# -----------------------------------------------------------------------
# PLAY 2 : APPLICATIONS SWARM (Legacy)
# Cible : Manager uniquement
# -----------------------------------------------------------------------
- name: "Déploiement Applications (Swarm Stacks)"
  hosts: ovh.core
  become: true
  tags: ["apps", "deploy"]
  
  handlers:
    - name: Reload NFTables
      service: name=nftables state=reloaded
      
  roles:
    - apps

# -----------------------------------------------------------------------
# PLAY 3 : INFRASTRUCTURE KUBERNETES (k8s) - FUTUR
# Cible : k8s_cluster (Server + Agents)
# -----------------------------------------------------------------------
- name: "Déploiement Cluster Kubernetes (k8s)"
  hosts: k8s_cluster
  become: true
  tags: ["k8s", "kubernetes"]
  
  # Handlers spécifiques au contexte k8s
  handlers:
    - name: Reload NFTables
      service: name=nftables state=reloaded

  roles:
    # Installe le Master puis les Workers
    # Commande : ansible-playbook site.yml --tags "k8s"
    - k8s

# -----------------------------------------------------------------------
# PLAY 4 : APPS KUBERNETES (HELM)
# Cible : server (UNIQUEMENT LE MANAGER)
# -----------------------------------------------------------------------
- name: "Déploiement Applications Kubernetes"
  hosts: server  # <--- CHANGEMENT ICI (Avant c'était k8s_cluster ou non défini)
  become: true
  tags: ["app_k8s"]
  
  roles:
    - app_k8s

# -----------------------------------------------------------------------
# PLAY 5 : RUNNER KUBERNETES (HELM)
# Cible : server (UNIQUEMENT LE MANAGER)
# -----------------------------------------------------------------------
- name: "Déploiement Runners Kubernetes"
  hosts: server  # <--- CHANGEMENT ICI (Avant c'était k8s_cluster ou non défini)
  become: true
  tags: ["k8s_runner"]
  
  roles:
    - runner


```

