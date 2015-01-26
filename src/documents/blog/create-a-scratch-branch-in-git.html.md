---
title: Create a Scratch Branch in Git
date: 2015-01-28 10:00
type: blog
layout: post
tags: ['Git','How To']
---

Sometimes it gets to 4:59 on a Friday and you've got a bunch of files you've been mucking around with but don't really want to commit. This could be because you're just using throwaway code or because you've made changes to existing files that you don't want to break.

In these circumstances, you have a few options:

**Save everything, power down your machine and go home**  
All well and good. But what if your hard drive dies over the weekend? Or the roof leaks and your machine is a mess of mould?

**Copy your files up to a shared folder**  
Better, but what shared folder? If you have somewhere non-tamperable on the network then fair enough: that's a decent option.

**Create a temporary branch in Git**  
Yeah, but we said that we didn't want to create a commit. That's where the idea of a scratch branch comes in.

## Scratchy Scratchy
The idea of a scratch branch is to act as a temporary store. We'll add our files to the branch and push it to a central repository, kind of like a [shelveset in TFS](http://msdn.microsoft.com/en-us/library/ms181403.aspx).

When we come back after the weekend, we'll revert the commit in a safe way to get our current state back.

We'll start with our current, edited repository.

<pre class="brush: bash">
E:\myproject (master)                  
λ ls                                        
gruntfile.js  karma.conf.js  main.js  packag
                                            
E:\myproject (master)                  
λ git status                                
On branch master                            
nothing to commit, working directory clean  
</pre>

All clean. Let's hack a couple of the files.

<pre class="brush: bash">
E:\myproject (master)                                    
λ git status                                                  
On branch master                                              
Changes not staged for commit:                                
  (use "git add &lt;file&gt;..." to update what will be committed)  
  (use "git checkout -- &lt;file&gt;..." to discard changes in working directory)
                                                              
        modified:   main.js                                   
        modified:   package.json                              
                                                              
no changes added to commit (use "git add" and/or "git commit -a")
</pre>

Now we'll create a scratch branch and switch to that. Our changes will come along.

<pre class="brush: bash">
E:\myproject (master)
λ git checkout -b master-scratch
M       main.js
M       package.json
Switched to a new branch 'master-scratch'
</pre>

We can now add our files to the scratch branch and push them to the server.

<pre class="brush: bash">
E:\myproject (master-scratch)
λ git add .

E:\myproject (master-scratch)
λ git commit -m "Work in progress changes."
[master-scratch e576ae1] Work in progress changes.
 2 files changed, 3 insertions(+), 3 deletions(-)

E:\myproject (master-scratch)
λ git push origin master-scratch
Counting objects: 10, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (10/10), done.
Writing objects: 100% (10/10), 3.05 KiB | 0 bytes/s, done.
Total 10 (delta 2), reused 0 (delta 0)
To git://originontheserversomewhere.git
 * [new branch]      master-scratch -> master-scratch
</pre>

And we can go home knowing our code is backed up.

## Monday Morning
So, we're now back in the office after the weekend and want to get our code back to the way we had it. Assuming no disaster has befallen our machine, we can start working in the current directory as normal.

First, we need to revert the last scratch commit but keep the changes that were made.

<pre class="brush: bash">
E:\myproject (master-scratch)
λ git reset --soft HEAD^

E:\myproject (master-scratch)
λ git status
On branch master-scratch
Changes to be committed:
  (use "git reset HEAD &lt;file&gt;..." to unstage)

        modified:   main.js
        modified:   package.json

E:\myproject (master-scratch)
λ git log -1
commit 102c791a561e83f1a3f2d8ab0f132858fc934e79
Author: Kevin Wilson
Date:   Fri Jan 16 17:21:06 2015 +0000

    Initial commit.
</pre>

*NOTE: If you're using Cmder, it has some issues with the ^ character. You can get the same result as above using the command __git reset --soft HEAD~1__*

You can see that our last scratch commit has been undone and our files are staged for commit.

Now you can either continue working with the files as is (this is a decent way of tracking which files you've changed since the scratch commit) or you can unstage everything. If we want to go back to having everything unstaged, we just reset.

<pre class="brush: bash">
E:\myproject (master-scratch)
λ git reset
Unstaged changes after reset:
M       main.js
M       package.json

E:\myproject (master-scratch)
λ git status
On branch master-scratch
Changes not staged for commit:
  (use "git add &lt;file&gt;..." to update what will be committed)
  (use "git checkout -- &lt;file&gt;..." to discard changes in working directory)

        modified:   main.js
        modified:   package.json

no changes added to commit (use "git add" and/or "git commit -a")
</pre>

And we can even switch back to master&hellip;

<pre class="brush: bash">
E:\myproject (master-scratch)
λ git checkout master
M       main.js
M       package.json
Switched to branch 'master'

E:\myproject (master)
λ git status
On branch master
Changes not staged for commit:
  (use "git add &lt;file&gt;..." to update what will be committed)
  (use "git checkout -- &lt;file&gt;..." to discard changes in working directory)

        modified:   main.js
        modified:   package.json

no changes added to commit (use "git add" and/or "git commit -a")
</pre>

&hellip;and delete the scratch branch locally and from the server.

<pre class="brush: bash">
E:\myproject (master)
λ git push origin :master-scratch
To git://originontheserversomewhere.git
 - [deleted]         master-scratch

E:\myproject (master)
λ git branch -D master-scratch
Deleted branch master-scratch (was 102c791).
</pre>

After that, you're good to go.