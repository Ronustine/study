疑惑点

1. 相比BIO是使用了内存映射（跳过内核态的复制，翻页中断）
2. 可以用游标，灵活，读写共用一份buffer，但是又有清除操作？
3. 为什么BIO会阻塞，NIO不会（事件分发？）
4. 代码怎么写