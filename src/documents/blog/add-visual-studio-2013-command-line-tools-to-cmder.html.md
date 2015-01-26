---
title: Add Visual Studio 2013 Command Line Tools to Cmder
date: 2015-01-14 10:00
type: blog
layout: post
tags: ['Cmder', 'How To']
---

[Cmder](http://bliker.github.io/cmder/) is an excellent console emulator and my go-to tool for Git on Windows.

Every now and again I need to dip into the Visual Studio Command Tools to do something like manually run MSBuild, so it's handy (and easy) to have this set up as a task in [Cmder](http://bliker.github.io/cmder/).

## 1. Manage Tasks
Click the plus icon towards the bottom right of the console and choose "Setup tasks" from the options.

<img src="http://kwilson.me.uk/blog/wp-content/uploads/2015/01/1-setup-tasks.png" alt="1-setup-tasks" width="645" height="561" class="aligncenter size-full wp-image-2131" />

## 2. Add a New Task
Select the plus icon at the bottom of the tasks list to create a new blank one.

<img src="http://kwilson.me.uk/blog/wp-content/uploads/2015/01/2-add-new-task.png" alt="2-add-new-task" width="718" height="563" class="aligncenter size-full wp-image-2141" />

## 3. Fill Out the Task Settings
Add a name for your new task (I used 'VS2013') and enter the following as the command:

<pre class="brush: ps">
cmd /k "%VS120COMNTOOLS%VsDevCmd.bat"  -new_console:d:"E:\projects":t:"VS2013"
</pre>

The *-new_console* flag allows you to [set various properties for the new tab](https://code.google.com/p/conemu-maximus5/wiki/NewConsole) such as the directory (that's the *:d* part) in which the console will start up, and the tab name (*:t*). My project directory is in *E:\projects* so that's what I've entered. You can set it to whatever directory suits you best. I've also named my tab "VS2013".

<img src="http://kwilson.me.uk/blog/wp-content/uploads/2015/01/3-save-task1.png" alt="3-save-task" width="678" height="509" class="aligncenter size-full wp-image-2181" />

Click "Save Settings" when you're done.

## 4. Launch Your New Task
You can now launch your new VS Tools task through the plus arrow in the bottom right.

<img src="http://kwilson.me.uk/blog/wp-content/uploads/2015/01/4-launch-task.png" alt="4-launch-task" width="643" height="576" class="aligncenter size-full wp-image-2161" />