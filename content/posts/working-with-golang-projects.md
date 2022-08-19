---
title: "Working With Open Source Golang Projects"
date: 2022-08-19T10:27:32-03:00
draft: false
---

I have been working with golang projects for quite some time and I found that the usual fork and pull request on github is a bit trickier when working with golang, as golang doesn't have dependency management tools like python has PyPI or npm for NodeJS, etc. Golang dependency management relies on pretty much git tag or commit hashed to specific repos for using golang modules in your code, in my personal opinion I have mixed feelings about the approach, however I am not going to discuss that here, I might do that in another post. However when contributing to open source projects I found myself in trouble until I understood how things worked in managing dependencies in golang.  

In the rest of this post I will not cover installation of tools used, which are git cli client, github cli client and golang and any other dependency needed to build the open source project chosen for the post.


# Problem statement

Let's take a well known golang open source software called [prometheus alertmanager](https://github.com/prometheus/alertmanager). If you want to build it from source you just do the following.

```bash
# Example building prometheus alert manager from source.
$ git clone https://github.com/prometheus/alertmanager.git
$ cd alertmanager
$ make build
```

Building it was pretty easy and straightforward.  
However if we want to modify the source code and then follow the project guidelines for contributing, we probably will need to issue a Pull Request from our own fork of the github repository.  
So the first step would be to fork the project into our own github account, I will be using github cli tool (not covered how to authenticate the cli tool in this post).

```bash
$ gh repo fork https://github.com/prometheus/alertmanager.git 
✓ Created fork fcastello/alertmanager
? Would you like to clone the fork? No
```
So, we created a fork of the upstream repository into my own (fcastello) github account using the github cli tool, I chose not clone the fork because we are not going to be using it, but why? and to see that we need to see a piece of code from the alertmanager main package

```go

package main

import (
	"context"
	...

	"github.com/prometheus/alertmanager/api"
	"github.com/prometheus/alertmanager/cluster"
	"github.com/prometheus/alertmanager/config"
	"github.com/prometheus/alertmanager/dispatch"
	"github.com/prometheus/alertmanager/inhibit"
    
```
As we can see in the imports alertmanager have references to multiple dependencies within the same repo like `github.com/prometheus/alertmanager/api` or `github.com/prometheus/alertmanager/cluster` which depends on the github URL of the alertmanager project.  

So if we build it with our fork
```bash
# Example building prometheus alert manager from source.
$ git clone https://github.com/fcastello/alertmanager.git
$ cd alertmanager
$ make build
```
This will probably build but if we modified files that are in for example `github.com/fcastello/alertmanager/api` then you might find that it's still using the alertmanager upstream code and not the one you just modified in your code. Why? because now the files you are modifying are in `github.com/fcastello/alertmanager/api` in your fork, but the import is still pointing to `github.com/prometheus/alertmanager/api`.  
So when I was pretty new to golang I solved it by searching and replacing all references to `github.com/prometheus/alertmanager/` with `github.com/fcastello/alertmanager/` this worked. I got my project to build but created a huge commits mess and to create the pull request I had to revert all imports back to their original, not a pretty solution. So then I found a much more elegant solution to the problem I was having.


# The solution
The solution I found more elegant for this problem, was to clone the upstream repo and add my fork as a remote and push my branches to the fork and not the original upstream repo (I would not be allowed to do so as I don't own the repo)

```bash
# I cloned the original upstream repo
$ git clone https://github.com/prometheus/alertmanager.git
$ cd alertmanager

# Then I listed the remotes for the repo
$ git remote -v
origin  https://github.com/prometheus/alertmanager.git (fetch)
origin  https://github.com/prometheus/alertmanager.git (push)
# As expected it's pointing to the upstream repo

# Then I added the fork on my github account to the repository named as fork
$ git remote add fork https://github.com/fcastello/alertmanager.git 

# Now I can see bot remote git repositories the upstream which is still
# named origin and my own which is called fork
$ git remote -v                                                    
fork    https://github.com/fcastello/alertmanager.git (fetch)
fork    https://github.com/fcastello/alertmanager.git (push)
origin  https://github.com/prometheus/alertmanager.git (fetch)
origin  https://github.com/prometheus/alertmanager.git (push)
```

Now we are ready to work with a branch locally, adding the remote can be done after if you are working with a local branch, the only thing to have in mind is to add the fork remote before pushing the local branch.  

So now we can do all modifications to the code and since we cloned the upstream repo the references to the imports don't change.  

Here is a basic example what we would to to work with a branch

```bash
# We create the local branch named mytestbranch 
git checkout -b mytestbranch    
Switched to a new branch 'mytestbranch'

# in this case we are only creating an empty file for showing how the process works
$ touch mytestfile

# show the status of git which shows our new file is untracked
$ git status
On branch mytestbranch
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        mytestfile
# we add that file to have it staged
$ git add mytestfile

# no git status says that file is staged and can be commited
$ git status
On branch mytestbranch
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   mytestfile

# we commit the staged files with a message
git commit -m "my test commit"

# and this is the IMPORTANT part of the process
# when pushing the commits from the branch we need to
# push them to the remote we named fork which point to our
# forked repository in github
$ git push -u fork mytestbranch:mytestbranch
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 287 bytes | 287.00 KiB/s, done.
Total 3 (delta 1), reused 1 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
remote: 
remote: Create a pull request for 'mytestbranch' on GitHub by visiting:
remote:      https://github.com/fcastello/alertmanager/pull/new/mytestbranch
remote: 
To https://github.com/fcastello/alertmanager.git
 * [new branch]        mytestbranch -> mytestbranch
Branch 'mytestbranch' set up to track remote branch 'mytestbranch' from 'fork'.
```

In the previous example we can see that changes were pushed to `https://github.com/fcastello/alertmanager.git` and when we are finished with our work on the branch that is on the forked repository we are ready to create the Pull Request following `https://github.com/fcastello/alertmanager/pull/new/mytestbranch` which will initiate the process to requesting the upstream project to pull changes from the `mytestbranch`in the forked repo on our account.  

This process makes our lives much easier because we don't need to touch the imports as we are still technically working on the upstream repository on a local branch, but we are pushing the branch to the different repository which is not the upstream repo.

# Conclusion

Working with multiple git remotes makes our live easier on open source projects golang development lifecycle. It is not pretty straightforward to figure out but once you get familiar with it it gets pretty easy to use. When contributing to an open source project first check the rules of the project they might have some special requirements to contribute to it or even have some extra steps to run their chosen CI tool. However this practice mentioned here is not a requirement to make a fork and pull request, it's just som trickery to have the local development experience much easier to work with.