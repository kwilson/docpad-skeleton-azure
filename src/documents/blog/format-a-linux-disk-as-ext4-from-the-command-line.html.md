---
title: Format a Linux Disk as Ext4 from the Command Line
date: 2014-10-11 00:40
type: blog
layout: post
tags: ['Linux', 'How To']
---

There are plenty of guides for how to do this online. But I end up spending 20 minutes searching every time I need to remind myself how to do it, so I'm putting it here to save me that hassle.

For the code, I'm just going to assume the drive is **/dev/sdb**. If you're following this, make sure you're using the correct drive name.

## Unmount The Drive
Only needed if it's been mounted already.

<pre class="brush: bash">
sync
umount /dev/sdb1
</pre>

## Prepare the Drive
Fire up fdisk
<pre class="brush: bash">
fdisk /dev/sdb
</pre>

If you already have a filesystem on the disk, wipe it.
<pre class="brush: bash">
d
</pre>

Now create a new primary partition.
<pre class="brush: bash">
p
</pre>

Enter the partition number.
<pre class="brush: bash">
1
</pre>

It'll now ask you to set the start and end cylinders on the disk. Just hit *return* for each to accept the defaults.

Change the system type to Linux.
<pre class="brush: bash">
t
83
</pre>

Write the changes to disk (this can't be undone).
<pre class="brush: bash">
w
</pre>

## Create the Filesystem
Format the new partition using Ext4.
<pre class="brush: bash">
mkfs.ext4 /dev/sdb1
</pre>

It'll now go off and start writing the filesystem. This can take a bit of time, but it should give you an update on what's being written.

## Mount Your Drive
Now your partition has been created, you need to mount it somewhere to use it. Here I'm using **/mnt/user/ext/**.
<pre class="brush: bash">
mnt /dev/sdb1 /mnt/user/ext/
</pre>

And there you go.