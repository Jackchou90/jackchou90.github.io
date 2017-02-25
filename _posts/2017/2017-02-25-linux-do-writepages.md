---
title: do_writepages 内核底层通用页缓存回写函数
layout: post
---

# do_writepages

该函数在页缓存回写这块非常重要，系统中各种页缓存回写操作最终都由其接管，是真正的页缓存操作基础。具体使用及流程参加如下两篇页缓存回写文章:   

1. [linux4.0页缓存回写机制](https://jackchou90.github.io/2017/02/24/linux40-page-cache-writeback.html)
2. [fadvise64_64中POSIX_FADV_DONTNEED选项作用及流程分析](https://jackchou90.github.io/2017/02/25/fadvise64-64-posix-fadv-dontneed.html)


