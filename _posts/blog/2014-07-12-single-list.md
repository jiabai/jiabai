---
layout: post
title: 单链表的一个装逼写法
description: 单链表是最简单不过的数据结构了，平常使用时如果这样写，是不是显得有点逼格满满呢？
category: blog
---

## 单链表的一个装逼写法

代码如下：

    #include <stdio.h>
    #include <stdint.h>
    #include <stdlib.h>

    #define STAILQ_ENTRY(type)   \
    struct {                     \
      struct type *stqe_next;    \
    }

    struct mbuf {
      uint32_t           magic;   /* mbuf magic (const) */
      STAILQ_ENTRY(mbuf) next;    /* next mbuf */
      uint8_t            *pos;    /* read marker */
      uint8_t            *last;   /* write marker */
      uint8_t            *start;  /* start of buffer (const) */
      uint8_t            *end;    /* end of buffer (const) */
    };

    #define STAILQ_HEAD(name, type)       \
    struct name {                         \
      struct type *stqh_first;            \
      struct type **stqh_last;            \
    }

    STAILQ_HEAD(mhdr, mbuf);

    #define STAILQ_NEXT(elm, field)    ((elm)->field.stqe_next)

    #define STAILQ_INSERT_TAIL(head, elm, field) do {  \
      STAILQ_NEXT((elm), field) = NULL;                \
      *(head)->stqh_last = (elm);                      \
      (head)->stqh_last = &STAILQ_NEXT((elm), field);  \
    } while (0)

    int main(void)
    {
      struct mhdr *mhdr = malloc(sizeof(struct mhdr));
      mhdr->stqh_first = NULL;
      mhdr->stqh_last  = &mhdr->stqh_first;

      int i;
      for (i = 0; i < 16; i++) {
        struct mbuf *mbuf = malloc(sizeof(struct mbuf));
        STAILQ_INSERT_TAIL(mhdr, mbuf, next);
        printf("%p ", mbuf);
      }
      printf("\n");

      for (i = 0; i < 16; i++) {
        struct mbuf *mbuf = mhdr->stqh_first;
        printf("%p ", mbuf);
        mhdr->stqh_first = mhdr->stqh_first->next.stqe_next;
      }
      printf("\n");

      return 0;
    }

输入结果如下：

    [jabari@hbase-rs2-test ~]$ gcc a.c 
    [jabari@hbase-rs2-test ~]$ ./a.out 
    0x1804030 0x1804070 0x18040b0 0x18040f0 0x1804130 0x1804170 0x18041b0 0x18041f0 0x1804230 0x1804270 0x18042b0 0x18042f0 0x1804330 0x1804370 0x18043b0 0x18043f0 
    0x1804030 0x1804070 0x18040b0 0x18040f0 0x1804130 0x1804170 0x18041b0 0x18041f0 0x1804230 0x1804270 0x18042b0 0x18042f0 0x1804330 0x1804370 0x18043b0 0x18043f0

