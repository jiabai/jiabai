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

```
fsimage_0000000000000000000 (fsimage)
fsimage_0000000000000000000.md5
edits_0000000000000003414-0000000000000003451 (edit logs, there are many ones with different name)
seen_txid (a separated file contains last seen transaction id)
```

When NN starts, Hadoop will load fsimage and apply all edit logs, and meanwhile do a lot of consistency checks, it'll abort if the check failed. Let's make it happen, I'll rm edits_0000000000000000001-0000000000000000002 from many of my edit logs in my NN workspace, and then try to sbin/start-dfs.sh, I'll get error message in log like:

> java.io.IOException: There appears to be a gap in the edit log.  We expected txid 1, but got txid 3.

So your error message indicates that your edit logs is inconsitent(may be corrupted or maybe some of them are missing). If you just want to play hadoop on your local and don't care its data, you could simply hadoop namenode -format to re-format it and start from beginning, otherwise you need to recovery your edit logs, from SNN or somewhere you backed up before.

通常在HA配置时，需要先启动journalnode，之后运行：

```
hdfs namenode -initializeSharedEdits
```

来把namenode的edits log同步到各个journalnode节点上，journalnode只负责存储edits log，不存FsImage。

经过我的操作并观察发现，这个同步不一定能跟namenode完全一致，不明白这程序同步的依据是什么，同步之后journalnode和namenode的edits_inprogress会不一样，不过也不用担心，也不需要额外操作，过一会儿edits_inprogress就会一致了。
还有一点要说，经过hdfs namenode -initializeSharedEdits命令同步的edits log只是最近的一部分，如果之前是非HA的运行过一段时间的集群，edits log会有很多也许有几千个，那么它只会同步最近一段时间的一部分edits log。

---

在standby namenode上执行：

```
hdfs namenode -bootstrapStandby
```

我发现只同步了FsImage，没有同步edits log，同样不明白这个命令的原理了。
【妈的智障，-bootstrapStandby本来就是只同步FsImage文件，edits log的load全靠从journalnode加载】

不过也无所谓，直接scp active namenode的所有元数据到standby namenode的元数据目录下，直接启动standby namenode就OK。

---

这又算是一个悬而未决的结果，有机会再补充吧。

