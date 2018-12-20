# Go-Learning-Notebook

一个由golang爱好者一起维护一个最适合初学者的golang学习笔记



##  Join Us

可以联系 @Vincent-X 加入我们的微信群，或直接进行贡献



## How to Contribute

### 参与流程
```
if(笔记无人认领编写){
    认领(在Issue页面的Pending标签里选择未认领的章节)
    在自己的分支上写笔记
    PR / 提交
    等待合并到主分支或其他评论	
}else{
    认领(在Issue页面的Modifying标签里选择待润色与修改的章节)
    在自己的分支上修改笔记，新增内容or修改内容
    PR / 提交
    等待合并到主分支或其他评论
}
```



### Git操作流程

**准备工作**
1. 首先fork项目
2. 把fork到自己分支的项目，也就是你名下的项目clone到本地
3. 在命令行运行 `git branch editing`来创建一个新分支，这里用`editing`，你也可以修改成其他的名字比如,`modifying`.
4. 运行 `git checkout editing` 来切换到新分支
5. 添加官方的远端库，命名为`upstream`（也可以是其他名字），用来获取更新
  * 运行 `git remote add upstream https://github.com/1024hub/Go-Learning-Notebook.git` 把主仓库添加为远端库

步骤1~5是一个初始化流程，只需要做一遍就行，之后请一直在`editing`(或其他名字）分支进行修改。

**每次提交**

6. 提交之前在目录内运行`git remote update` 更新远端库
7. 在目录内运行`git fetch upstream master` 拉取远端的更新到本地
8. 运行`git checkout editing` 切换回你的日常分支后，运行`git rebase upstream/master`将远端库的更新合并到你的分支

如果修改过程中我们的主仓库有了更新，在对应的库目录下重复6、7、8步即可。也可以简写为`git pull --rebase upstream master`一条命令。或者你用 SourceTree 等 GUI 的话，在 push 面板下勾选用变基替代合并。也可以起到相同的作用，巧用变基，可以避免不必要的合并。

9. 修改之后，首先 Push 到你的库，然后登录 GitHub，在你的库的首页可以看到一个`pull request`按钮，点击它，填写一些修改(新建)的说明信息， 然后提交即可。

不管有没有直接修改权限，也建议使用PR处理，大规模多次零散的提交，这样管理和沟通都会方便。对open的PR所在的 branch（分支）进行多次提及，如 `vincent-x/Go-Learning-Notebook master`这些提交会挂在同一个branch之下。
