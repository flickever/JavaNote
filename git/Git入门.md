# Git入门

### 设置用户签名

```shell
git config --global user.name 用户名

git config --global user.email 邮箱
```



### 初始化本地库

```
git init
```



### 查看本地库状态

```
git status
```



### 添加暂存区

```
git add 文件名
```



### 提交本地库

```
git commit -m "日志信息" 文件名
```



### 查看历史版本

```
git reflog 查看版本信息

git log 查看版本详细信息
```



### 版本穿梭

```
git reset --hard 版本号
```



### Git分支操作

#### 查看分支

```
git branch -v
```

#### 创建分支

```
git branch 分支名
```

#### **修改分支**

```shell
--在 maste 分支上做修改
$ vim hello.txt
--添加暂存区
$ git add hello.txt
--提交本地库
$ git commit -m "my forth commit" hello.txt
--查看分支
$ git branch -v
```

#### 切换分支

```
git checkout 分支名
```

#### **合并分支**

```
git merge 分支名
```

#### **产生冲突**

冲突产生的表现：后面状态为 MERGING

冲突产生的原因：

​		合并分支时，两个分支在**同一个文件的同一个位置**有两套完全不同的修改。Git 无法替我们决定使用哪一个。必须**人为决定**新代码内容。

**解决冲突**

1. 编辑有冲突的文件，删除特殊符号，决定要使用的内容
2. 添加到暂存区
3. 执行提交（注意：此时使用 git commit 命令时**不能带文件名**）

```
$ git commit -m "merge hot-fix"
```



### **远程仓库**

#### **创建远程仓库别名**

```shell
git remote -v 查看当前所有远程地址别名
git remote add 别名 远程地址

例子：
$ git remote add ori https://github.com/atguiguyueyue/git-shTest.git
```

#### **推送到远程仓库**

```
git push 别名 分支

例子
$ git push ori master
```

#### 克隆远程仓库到本地

```shell
git clone 远程地址

例子
$ git clone https://github.com/atguiguyueyue/git-shTest.git
```



#### **邀请加入团队**

1. github -> Setting -> Manage access -> invite a collaborator
2. 填入想要合作的人的账号
3. 复制地址并发送给该用户
4. 使用被邀请的账号，地址栏复制收到邀请的链接，点击接受邀请
5. 成功之后，被邀请账号可以修改仓库



#### **拉取远程库内容**

```shell
git pull 远程库地址别名 远程分支名

例子
$ git pull ori master
```



#### **跨团队协作**

1. 将远程仓库的地址复制发给邀请跨团队协作的人
2. 被邀请人GitHub账号里的地址栏复制收到的链接，然后点击 **Fork** 将项目叉到自己的本地仓库。
3. 在线编辑叉取过来的文件
4. 编辑完毕后，填写描述信息并点击页面左下角绿色按钮提交
5. 接下来点击上方的Pull请求，并创建一个新的请求
6.  仓库拥有者GitHub 账号可以看到有一个 **Pull request**请求
7. 如果代码没有问题，可以点击 **Merge pull request** 合并代码



#### **SSH** **免密登录**

```
 ssh-keygen -t rsa -C 用户描述信息
```



