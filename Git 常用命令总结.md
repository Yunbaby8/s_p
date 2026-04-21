查看本地分支

```
git branch
```

创建新分支（只创建，不切换）

```
git branch newbranch
```

切换到已有分支

```
git checkout newbranch
```

创建新分支并切过去

```
git checkout -b newbranch
```

删除本地分支

```
git branch -d sirius_dev
```

如果 Git 说这个分支没合并，不让删，但你确定不要：

```
git branch -D sirius_dev
```



可以，既然你这次**还没有 commit**，那就直接把**当前工作区改动**打成 patch。

最常用就是这句：

```
git diff > my_changes.patch
```

意思是：

- `git diff`：导出当前**工作区**相对于当前版本的改动
- `>`：输出到文件
- `my_changes.patch`：patch 文件名





git push -u origin task453

推送到 `origin/task453_3.0`

顺便设置本地分支跟踪远端分支