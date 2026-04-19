[TOC]

## 压缩某个分支的所有节点，压成一个
> 具体场景：一个需求或者技术改造分支进度较慢，master分支已经向前走了很多。如果直接rebase，会面临每个节点都要处理冲突的场景，相当耗费时间。将分支提交压缩成一个，再做rebase操作

1. 进入要压缩的分支中；
2. 获取此分支新增的提交数：
```bash
git rev-list --count $(git merge-base master 分支名)..分支名
```
3. 得到计算数量，如20次提交；
4. git rebase -i HEAD~20；
5. 把第一条保留 pick，其余全部改成 squash
```
pick abc111 第一次提交
squash def222 第二次
squash ghi333 第三次
squash ghi444 第四次
...
```
6. 保存 → 完成 → 得到一个单提交；
7. git push -f


相关命令解析
- git merge-base master feature 得到 feature 分支与 master 的共同祖先（分支点）
- merge-base..feature 表示“从分支点到 feature 的所有提交”
- rev-list --count 统计数量

或者一条命令：`git rebase -i $(git merge-base master HEAD)`

这是最优雅的方式：
- git 会自动从分支点开始 rebase
- 不需要手算 20
- 编辑器中会自动列出这 20 个提交
- 然后同样把除第一条以外全部 squash 即可