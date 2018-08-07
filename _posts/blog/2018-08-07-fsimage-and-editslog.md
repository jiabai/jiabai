---
layout: post
title: namenode HA配置：FsImage和edits log
description: HA配置中，namenode 1&2&journalnode的元数据都错乱了，怎么恢复的？
category: blog
---

The metadata in Hadoop NN consists of:

- fsimage: contains the complete state of the file system at a point in time
- edit logs: contains each file system change (file creation/deletion/modification) that was made after the most recent fsimage.

If you list all files inside your NN workspace directory, you'll see files include:

> fsimage_0000000000000000000 (fsimage)
> fsimage_0000000000000000000.md5
> edits_0000000000000003414-0000000000000003451 (edit logs, there're many ones with different name)
> seen_txid (a separated file contains last seen transaction id)

When NN starts, Hadoop will load fsimage and apply all edit logs, and meanwhile do a lot of consistency checks, it'll abort if the check failed. Let's make it happen, I'll rm edits_0000000000000000001-0000000000000000002 from many of my edit logs in my NN workspace, and then try to sbin/start-dfs.sh, I'll get error message in log like:

> java.io.IOException: There appears to be a gap in the edit log.  We expected txid 1, but got txid 3.

So your error message indicates that your edit logs is inconsitent(may be corrupted or maybe some of them are missing). If you just want to play hadoop on your local and don't care its data, you could simply hadoop namenode -format to re-format it and start from beginning, otherwise you need to recovery your edit logs, from SNN or somewhere you backed up before.

---

暂时先这些。
