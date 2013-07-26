Installation
-------------

Symlink the script to a location visible by `PATH`. I have mine set to `~/bin`.

An optional global variable `GIT_CACHED_DIR` can be set. This is where the cached objects are stored. The default is shown below.

```bash
export PATH=$HOME/bin:$PATH
export GIT_CACHED_DIR=$HOME/.gitobjectstore
```

Alternative installation: Replacing "git" altogether
----------------------------------------------------

You can also replace the `git` command globally, which is very helpful f.e. when working with (legacy) tools
and scripts which are hard-coded to use the `git` binary. This f.e. applies to [composer](http://getcomposer.org/) (which is a PHP package management solution).

You need to rename `gitc` to `git` and place it on your `$PATH` such that it is loaded *first*, i.e. before
the normal git (f.e. in `~/bin`). Furthermore, you need to set the environment variable `PATH_TO_GITC` such
that it points to the original `git` command.

So, in a nutshell:
```bash
mkdir -p ~/bin/

cp /path/to/gitc ~/bin/git
```

Afterwards, add the following to your `~/.profile` or `~/.zshrc` or `~/.bashrc` (depending on what you use):

```bash
##### GITC Git Cache #####
export PATH=~/bin:$PATH
export GITC_ORIGINAL_GIT=/path/to/original/git
alias git-uncached=$GITC_ORIGINAL_GIT
```


Usage
------

`gitc` accepts all `git` commands. For the ones it does not know of, it will simply pass-through to git.

When cloning with `gitc`, it will transparently use the cache.

```bash
gitc clone --branch 8.x git://git.drupal.org/project/drupal.git
```

If at anytime you need to remove all reference to the cache (runs `git repack -a -d -l` and removes pointer in the alternates file.:

```bash
gitc cache-detach
```

It can be added back at any time (runs `git gc` garbage collection to remove local objects):

```bash
gitc cache-attach
```

To repair it (it simply detaches and reattaches):

```bash
gitc cache-repair
```

If you need to run git commands on the cache store itself, use the `cache` sub-command:

```bash
gitc cache remote
```

When cloning, attaching or repairing, you can pass in a `--store-group` option to tell it to use a separate object store. It defaults to `default`.

```bash
gitc clone --store-group foobar --branch 8.x git://git.drupal.org/project/drupal.git
```

The above command will point the object store to `$HOME/.gitobjectstore/foobar`. (defaults to `$HOME/.gitobjectstore/default`)

Notes
------

  - The cached clone is set with `--reference` pointing to the object storage. See the man page for git-clone.
  - If you move the clone to another machine it may invalidate the path pointing to the object store. Changing `GIT_CACHE_DIR` will do the same. Simply call `gitc cache-repair` in your working directory to fix it.
  - This is the first bash script I’ve created. Consider this a fair warning. Tested on Mac OSX Lion.
  - I personally don't use this for critical work. I'm using this to reduce the bandwidth and disk space on multiple versions of checked out repos that are mostly read only. It does work fine from what I've tested.
  - This new development version is incompatible with the master branch.


Internals
---------

(reverse engineered)

*clone*

- split apart
- # 1 addr - full address
  # | 2 prot - protocol, e.g. http://, git://, etc...
  # | | 3 user - the user in user@git.drupal.org[...]
  # | | | 4 doma - domain, e.g. git.drupal.org
  # | | | | 5 port - port number
  # | | | | | 6 path - path after domain
  # | | | | | | 7 proj - project name
- only do it if DOMAIN is set (becuase then it's not local)
- initialize SHARED cache directory if it does not exist
	- cd ~/.gtiobjects/default
	- git init --bare
	- git config --add gc.pruneexpire never
	- (+ error handling; on failures remove it again)
- go INTO cache directory
- initialize clone if it does not exist (this is determined by checking the output of "git remote" and comparing it with the PROJECT)
	- git remote (lists something like)
		BuildEssentials
		Frontend
		PlusPunktHosting
		ProductManagement
		TYPO3.Aloha
		TYPO3.Eel
		TYPO3.Expose
		TYPO3.ExtJS
		TYPO3.Flow
		TYPO3.Fluid
		TYPO3.Form
		TYPO3.Imagine
		TYPO3.Kickstart
		TYPO3.Media
		TYPO3.Neos
		TYPO3.Neos.NodeTypes
		TYPO3.Party
		TYPO3.Setup
		TYPO3.TYPO3CR
		TYPO3.Twitter.Bootstrap
		TYPO3.TypoScript
		migrations
	- TODO error: if two different PROJECTS are named BuildEssentials but are not the same --> error!!!!
	- (create temporary directory, and clone using git clone --bare (adress) (tempdir))
	- inside git cache: add remote "PROJECTNAME" (tempdir of above)

	- git config --add remote.[PROJECTNAME].tagopt --no-tags
	  # Add default configuration for the project.
      # Do not download tags due to the overlaps across projects.

	- # Update with temporary clone then point remote URL externally; and fetch again
      echo "Creating cache: $doma >> $proj..."
      git remote update [PROJECTNAME] # (fetches)
      $GITC_ORIGINAL_GIT remote set-url $proj $addr
      $GITC_ORIGINAL_GIT fetch $proj
    - (TEMP DIR NOT NEEDED ANYMORE)
    - ---------------------------------------- INITIALIZE FINISHED
    -  git clone (original URL) --reference (GIT CACHE DIR) --config git-cached.project=PROJECT --config git-cached.domain=DOMAIN --config git-cached.cache-dir=CACHE_DIR
    -  !!!! MAGIC SAUCE: "REFERENCE" parameter (see git help); allows to re-use existing OBJECTS
    - --> only an OBJECT store, that's why we do not need any tags while initializing this repository.


 *git fetch / git pull*
- go to cache directory (which we must know from local git config)
- (in cache directory) git fetch [PROJECTNAME] -- updates the cache.
- -> then, run GIT AS USUAL!



*Helpers: not needed for us*


