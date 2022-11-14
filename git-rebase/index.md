---
theme: uncover
class: invert
size: 16:9
---

<span style="text-shadow: 2px 2px 0px darkgreen;">git rebase workshop</span>

![bg contain](./cover.png)

----

agenda

1. how does merge work? (you probably know this)
2. how does rebase work?
2. how does _interactive_ rebase work?

----

git config

```gitconfig
[alias]
    graph  = log --graph --oneline --boundary
    please = push --force-with-lease
```

----

example repository state

https://github.com/tstelzer/example-git-repository

```shell
$ ls
a.txt  b.txt

$ git graph

* 16924da  (HEAD -> refs/heads/master) feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

branching off and making a change

```shell
$ git checkout -b feature/with-merge

$ echo "Merging is awesome!" >> a.txt

$ git commit -am "feat: update a"

$ git graph

* 2a6d066  (HEAD -> refs/heads/feature/with-merge) feat: update a
* 16924da  (refs/heads/master) feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

making a parallel change on master

```shell
$ git checkout master

$ echo "Just doing my work." >> a.txt

$ git commit -am "feat: diverge!"

$ git graph --all

* f037f15  (HEAD -> refs/heads/master) feat: diverge!
| * 08429a7  (refs/heads/feature/with-merge) feat: update a
|/
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

merging `master` on top of our feature branch

```shell
$ git checkout feature/with-merge

$ git merge master

Auto-merging a.txt
CONFLICT (content): Merge conflict in a.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Ugh, let's fix the conflict.

----

post merge

```shell
$ git graph

*   0320e01  (HEAD -> refs/heads/feature/with-merge) Merge branch 'master' into feature/with-merge
|\
| * f037f15  (refs/heads/master) feat: diverge!
* | 08429a7  feat: update a
|/
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

undoing a merge

```shell
$ git merge --abort      # during merge conflict
$ git reset --hard HEAD~ # undoing merge commit
```

----

preparing another branch (for rebasing)

```shell
$ git checkout master
$ git checkout -b feature/with-rebase 16924da # <- commit prior to master change

$ echo "Rebasing is great." >> a.txt

$ git commit -am "feat: update a"

$ git graph

* 522eb9e  (HEAD -> refs/heads/feature/with-rebase) feat: update a
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

rebasing!

```shell
$ git rebase master

Auto-merging a.txt
CONFLICT (content): Merge conflict in a.txt
error: could not apply a7c0aea... feat: update a
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply a7c0aea... feat: update a
```

Again, fixing the conflict.

----

post rebase (compared with merge)

```shell
$ git graph

* 4b1f195  (HEAD -> refs/heads/feature/with-rebase) feat: update a
* 7dc6cbc  (refs/heads/master) feat: diverge!
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root

$ git graph feature/with-merge
*   0320e01  (refs/heads/feature/with-merge) Merge branch 'master' into feature/with-merge
| \
| * 7dc6cbc  (refs/heads/master) feat: diverge!
* | 2a6d066  feat: update a
|/
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

you rewrote history, so you need to force-push

```shell
$ git push --force-with-lease
$ git please # <- or use the alias :)
```

<span style="color: red;">never force-push to public branches</span> :skull:

----

undoing a rebase

```shell
$ git rebase --abort         # during rebase
$ git reset --hard ORIG_HEAD # undoing rebase
```

----

when you've fucked up (not just for rebase)

```shell
$ git reflog # output shortend for readability

4cca3ca (HEAD -> refs/heads/feature/with-rebase) HEAD@{0}: rebase (continue) (finish): returning to refs/heads/feature/with-rebase
4cca3ca (HEAD -> refs/heads/feature/with-rebase) HEAD@{1}: rebase (continue): feat: update a
7dc6cbc (refs/heads/master) HEAD@{2}: rebase (start): checkout master
522eb9e HEAD@{3}: commit: feat: update a

$ git reset --hard HEAD@{3}

$ git graph

* 522eb9e  (HEAD -> refs/heads/feature/with-rebase) feat: update a
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

----

interactive rebase (for completeness)

```shell
$ echo "Some random change." >> a.txt
$ git commit -am "feat: some minor change"

$ echo "Something related to the first commit." >> b.txt
$ git commit -am "feat: sorta belongs into the other commit"

$ git graph

* e134ec3  (HEAD -> refs/heads/feature/with-rebase) feat: sorta belongs into the other commit
* e7e0eef  feat: some minor change
* bf3aa50  feat: update a
* d62b7dd  (refs/remotes/origin/master, refs/heads/master) feat: diverge!
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root

$ git rebase -i e7e0eef~            # the ~ sorta means "before", and will include this commit
$ git rebase --interactive bf3aa50  # same thing as above
```

----

change

```sh
pick e7e0eef feat: some minor change
pick e134ec3 feat: sorta belongs into the other commit

# Rebase bf3aa50..e134ec3 onto bf3aa50 (2 commands)
```

to

```sh
pick e7e0eef feat: some minor change
fixup e134ec3 feat: sorta belongs into the other commit

# Rebase bf3aa50..e134ec3 onto bf3aa50 (2 commands)
```

save and quit

----

post rebase

```sh
* aa05ba5  (HEAD -> refs/heads/feature/with-rebase) feat: some minor change
* bf3aa50  feat: update a
* d62b7dd  (refs/remotes/origin/master, refs/heads/master) feat: diverge!
* 16924da  feat: add b
* 5533465  feat: add a
* 0fc0681  root
```

we've merged the two commits into one

----

| merging                             | rebasing                        |
|:------------------------------------|:--------------------------------|
| non-destructive                     | rewrites history                |
| harder to read (non-linear history) | easier to read (linear history) |
| clear collaboration boundaries      | no boundaries                   |

----

when to use what?

- `git merge` for public branches (e.g. PRs)
- `git rebase` for pulling updates into local branch
- `git rebase -i` for cleaning up local branch

----

Thanks For Listening!

Source for slides at [github.com/tstelzer/presentation](https://github.com/tstelzer/presentation)

