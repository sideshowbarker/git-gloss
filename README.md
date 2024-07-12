## git-gloss ✨ makes git logs show PR/issue/review links

`git-gloss` automatically adds [git notes](https://scottchacon.com/2010/08/25/notes/) to all your git logs — with GitHub PR/issue/reviewer/author links.

---

### How to use git-gloss

You can download and run `git-gloss` within a directory having a GitHub repo clone by just doing this:

```
curl -fsSLO https://sideshowbarker.github.io/git-gloss/git-gloss && bash ./git-gloss
```

That will add [git notes](https://scottchacon.com/2010/08/25/notes/) locally for all commits in the local commit history with an associated GitHub pull request.

Then, when you run `git log`, the log output for each commit will look something like this:

```
commit 9812031a02e539f08a6936e9c17d919a44c912b8
Author: Jonatan Klemets
Date:   Sun Jul 23 19:38:04 2023 +0300

    LibWeb: Implement spec-compliant integer parsing

    This patch adds two new methods named `parse_non_negative_integer` and
    `parse_integer` inside the `Web::HTML` namespace that uses `StringUtils`
    under the hood but adds a bit more logic to make it spec compliant.

Notes:
    Author: https://github.com/Jon4t4n 🔰
    Commit: https://github.com/SerenityOS/serenity/commit/9812031a02
    Pull-request: https://github.com/SerenityOS/serenity/pull/20140
    Issue: https://github.com/SerenityOS/serenity/issues/19937
    Reviewed-by: https://github.com/AtkinsSJ ✅
    Reviewed-by: https://github.com/nico
```

🔰 – indicates this is author’s first commit to the repo\
✅ – indicates a review approval

You can also run `git-gloss` on any subset of a rep’s commit history — by giving it zero or more commit hashes:

```
./git-gloss 9812031a02 ebc5b33b77a 418f9ceadd
```

---

> [!TIP]
> If you want to put notes in the logs for multiple repos, see the [Add a “git gloss” command](#add-git-gloss-command) section for a how-to on setting up a new `git gloss` command that you can run just as you would any other `git` command.

---

### How to share the notes

Once `git-gloss` finishes running, here’s how you can share the notes with everyone in your GitHub project:

1. Push the notes back to your project remote at GitHub by running this command:

   ```
   git push origin 'refs/notes/*'
   ```

2. Others in your project can then fetch the notes from GitHub by running this command:

   ```
   git fetch origin 'refs/notes/*:refs/notes/*'
   ```

   Alternatively, rather than running the above command manually, others in the project can update their git configuration by running the following command;

   ```
   git config --add remote.origin.fetch '+refs/notes/*:refs/notes/*'
   ```

   That will cause all notes to be fetched from the remote every time they use `git fetch` or `git pull`.

3. Run `git-gloss` again to add notes for any new commits made after the last time you ran `git-gloss`.

4. Keep your project’s notes up to date by repeating steps 1 to 3 at a regular cadence (e.g., once day or so).

   Alternatively — to have steps 3 and 4 get done for you automatically, every time anyone from the project pushes to the main branch — you can set up a GitHub Actions workflow, with the file contents like this:

   ```yml
   name: Push notes
   on:
     push:
       branches:
         - master
   permissions:
     contents: write
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0
         - uses: fregante/setup-git-user@v2
         - run: |
             git fetch origin "refs/notes/*:refs/notes/*"
             curl -fsSLO https://sideshowbarker.github.io/git-gloss/git-gloss && bash ./git-gloss
             git push origin "refs/notes/*"
           env:
             GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
   ```

> [!IMPORTANT]
> It’s especially important that your workflow file include the following:
>
> ```
> with:
>  fetch-depth: 0
> ```
>
> …as shown in the [Fetch all history for all tags and branches](https://github.com/actions/checkout?tab=readme-ov-file#fetch-all-history-for-all-tags-and-branches) example in the actions/checkout documentation.
>
> `git-gloss` needs access to the entire commit history — and that requires setting `fetch-depth: 0`. Otherwise, without that set, the actions/checkout action fetches only 1 commit from the history.
>
> Among the problems with fetching only one commit: If your project uses the [Rebase and merge](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github#rebasing-and-merging-your-commits) method, it’s possible that each push to the repo may contain multiple commits — and if so, without `fetch-depth: 0` set, only one of the commits in the push would get processed by `git-gloss`, and the rest would be ignored.

---

### How long does it take?

`git-gloss` seems able to process at most about 1000 commits per hour — about 17 or 18 commits per minute.

So, the first time you run it in a repo with many commits, it’ll take a long time — hours, or even a day or more.

For example, if your repo has somewhere around 1000 commits, it’ll take at least 1 hour to finish. If your repo has somewhere around 10,000 commits, it’ll take more than 10 hours. And so on.

> [!NOTE]
> You can stop `git-gloss` at any time with <kbd>Ctrl</kbd>-<kbd>C</kbd>. After stopping it, when you run it again, it will start off wherever it left off. So, if you have a repo with somewhere around 2000 commits, and you stopped `git-gloss` after it was running for about an hour, then it will run for about another hour before it finishes.

### How to “back up” notes

Given how long (multiple hours) it can take `git-gloss` to add all notes for a large history, you should do this:

1. Periodically stop `git-gloss` (say, once an hour), using <kbd>Ctrl</kbd>-<kbd>C</kbd>.

2. Run the following command to push your (partial) notes to your project repo:

   ```
   git push origin 'refs/notes/*'
   ```

3. Restart `git-gloss`, to continue adding more notes.

4. Repeat steps 1 to 3 periodically (say, once an hour) until `git-gloss` finishes.

> [!IMPORTANT]
> Doing the steps above will ensure that you have a “backup” of the notes you’ve generated so far — and if ever needed, you can then “restore” your notes from that backup by running the following command:
>
> ```
> git fetch origin 'refs/notes/*:refs/notes/*'
> ```

### Why is it so slow?

For each commit `git-gloss` processes, it makes 4 calls to GitHub API endpoints — requiring network resources and time. And in a typical environment, for each commit the total time needed (mostly due to those calls) seems to to work out to at least 3.5 seconds or so — which means it can process only about 17 or 18 commits a minute.

Regardless, the GitHub API has [rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) that prevent making more than 5000 requests per hour — which works out to about 83 requests per minute. And so, because `git-gloss` makes 4 requests for each commit, that also limits it to being able to make only enough requests per minute for 20–21 commits at most (20 ⨉ 4 ⨉ 60 = 4800).

### How to handle problems

 It’s possible to end up making enough requests in your environment that you exceed the 5000-requests-per-hour limit, especially if you happen to be running other applications that make GitHub API calls at the same time you’re running `git-gloss`. When you do exceed the limit while running `git-gloss`, you’ll see it log a message like this:

> ⚠️  ***Attention: Hit the GitHub API rate limit. Sleeping for 933 seconds.***

 If you do see that message, you don’t need to worry; instead, `git-gloss` will automatically wait for the specified number of seconds — and then, after that, it will start sending GitHub API requests once again.

`git-gloss` tries to gracefully handle all errors, and you may occasionally see it log a message like this:

> ⚠️  ***Attention: Failed to fetch issues from GitHub; re-trying.***

If you do see that message, you don’t need to worry; instead, `git-gloss` will automatically retry the GitHub API request — and will keep re-trying it until it succeeds.

But it’s possible you may run into other errors — unhandled errors. So you can keep an output log, and review it after `git-gloss` finishes. You can create a `git-gloss` output log like this:

```
tmpfile=$(mktemp) && echo "Logging output to $tmpfile";
git-gloss 2>&1 | tee tmpfile
```

That creates a log file and sends the output to both the terminal and the log file (using the `mktemp` and `tee` utilities, which are standard in any Linux/Unix environment — including the macOS Terminal/shell environment).

After `git-log` finishes (or even as it’s running), you can review the log for any unhandled error output.

> [!NOTE]
> You should [raise an issue](https://github.com/sideshowbarker/git-gloss/issues/new) if you do find any unhandled errors.

For each unhandled error you find, you’ll need to complete the following steps:

1. Remove any note which `git-gloss` may have added for the given commit:

   ```
   git notes remove 67c727177e
   ```

2. (Re)add a note for the given commit, by running `git-gloss` with the commit hash specified:

   ```
   git-gloss 67c727177e
   ```

> [!CAUTION]
> `git-gloss` provides no way to undo its actions and remove all notes it added. The only practical way to undo its actions may be to completely remove all notes, including any you may have added by other means.
>
> But if you haven’t added notes by any other means: To remove all `git-gloss`-added notes, run this:
>
> ```
> git update-ref -d refs/notes/commits
> ```

### How to fix “loose objects”

After running `git-gloss` for some time, if you then do other git operations in the same clone where you’ve got `git-gloss` running, you may see git reporting the following series of messages:

```
Auto packing the repository in background for optimum performance.
See "git help gc" for manual housekeeping.
warning: The last gc run reported the following. Please correct the root cause
and remove .git/gc.log
Automatic cleanup will not be performed until the file is removed.

warning: There are too many unreachable loose objects; run 'git prune' to remove them.
```

> [!CAUTION]
> **Absolutely never, ever, under any circumstances run `git prune` at the same time `git gloss` is running.**
>
> Invoking `git prune` while `git-gloss` is running may cause corruption of whatever notes data you’ve generated so far — and may even bork your entire local git environment for the clone in such a way as to prevent you from being able to successfully run any other git commands at all.

If the reason you’re reading this section is that you *did* run `git prune` — and you’re now trying to figure out how what to do — then: **don’t worry**, because you *can* recover from it. Here’s how → Run the following command:

> ```
> git update-ref -d refs/notes/commits
> ```

That will remove all the notes you’ve added *locally* so far. But if you’ve periodically been pushing [backups](#how-to-back-up-notes), then you can next do this:

> ```
> git fetch origin 'refs/notes/*:refs/notes/*'
> ```

…and that *may* successfully recover all the notes you had added up to the point where you last backed up.

Regardless, your next step is just [re-run `git-gloss`](#how-to-use-git-gloss) so that it can again start adding notes.

But note that having too many loose objects in a repo is not really a problem that has any serious effects; it’s not something to spend any time worrying about.

So the only real “problem” to fix here is: How to make git stop emitting that series of warning messages.

And the way to do that is just this:

**Once `git gloss` has completely finished**, *then at that time* you can safely run `git prune` (and you should).

But otherwise, prior to `git gloss` completely finishing, **do not spend any time worrying about**; having too many loose objects in a repo for a while (or even for a long time) doesn’t cause any real problems worth worrying about.

### Dependencies

* `git` – any version ([v2.42+](https://stackoverflow.com/a/76633969/)) with support for the `--no-separator` option for the `git notes` command

* `jq` – [https://jqlang.github.io/jq/](https://jqlang.github.io/jq/) (JSON processor)

* `gh` – [https://cli.github.com/](https://cli.github.com/) (GitHub CLI), with the [`GH_TOKEN` or `GITHUB_TOKEN`](https://cli.github.com/manual/gh_help_environment) environment variables set

* `grep` – any grep-compatible program

> [!IMPORTANT]
> On macOS in particular, [GNU grep](https://apple.stackexchange.com/a/193300) — rather than than the Apple-provided `grep` — is recommended,for performance reasons; example:
>
> ```
> brew install grep
> ```

### Environment variables

You can affect the `git-gloss` behavior using the environment variables described in this section.

> [!TIP]
> Rather than separately exporting each environment variable to your shell, you can instead specify them all at the same time in the invocation you use for running `git-gloss` — like this:
>
> ```
> GIT=/opt/homebrew/bin/git GREP=/opt/homebrew/bin/ggrep \
>     OTHER_REPO=SerenityOS/serenity >    ./git-gloss
>```

#### `GIT`

You can use this to specify a path to a different `git` binary — for instance, in the case where you have multiple different `git` versions on your system; example:

```
export GIT=/opt/homebrew/bin/git
```

> [!NOTE]
> Because `git-gloss` calls the `git notes` command with the `--no-separator` option — which was added in git [version 2.42+](https://stackoverflow.com/a/76633969/) — the git version you use with `git-gloss` must be version 2.42 or later.

#### `GREP`

You can use this to specify a path to any grep-compatible binary on your system; for instance, to avoid using the Apple-provided `grep` on macOS; example:

```
export GREP=/opt/homebrew/bin/ggrep
```

> [!IMPORTANT]
> On macOS in particular, [GNU grep](https://apple.stackexchange.com/a/193300) — rather than than the Apple-provided `grep` — is recommended, for performance reasons; example:
>
> ```
> brew install grep
> ```

#### `OTHER_REPO`

You can use this to specify an `[owner]/[repo]` repo other than the current repo; e.g., a repo the current repo shares part of its commit history with (because the current repo was created from an older repo); example:

```
export OTHER_REPO=SerenityOS/serenity
```

If you specify an `OTHER_REPO` value, then if `git-gloss` can’t find any pull request for a particular commit in the current repo, it will then look for a pull request in the repo you specified in the `OTHER_REPO` value.

### Add “git gloss” command

Clone the `git-gloss` repo and add its directory to your `$PATH`:

  ```bash
  git clone https://github.com/sideshowbarker/git-gloss.git
  cd git-gloss
  echo export PATH=\"$PATH:$PWD\" >> ~/.bash_profile
  ```

  Now you can just type `git gloss` in any repo/clone directory, to add notes to the logs for that repo.

### Notes

You can see how much space your notes tree is taking up by running this command:

```
git ls-tree -r $(git rev-parse refs/notes/commits) \
    | awk '{print $3}' | git cat-file --batch-check='%(objectsize:disk)' \
    | awk '{s+=$1} END {printf "%.2f MB\n", s / 1048576}'
```

You’re likely to find that it takes up about 1MB to 1.3MB for every 20,000 commits in the repo history.
