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