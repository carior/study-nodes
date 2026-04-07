当前项目需要切换到master分支，在终端中打开当前项目，执行`mkdir -p .worktrees`

若同一个项目有两个需求分支需要同时进行，在2个终端分别打开两个项目

终端1执行：

`git worktree add .worktrees/对公签约结算 feature/对公益约结算线上化`

终端2执行：

`git worktree add .worktrees/收入扶持 feature/原创品类作品收入扶持5.0 `

查看当前的worktree`git worktree list`

![image-20260306172203147](/Users/staff/Library/Application Support/typora-user-images/image-20260306172203147.png)



![image-20260306172348037](/Users/staff/Library/Application Support/typora-user-images/image-20260306172348037.png)

 

开发完成后，删除worktrees，在当前项目下执行：

`git worktree remove .worktrees/收入 扶持`

### 常见报错

![image-20260325113807649](/Users/staff/Library/Application Support/typora-user-images/image-20260325113807649.png)

当前项目也在这个功能分支，可以切换到master分支
