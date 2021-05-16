# 使用 Github actions 部署 SSR 应用

关于 github actions 应该不用多做介绍，不了解的同学可以看看官方文档：[https://docs.github.com/cn/actions](https://docs.github.com/cn/actions)<br />这次我将不使用第三方插件，完全从零实现利用 github actions 部署 SSR 应用，大致步骤为:

- clone 项目并切换至对应分支
- 项目打包
- ssh 配置
- 上传静态资源
- 利用 ssh 操作远程服务器

## 新建一个基本项目

```bash
$ mkdir github-actions-deploy-ssr
# 新建 src/index.js 文件，内容为 console.log('app')
$ mkdir src && echo "console.log('app')" >> src/index.js
```

接下来我们按照[文档](https://docs.github.com/en/actions)，在项目目录下 `.github/workflows` 文件夹下创建一个脚本文件 `deploy.yml`

```bash
$ mkdir .github && mkdir .github/workflows && touch .github/workflows/deploy.yml
```

写个最简单的脚本

```yaml
name: Build and deploy
on:
  push:
    branches:
      - master
jobs:
  Build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: echo "build"
      - run: echo "deploy"
```

- name 脚本名称
- on 触发条件，详细介绍可见[文档](https://docs.github.com/cn/actions/reference/events-that-trigger-workflows)，这里配置的是在 push 到 master 的时候触发
- jobs 任务细节，可定义多个 job
- job.runs-on 执行的虚拟机环境
- job.steps 具体执行步骤，steps 是个数组，可以配置 name / run / uses 等多个配置，并且 name 和 run 必须有一个
- job.step.name 步骤名称
- job.step.run 执行的内容，就是 shell 命令
- job.step.uses 应该是会用到比较多的，他可以直接使用 github 上面的第三方插件。比如 `actions/checkout@v2` 。这些插件名称规则是 `${github_owner}/${github_repo}@${branch|tag}`，比如 `actions/checkout@v2` 就是使用 [https://github.com/actions/checkout](https://github.com/actions/checkout) 这个仓库里面 v2 的 tag


<br />所以上面脚本的大概的意思就是：在 push 到 master 的时候，在 `ubuntu`的最新环境下执行两个步骤:

- 输出 build
- 输出 deploy


<br />我们提交代码到 github 看下效果<br />![](https://cdn.nlark.com/yuque/0/2021/png/658400/1618757886388-d50840b5-f42b-43b2-922e-68052c72c1f8.png#clientId=ud5c735aa-da55-4&from=drop&id=ub3b9b730&margin=%5Bobject%20Object%5D&originHeight=944&originWidth=2070&originalType=binary&size=124318&status=done&style=none&taskId=u5bc8c3eb-2d75-475f-8004-1159c75b4ac)<br />OK，没毛病。可以看到第二个 step 我们没有配置 name，所以他的标题就是 `Run ${command}` -> `Run echo "deploy"`

## github actions 脚本

需要打包和部署的话，首先我们需要把我们的项目代码 clone 下来。这里可以使用 [actions/checkout](https://github.com/actions/checkout) 去做，当然，我们也可以自己写

### 克隆项目

我们先在脚本里面添加一个 step 叫 `Clone repository`

```yaml
jobs:
  Build-and-deploy:
    # ... 省略
    steps:
    	# ... 省略
      - name: Clone repository
        run: |
          echo "clone repository"
```

我们这部需要做两件事

- clone 项目 (git clone https://github.com/xxx/yyy)
- 将项目目录设置为工作目录

#### Clone

我们可以直接把命令写死，比如

```bash
$ git clone https://github.com/wokeyi/github-actions-deploy-ssr
```

但是这样太不智能，看下[文档](https://docs.github.com/cn/actions/reference/context-and-expression-syntax-for-github-actions#github-context)可以看到。github actions 为我们提供了很多上下文，比如 `github.repository`/ `github.ref`等，那么我们就可以把命令改成

```bash
# 用 ${{ }} 的方式使用 github actions 变量
$ git clone https://github.com/${{ github.repository }}
```

#### 将项目目录设置为工作目录

这一步我们拿到项目名称，因为 git clone 下来就是以项目名称命名，比如我们这个 demo 名称就是 `github-actions-deploy-ssr` ，但 `github.repository` 这个变量提供的为 `wokeyi/github-actions-deploy-ssr` 的字符串, 所以我们需要简单写个字符串操作的脚本

```bash
# 将 github.repository 用变量 repository_name 保存
$ REPO_NAME="${{ github.repository }}"
# 字符串操作，相当于 const ARRAY = REPO_NAME.split('/')
$ ARRAY=(${REPO_NAME//\// })
# 拿到数组第二项
$ DIR_NAME=${ARRAY[1]}
$ cd $DIR_NAME
```

这样子 `DIR_NAME` 就是项目的目录，后续操作都将在这里面操作<br />
<br />但是这里有个坑，我们在这一步 `cd $DIR_NAME` 进去了 clone 下来的项目目录，下一个 step 又回到了默认的目录，这里有两种解决方案<br />

1. 配置 `working-directory` 属性 (在 env 里面增加一个 `working_dir` 变量，用 `echo "working_dir=$DIR_NAME" >> $GITHUB_ENV` 来更新 这个变量，并把他设置在后续的 step 中)
1. 直接 git clone 到当前目录


<br />很明显第二种方法比较简单，我们按照这个方法来改，只需一行代码

```bash
# 后面多了一个 `.`
$ git clone https://github.com/${{ github.repository }} .
```

#### 切换到对应分支

因为 clone 下来时默认分支一般是 `master`，但是我们可能是在其他分支触发的更新，所以直接在 `master` 分支上打包明显是不对的。所以这一步我们需要 `checkout` 到对应分支<br />
<br />增加一个 step 加 `Check out branch`

```yaml
jobs:
  Build-and-deploy:
  	# ... 省略
    steps:
    	# ... 省略
      - name: Check out branch
        run: |
          echo "check out branch"
```

这一步我们需要做这些事情：

- 拿到当前分支名
- 拿到触发这个任务的分支名 ( github context 里面提供了 `github.ref` 给我们使用)
- 分支一样不做操作，不一样则切换到对应分支


<br />直接上代码

```bash
# 当前触发的分支: refs/heads/master
GITHUB_REF="${{ github.ref }}"
# 相当于 const ARRAY = GITHUB_REF.split('/')
ARRAY=(${GITHUB_REF//\// })
BRANCH_NAME=${ARRAY[2]}
# 拿到当前的分支名
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
# 判断当前分支和触发分支是否一样
if [ $BRANCH_NAME != $CURRENT_BRANCH ]; then
  echo "Start to fetch origin/$BRANCH_NAME"
  git fetch origin $BRANCH_NAME
  git checkout $BRANCH_NAME
else
  echo "Current branch is $CURRENT_BRANCH."
fi
```

## 项目打包

上面的流程跑完后我们就拿到了要操作的代码，在打包这一步大部分都是 `yarn` + `yarn build`。这个根据特定项目来，不做过多叙述

```yaml
jobs:
  Build-and-deploy:
  	# ... 省略
    steps:
    	# ... 省略
      - name: Clone repository
      - name: Check out branch
      - name: Build
        run: |
        	# yarn
          # yarn build
          echo "Build"
```

## ssh 配置

不管是客户端上传静态资源到 nginx 或者服务端应用的 `docker-compose build`都需要 ssh，我们这一步来完成 ssh 配置<br />​

我们知道想要**免密码**通过 ssh 登录远程服务器的话，需要把本机的公钥放在服务器的 `~/.ssh/authorized_keys`中。这样子的话就有一个问题，我们不知道 github 触发 actions 的机器的公钥，就没办法配置在远程服务器中了。<br />​

这种情况我们有这么一个解决方案：

- 将本机的**私钥**提供给 GitHub actions，利用私钥动态创建对应的公钥(生成的公钥和本机的公钥是一模一样的)
- 将本机的公钥保存在服务器的 `~/.ssh/authorized_keys` 中即可

<br />

#### 在 secrets 中保存 ssh 配置

<br />进入代码仓库中的 `Settings`-> `Secrets`中，点击右上角 `New repository secret` 可以创建一个 secret<br />![截屏2021-05-15 上午10.14.39.png](https://cdn.nlark.com/yuque/0/2021/png/658400/1621044886115-9d6f3203-f8b6-411c-a7d3-f6e9a93661a9.png#clientId=u75639dfc-f39e-4&from=drop&id=u5bd1be2d&margin=%5Bobject%20Object%5D&name=%E6%88%AA%E5%B1%8F2021-05-15%20%E4%B8%8A%E5%8D%8810.14.39.png&originHeight=1446&originWidth=2796&originalType=binary&size=367560&status=done&style=none&taskId=u421aa753-bba2-41c2-bd40-52f18c2e6b0)

- 创建一个 Key 为 `SSH_PRIVATE_KEY` ，内容是本机私钥的 secret，本机私钥保存在 `~/.ssh/id_rsa` 中 (还没有私钥的话可以通过执行 `ssh-keygen` 创建一个新的的公私钥对)
- 创建一个 Key 为 `SSH_USERNAME` 内容是连接远程服务器的用户名
- 创建一个 Key 为 `SSH_HOST` 内容是连接远程服务器的公网 IP



这样子我们就完成了 ssh 所需的环境配置，可以通过 `${{ secrets.SSH_PRIVATE_KEY }}` 在 github actions 中使用<br />​<br />

#### 动态创建公钥

有了私钥后，我们可以通过 `ssh-keygen` 来生成对应的公钥

```bash
SSHPATH="$HOME/.ssh"
# 写入私钥
echo "${{ secrets.SSH_PRIVATE_KEY }}" >> $SSHPATH/id_rsa
# 更新私钥文件权限
chmod 600 $SSHPATH/id_rsa
# 通过私钥生成公钥
ssh-keygen -y -t rsa -f $SSHPATH/id_rsa
```

这样子的话我们就创建了登录的公私钥，当然还要一些细节要处理，比如确认 `.ssh` 这个文件夹在不在。完整代码

```yaml
jobs:
  Build-and-deploy:
  	# ... 省略
    steps:
    	# ... 省略
      - name: Clone repository
      - name: Check out branch
      - name: Build
      - name: Setup ssh
      	run: |
        	SSHPATH="$HOME/.ssh"
          # 确认 .ssh 文件夹
          if [ ! -d "$SSHPATH" ]
          then
            mkdir $SSHPATH
          fi
          if [ ! -f "$SSHPATH/known_hosts" ]
          then
            touch $SSHPATH/known_hosts
          fi
          # 更新权限
          chmod 700 $SSHPATH
          chmod 600 $SSHPATH/known_hosts
          # 生成公钥
          echo "${{ secrets.SSH_PRIVATE_KEY }}" >> $SSHPATH/id_rsa
          chmod 600 $SSHPATH/id_rsa
          ssh-keygen -y -t rsa -f $SSHPATH/id_rsa
```

## 上传静态资源到 nginx

这一步同样比较简单，直接通过 scp 即可，上代码

```yaml
jobs:
  Build-and-deploy:
  	# ... 省略
    steps:
    	# ... 省略
      - name: Clone repository
      - name: Check out branch
      - name: Build
      - name: Setup ssh
      - name: Upload static resources
      	run: |
        	scp -o StrictHostKeyChecking=no -r ./static ${{ secrets.SSH_USERNAME}}@${{ secrets.SSH_HOST }}:/dest
```

## 利用 ssh 操作远程服务器

在 ssr 应用中，由于需要服务端渲染需要跑一个 Node 服务，我们通常通过 docker 等来操作

```yaml
jobs:
  Build-and-deploy:
  	# ... 省略
    steps:
    	# ... 省略
      - name: Clone repository
      - name: Check out branch
      - name: Build
      - name: Setup ssh
      - name: Upload static resources
      - name: Update server
      	run: |
        	scp -o StrictHostKeyChecking=no -r ./dist ${{ secrets.SSH_USERNAME }}@${{ secrets.SSH_HOST }}:/path/to/working-dir
          echo "
            cd /path/to/working-dir
            # --no-cache 禁止了 docker 构建的缓存，但是会导致构建时间加长
            docker-compose build --no-cache project
            docker-compose up -d project
          " | ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USERNAME }}@${{ secrets.SSH_HOST }}
```

<br />完整代码可查看: https://github.com/wokeyi/github-actions-deploy-ssr/blob/master/.github/workflows/deploy.yml