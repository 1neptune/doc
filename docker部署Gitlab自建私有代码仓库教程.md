##### 创建目录

```bash
mkdir -p /data/gitlab/{config,logs,data,backups}
cd /data/gitlab
```

##### 创建 docker-compose.yml

```bash
cat > /data/gitlab/docker-compose.yml <<'EOF'
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: '172.19.6.98'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://172.19.6.98:9080'
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 4522
        prometheus_monitoring['enable'] = false
        gitlab_kas['enable'] = false
        puma['worker_processes'] = 8
        sidekiq['max_concurrency'] = 20
    ports:
      - "9080:9080"
      - "4522:22"
    volumes:
      - /data/gitlab/config:/etc/gitlab
      - /data/gitlab/logs:/var/log/gitlab
      - /data/gitlab/data:/var/opt/gitlab
      - /data/gitlab/backups:/var/opt/gitlab/backups
    shm_size: "256m"
EOF
```

##### 启动 GitLab

```bash
cd /data/gitlab
docker compose pull gitlab
docker compose down gitlab
docker compose up -d  gitlab
docker compose logs -f gitlab
# 获取 root 初始密码
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password

root/4XXeJLEbngD/j5n839hEJjb7I6pKik2f6t7lAUUbFjI=
```

##### 访问GitLab

```bash
http://172.19.6.98:9080/

marcopolo/Yao86784126
```

![](D:/marktext/images/2026-09-21-15-09-14-image.png)

##### 配置Git

```bash
git config --global user.name "yaoyw"

git config --global user.email "yaoyw@marcopolo.com.cn"

ssh-keygen -t rsa -b 4096 -C "yaoyw@marcopolo.com.cn"

C:\Users\1nepnute\.ssh\id_rsa.pub

复制公钥，GitLab 右上角头像 → Edit profile → SSH Keys → 粘贴到 Key 框

ssh -T git@172.19.6.98 -p 4522

# 输出 
The authenticity of host '[172.19.6.98]:4522 ([172.19.6.98]:4522)' can't be established.
ED25519 key fingerprint is SHA256:7ZPRtFA9vCGgmt+G2YSfKtwUuDOkCBI4hcKYy4NRhlk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[172.19.6.98]:4522' (ED25519) to the list of known hosts.
Welcome to GitLab, @marcopolo!
```

##### 配置项目

```bash
Projects → New project → Create blank project：

Project name：payslip

Project URL：选组 marcopolo

http://172.19.6.98:9080/marcopolo/payslip

Visibility：Private

勾选 Initialize repository with a README
```

##### 配置项目用户权限

```bash
项目 marcopolo/payslip → Manage → Members → Invite member：

选 yaoyw

Role：Maintainer

点 Invite
```

##### 创建 test 和 master 分支

```bash
项目 → Repository → Branches → New branch：

main 创建 test

main 创建 master
```

##### 设置保护分支

```bash
项目 → Settings → Repository → Protected Branches


Branch    Allowed to merge    Allowed to push
main    Maintainers            No one
master    Maintainers          No one
```

##### ![](D:/marktext/images/2026-09-21-15-36-32-image.png)

##### 设置保护 Tag

```bash
项目 → Settings → Repository → Protected Tags → New protected tag


Tag                    v*
Allowed to create    Maintainers
```

##### ![](D:/marktext/images/2026-09-21-15-37-24-image.png)

##### 设置默认分支

```bash
项目 → Settings → Repository → Default branch 改为 main
```

![](D:/marktext/images/2026-09-21-15-38-02-image.png)

##### 初始化本地仓库

```bash
cd /d E:\智能工资条

git init

# 配置身份
git config user.name "姚裕伟"
git config user.email "yaoyw@marcopolo.com.cn"

# 创建 .gitignore
# E:\智能工资条 目录下创建 .gitignore
node_modules/
dist/
*.log
.DS_Store
.idea/
.vscode/
*.tmp
```

##### 添加远程仓库

```bash
# SSH免密
git remote set-url origin ssh://git@172.19.6.98:4522/marcopolo/payslip.git

# GitLab 登录密码
git remote add origin http://172.19.6.98:9080/marcopolo/payslip.git

git remote -v

# 输出
origin  http://172.19.6.98:9080/marcopolo/payslip.git (fetch)
origin  http://172.19.6.98:9080/marcopolo/payslip.git (push)
```

##### 拉取远端分支

```bash
git fetch origin

#输出
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
Unpacking objects: 100% (3/3), 2.73 KiB | 186.00 KiB/s, done.
From http://172.19.6.98:9080/marcopolo/payslip
 * [new branch]      main       -> origin/main
 * [new branch]      master     -> origin/master
 * [new branch]      test       -> origin/test

# 查看远端分支
git branch -r
# 输出
  origin/main
  origin/master
  origin/test

# 查看当前改动
git status

# 切换到 test 分支 
git checkout -b test origin/test


# 切换到 main 分支
git checkout -b main origin/main

# 添加所有文件到暂存区
git add .


# 确认要提交的文件
git status


# 如果发现了不该提交的文件，比如 .env：

# 从暂存区移除（保留本地文件）
git rm --cached .env

# 补进 .gitignore
echo ".env" >> .gitignore
git add .gitignore

# 提交
git commit -m "feat: 智能工资条项目本地测试环境"

# 切换 test分支
git checkout test

# 推送test分支
git push origin test


# 输出 原因：test分支设置了Protected branches
E:\智能工资条>git push origin test
Enumerating objects: 19565, done.
Counting objects: 100% (19565/19565), done.
Delta compression using up to 12 threads
Compressing objects: 100% (13239/13239), done.
Writing objects: 100% (19564/19564), 520.06 MiB | 34.05 MiB/s, done.
Total 19564 (delta 4816), reused 19563 (delta 4816), pack-reused 0
remote: Resolving deltas: 100% (4816/4816), done.
remote: GitLab: You are not allowed to push code to protected branches on this project.
To http://172.19.6.98:9080/marcopolo/payslip.git
 ! [remote rejected]   test -> test (pre-receive hook declined)
error: failed to push some refs to 'http://172.19.6.98:9080/marcopolo/payslip.git'


# 创建分支
前缀     用途    示例
feature/    新功能    feature/login、feature/payslip-query
hotfix/    紧急修复    hotfix/payslip-page-500
fix/    普通 Bug 修复    fix/login-error
release/    发布分支    release/v1.0.0  release/v0.1.0-test
docs/    文档    docs/api-update
refactor/    重构

# 从 test 创建功能分支
git checkout -b release/v0.1.0-test

# 1. 添加所有文件到暂存区
git add .

# 2. 确认暂存了什么
git status

# 提交
git commit -m "feat: 智能工资条项目测试环境"

# 推送功能分支
git push -u origin release/v0.1.0-test


# 打测试版 Tag
git tag -a v0.1.0-test -m "Release v0.1.0-test 内部测试版"

git push origin v0.1.0-test


# 删除分支
git branch -d release/v0.1.0-test


# 创建 MR：release → master

项目 → Merge requests → New merge request：

Source：release/v0.1.0-test

Target：test

Title：release: v0.1.0-test 测试版

Create → Merge
```

##### 取消自动部署

![](D:/marktext/images/2026-09-21-17-00-16-image.png)

![](D:/marktext/images/2026-09-21-17-00-03-image.png)

![](D:/marktext/images/2026-09-21-16-59-44-image.png)

##### 拉取代码

```bash
# 查看代码版本

# 1. 先备份当前 test 分支
git branch test-backup

# 2. 看状态
git status
git branch

# 3. fetch 预览远端有什么
git fetch origin
git log --oneline test..origin/test   # 远端多了哪些
git log --oneline origin/test..test   # 本地多了哪些

# 4. 有未提交改动先 stash
git stash

# 5. rebase 合并
git pull --rebase origin test

# 6. 恢复 stash
git stash pop

# 7. 推送
git push origin test

# 8. 成功后删备份
git branch -d test-backup


#查看当前分支所有提交
 git log --oneline
# 输出
62f8b26e (HEAD -> test, origin/test) Merge branch 'release/v0.1.0-test' into 'test'
701095b9 (origin/release/v0.1.0-test) feat: 智能工资条项目测试环境
086a5f19 (tag: v0.1.0-test) feat: 智能工资条项目测试环境
ebf91b07 feat: 智能工资条项目测试环境
a6e99272 feat: 智能工资条项目本地测试环境
67f42add (origin/master, main) Initial commit
```

##### Git 恢复源码实战

```bash
Git 恢复源码步骤总结

本次问题：`test` 分支历史从未追踪源码（只提交过 node_modules/target/.tools 等产物），加上误操作导致工作区源码目录被移出。恢复依据： master 分支快照`04907246` （2026-09-21 15:58 提交）含完整最新源码 。


# 定位源码复原点
# 找出哪个提交含 backend/src（含源码的历史快照）
git log --all --oneline -- backend/src
# 或直接看分支树
git ls-tree -r --name-only 04907246 -- backend/src      # 该提交含 74 个后端源码
git ls-tree -r --name-only 04907246 -- frontend/src     # 含 22 个前端源码


# 将源码恢复到工作区（关键一步）
git checkout 04907246 -- .


# 含义：把`04907246` 树中 所有文件 检出到工作区+索引（不改 HEAD），源码即回到磁盘。

# 处理 .env.example 入库

# 调整 .gitignore：允许 .env.example（模板）入库，仅忽略真实 .env
git add .gitignore

# 若文件曾被忽略，需强制添加（本次 rocky9/.env.example 曾被 master 版 .gitignore 忽略）
git add -f deploy/.env.example deploy/rocky9/.env.example


# 提交并推送到 test
git add -A
git commit -m "restore: 从 master 快照(04907246)恢复完整源码到 test 分支，并纳入 .env.example 部署模板" 
git push origin test # 生成提交 aad951eb

# 同步到 main（本次采用"main 直接指向源码快照"方案）

# 本地 main 直接指向 test 的最新源码提交（须在非 main 分支上执行）
git branch -f main aad951eb

# 因 main 历史与 test 不同源，需带租约保护的安全强制推送
git push origin main --force-with-lease
# 输出: + c07d1cd3...aad951eb main -> main (forced update)  ✅


# 验证
git ls-remote origin test main        # 两个分支应都返回 aad951eb…
```

##### 代码合并

```bash
如果我在写代码的时候。有人上传代码到test仓库了，但是我没有拉取到最新代码

一句话总结： push 失败 = 远端有更新，先`git pull --rebase` 再`git push` ；本地没提交的改动用`git stash` 挂起再`pull` ，最后`stash pop` 。所有操作都可逆，不会丢已保存的代码。

写代码前/后各拉一次，形成习惯
git pull origin test            # 开工前
# ... 写代码 ...
git add . && git commit -m "..." 
git pull origin test --rebase   # push 前再拉一次，几乎不会冲突
git push origin test

# 情况 A：本地有改动，但 还没 commit
# 1) 把未提交的改动暂存起来（-u 连未跟踪的新文件一起）
git stash push -u -m "wip-我的改动"

# 2) 拉取 test 分支最新代码
git pull origin test

# 3) 恢复自己的改动
git stash pop


# 情况 B：本地已 commit， push 时被拒 （远端有新提交）
# 1) 推荐用 rebase 拉取：把自己的提交"垫"到远端最新提交之上，历史更干净
git pull origin test --rebase

# 2) 再推上去
git push origin test

# 若`--rebase` 出现冲突，解决后继续
git add <冲突文件>
git rebase --continue
git push origin test


# 情况 C：已经 push 过了的提交，别人又改了同一文件
# 1) 拉取远端，必要时合并
git pull origin test

# 2) 若有冲突，改成保留双方需要的部分后
git add <文件>
git commit
git push origin test


# 冲突处理（A/B 都可能遇到）
git add <冲突文件>          # 手动或 IDE 处理后标记为解决
git stash pop              # 若之前是 stash 方式
# 或
git rebase --continue      # 若之前是 rebase 方式
```
