- 安装git 和 git-email

`sudo apt-get update`

- 配置git

```bash
git config --global user.name "nameVal"
git config --global user.email "eamil@qq.com"

# 查看配置
git config --get-all user.name
git config --get-all user.email
```



- 配置smtp

```bash
vi ~/.gitconfig
# 在文件末尾添加
[sendemail]
        smtpencryption=ssl
        smtpserver=smtp.163.com
        smtpuser=a929244872@163.com
        smtpserverport=465
        smtppass=RGAIVJTTGIPXWGSA    #邮箱授权码ibvnraoxojoqbded   qq
#sendemail必须与[user]的email一致
```

![](./image/1.PNG)

- 生成补丁

```bash
#修改文件后
git add .
git status
git commit -s -v
```

Commit 信息的格式有严格限制

```
drivers: fix some error

Why I do these changes and how I do it.

Signed-off-by: My Name <my_email@gmail.com>
```

如果 commit 之后还想修改 Commit 信息的话需要使用命令 `git commit --amend -v`

```bash
git format-patch -s -1 # 将自动按最近一次提交生成一个补丁文件 xxx.patch
git format-patch  --subject-prefix='PATCH'  -1

./scripts/checkpatch.pl xxx.patch # 检查补丁的格式是否合法
./scripts/get_maintainer.pl xxx.patch # 补丁是以邮件形式发送，这里是找出要发送的邮箱

#输出如下
Andrew Morton <akpm@linux-foundation.org> (maintainer:MEMORY MANAGEMENT)
linux-mm@kvack.org (open list:MEMORY MANAGEMENT)
linux-kernel@vger.kernel.org (open list)

```

- 发送邮件

```bash
#给自己发送测试
git send-email --smtp-debug --to=a929233872@163.com,929244872@qq.com --cc=929244872@qq.com 0001-mm-fix-some-error.patch

```


