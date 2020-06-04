---
title: GitLab CI/CD 管道配置详解
tags:
  - GitLab
  - CI/CD
categories:
  - CI/CD
abbrlink: 3551
description: 本文章主要包含 GitLab CI/CD 中 .gitlab-ci.yml 文件中可使用的关键字语法及用法详解.
---

在每个 GitLab 项目中,使用名为 `.gitlab-ci.yml` 的 YAML 文件配置 GitLab CI/CD  pipelines (管道).

`.gitlab-ci.yml` 文件定义 pipeline 的结构和执行顺序,并确定了如下内容

- GitLab Runner(负责运行管道的实例)执行什么任务
- GitLab Runner 的高级配置,可用于配置 GitLab Runner

有关 `.gitlab-ci.yml` 文件示例参考可从如下位置找到

- [GitLab CI/CD Examples](https://docs.gitlab.com/ee/ci/examples/README.html)
- [gitlab 的 .gitlab-ci.yml 文件](https://gitlab.com/gitlab-org/gitlab/blob/master/.gitlab-ci.yml)

## 介绍

pipeline 配置从 job(作业)开始.作业是组成 `.gitlab-ci.yml` 的最基本元素.它主要有下功能或特性

- 定义约束,指出应在什么条件下执行它们
- 任意名称的顶级元素,至少包含 `script` 子句
- 可以定义多个

简单示例如下

```yaml
job1:
  script: "execute-script-for-job1"

job2:
  script: "execute-script-for-job2"
```

以上两个示例是具有两个单独作业的 CI/CD 配置,每个作业执行一个命令.当然,此命令可以在存储库中直接执行代码或运行脚本.

作业由 Runner 提取并在 Runner 系统环境中运行.每个作业彼此独立运行.

### 验证 `.gitlab-ci.yml`

每个 GitLab CI/CD 实例都有一个成为 Lint 的嵌入式调试工具,该工具可验证 `.gitlab-ci.yml` 文件的内容,您可以在项目名称空间的 `$PATH_TO_PROJECT/-/ci/lint`($PATH_TO_PROJECT=$GitLab_URL/user_or_group_name/project_name)路径下找到.如 `https://gitlab.example.com/gitlab-org/project-123/-/ci/lint`.

### 作业命名规范

每个作业都有唯一的名称,且以下保留关键字不能作为作业的名称:

```text
image
services
stages
types
before_script
after_script
variables
cache
include
```

## 配置参数

作业通过一系列参数定义作业的行为.以下列出了作业的可用参数:

关键字 | 描述
:---: | :---:
`script` | 由 Runner 执行的脚本
`image` |  | 使用 docker 镜像.也可用 `image:name`, `image:entrypoin`
`services` | 使用 docker 服务镜像.也可用 `services:name`, `services:alias`, `services:entrypoint`, `services:command`
`before_script` | 包含执行作业之前的一组命令
`after_script` | 包含执行作业之后的一组命令
`stage` | 定义作业的阶段(默认值为 `test`)
`only` | 限定作业何时创建.也可用 `only:refs`, `only:kubernetes`, `only:variables`, `only:changes`
`except` | 限定作业不创建.也可用  `except:refs`, `except:kubernetes`, `except:variables`, `except:changes`
`rules` | 根据作业的属性评估确定是否创建工作的条件列表.不得与 `only/except` 一起使用
`tags` | 用于选择 Runner(运行器) 的标签列表
`allow_failure` | 是否允许作业失败
`when` | 限定什么时候开始作业.也可用`when:manual`, `when:delayed`
`environment` | 作业部署到的环境名称.也可用 `environment:name`, `environment:url`, `environment:on_stop`, `environment:auto_stop_in`, `environment:action`
`cache` | 缓存文件列表,以备在后续的作业中使用.也可用 `cache:paths`, `cache:key`, `cache:untracked`, `cache:policy`
`artifacts` | 成功时附加到作业的文件和目录列表
`dependencies` | 指定本次作业的依赖作业列表,该列表中作业由 `artifacts` 附加的文件或目录会作为本次作业的依赖
`coverage` | 提供正则表达式,从作业输出中提取指定的内容
`retry` | 出现问题时可自动重试作业的条件和次数
`timeout` | 定义作业级别的超时时间
`parallel` | 可并行运行的作业实例个数
`trigger` | 定义下游管道触发器
`include` | 此作业包含的外部 yaml 文件.也可用 `include:local`, `include:file`, `include:template`, `include:remote`
`extends` | 该作业要继承的配置条目
`pages` | 上传作业结果,用于在 GitLab Pages 展示
`variables` | 定义作业级别的变量
`interruptible` | 定义新的作业运行是否可以取消作业
`resource_group` | 限制作业并发

## 全局参数

可以在全局级别定义一些参数,这会影响管道中的所有作业.

以下参数可以使用 `default` 关键字配置块将某些全局参数设置为所有作业的默认值.作业可以覆盖这些默认值.

```text
image
services
before_script
after_script
tags
cache
artifacts
retry
timeout
interruptible
```

示例如下,默认使用 `image: ruby:2.5` 镜像,而 `rspec 2.6` 作业使用 `image: ruby:2.6` 镜像

```yaml
default:
  image: ruby:2.5

rspec:
  script: bundle exec rspec

rspec 2.6:
  image: ruby:2.6
  script: bundle exec rspec
```

### `stages`

`stages` 用于定义使用的阶段作业,且是全局的

`stages` 允许具有灵活的多级管道,`stages` 中的元素定义了作业执行的顺序:

- 同一阶段的作业并行运行
- 前一阶段的作业成功运行后,才运行下一阶段作业

还有两种情况需要注意:

- 如果 `.gitlab-ci.yml` 中未定义 `stages`,默认运行 `build`, `test`, `deploy` 阶段的作业
- 如果作业没有指定阶段,则默认将该作业分配为 `test` 阶段

### `include`

`include` 关键字用于包含外部 YAML 文件,可将 CI/CD 配置分解为多个文件,提高配置文件的可读性.

`include` 要求外部 YAML 文件具有 `.yml` 或 `.yaml` 后缀名,否则不会被导入

`include` 支持如下方法,默认方法是 `local`.在进行 include 配置时支持直接引入对应文件路径,`include` 会自动分析方法. [示例](https://docs.gitlab.com/ee/ci/yaml/includes.html)

方法 | 描述
:---: | :---
`local` | 本项目代码库中的文件
`file` | 其它项目代码库中的文件
`remote` | 远程 URL 的文件
`template` | GitLab 提供的模版

#### `include:local`

`include:local` 用于引入与 `.gitlab-ci.yml` 来自同一代码库同一分支的文件,请确保 `.gitlab-ci.yml` 与要引入的文件在同一分支上

`include:local` 使用代码仓库 `/` 根路径进行引用.

```yaml
include:
  - local: '/templates/.gitlab-ci-template.yml'

include: '.gitlab-ci-production.yml'
```

#### `include:file`

`include:file` 支持引入同一个 GitLab 实例中另一个代码仓库中的文件. 并使用代码仓库 `/` 根路径进行引用.

您也可以通过 `ref` 指定代码仓库的分支或 `HEAD` 以确保引入正确仓库分支中的文件

```yaml
include:
  - project: 'my-group/my-project'
    ref: master
    file: '/templates/.gitlab-ci-template.yml'

  - project: 'my-group/my-project'
    ref: v1.0.0
    file: '/templates/.gitlab-ci-template.yml'

  - project: 'my-group/my-project'
    ref: 787123b47f14b552955ca2786bc9542ae66fee5b # Git SHA
    file: '/templates/.gitlab-ci-template.yml'
```

#### `include:remote`

`include:remote` 支持通过 HTTP/HTTPS 引入来自其他位置中的文件,远程文件必须可通过 GET 请求公开访问

远程文件使用完整 URL 引用.

```yaml
include:
  - remote: 'https://gitlab.com/awesome-project/raw/master/.gitlab-ci-template.yml'
```

#### `include:template`

`include:template` 支持引入 GitLab 提供的 `.gitlab-ci.yml` 模版文件

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml
```

## 参数详解

### `image`

指定一个 Docker 镜像来运行作业.详见[使用 Docker 镜像](https://docs.gitlab.com/ee/ci/docker/using_docker_images.html#define-image-and-services-from-gitlab-ciyml)

### `script`

`script` 是脚本所必须的关键字,它是由 Runner 执行的 shell 脚本.支持使用包含多个命令的数组以便顺序执行命令

```yaml
job:
  script:
    - uname -a
    - bundle exec rspec
```

如果有命令的退出代码不为 0,则作业将失败,且不会执行其他命令.通过将退出代码存储在变量中,可以避免这种情况:

```yaml
job:
  script:
    - false || exit_code=$?
    - if [ $exit_code -ne 0 ]; then echo "Previous command failed"; fi;
```

### `before_script` 和 `after_script`

`before_script` 用于定义在每个作业(包括部署作业)之前运行的命令,`after_script` 用于定义在每个作业(包括失败的作业)之后运行的命令.它们都是一个数组

`before_script` 中指定的脚本与主脚本中指定的脚本串联在一起,并在单个 shell 中一起执行

`after_script` 中指定的脚本在新的shell中执行,与任何 `before_script` 或 `script` 定义的命令分开.

它们有如下需要注意的地方:

- 当前工作目录设置为默认目录
- 无法访问 `before_script` 或 `script` 定义的脚本完成的修改.
  - `script` 中定义的命令别名和变量
  - 由 before_script 或 `script` 安装的软件
- 独立的超时时间,默认为 5 分钟
- 不影响作业的退出代码.如果 `script` 成功,但 `after_script` 超时或失败,作业将标记为成功作业

### `stage`

`stage` 是按照每个作业定义的,且依赖于全局定义的 `stages`. 它将作业分为不同的阶段,同一阶段的作业可以并行执行

```yaml
stages:
  - build
  - test
  - deploy

job 0:
  stage: .pre
  script: make something useful before build stage

job 1:
  stage: build
  script: make build dependencies

job 2:
  stage: build
  script: make build artifacts

job 3:
  stage: test
  script: make test

job 4:
  stage: deploy
  script: make deploy

job 5:
  stage: .post
  script: make something useful at the end of pipeline
```

默认情况下,GitLab Runner 一次仅运行一个作业.可通过设置作业在不同的 Runner 上运行或修改 Runner 的 `concurrent` 配置来使作业并行运行.

> (GitLab 12.4 新增)无论在 `stages` 中 顺序如何,`.per` 始终是管道中的第一个阶段.`.post` 始终是管道中的最后一个阶段.用户定义的阶段在这两个阶段之间运行

### `extends`

`extends` 定义使用 `extends` 关键字的作业(子作业)要继承的作业(父作业).`extends` 会将子作业内容递归合并到父作业中

```yaml
.tests:
  script: rake test
  stage: test
  only:
    refs:
      - branches

rspec:
  extends: .tests
  script: rake rspec
  only:
    variables:
      - $RSPEC

# 等价于
rspec:
  script: rake rspec  # 作业中相同关键字的 key 已经被重写
  stage: test
  only:
    refs:             # 子作业中没有的关键字会继承父作业中关键字及其值
      - branches
    variables:
      - $RSPEC
```

且如果 `extends` 包含多个值,则会按照 `extends` 中定义的作业列表顺序进行继承.如果作业列表有相同的关键字,则后面作业中关键字的值会覆盖前面作业关键字的值.

```yaml
.only-important:
  only:
    - master
    - stable
  tags:
    - production

.in-docker:
  tags:
    - docker
  image: alpine

rspec:
  extends:
    - .only-important
    - .in-docker
  script:
    - rake rspec

# 等价于
rspec:
  only:
    - master
    - stable
  tags:
    - docker
  image: alpine
  script:
    - rake rspec
```

### `rules`

> GitLab 12.3 新增

`rules` 将按照顺序评估列表中的规则,直到有规则匹配到,实现为作业动态提供属性的功能.`rules` 不能与 `only/except` 同时使用,它旨在替换该功能

可用的规则字句字段包括

- `if`(与 `only:variables` 类似)
- `changes` (与 `only:changes` 类似)
- `exists`

#### `rules:if`

`rules:if` 使用定义的 if 规则设置作业执行的时机

`rules:if` 与 `only:variables` 略有不同.`only:variables` 仅接收单个表达式字符串,而 `rules:if` 支持表达式字符串数组形式,支持多个条件判断.支持使用 `&&` 或 `||` 将表达式集合组合为一个表达式,且支持使用变量匹配语法.

如果提供的规则均不匹配,则作业将被默认设置为 `when:never`,且不包含在执行作业的管道中.如果配置中不包含 `rules:when`,则继承 `job:when` 的值,默认为 `on_success`.

```yaml
job:
  script: "echo Hello, Rules!"
  rules:
    - if: '$NAME =~ /^always/ # 如果变量 NAME 匹配 always,则设置 when: always
      when: always
    - if: '$NAME =~ /^manual/ # 如果变量 NAME 匹配 manual,则设置为 when: manual
      when: manual
    - if: '$NAME'  # 如果变量 NAME 设置了,且不为空,则设置为 when: on_success

    # 如果以上均没有匹配,则设置 when: never
```

#### `rules:changes`

`rules:changes` 与 `only:changes` 和 `except:changes` 工作方式相同,它接受路径的数组,并监控它.如果没有对监控中的文件路径做修改,则不会返回 true

```yaml
docker build:
  script: docker build -t my-image:$CI_COMMIT_REF_SLUG .
  rules:
    - changes: # 如果 Dockerfile 文件被修改,则将其设置为 `when: manual`
      - Dockerfile
      when: manual
```

#### `rules:exists`

> GitLab 12.4 新增

`rules:exists` 接受文件路径或文件路径匹配的数组.如果其中任何一个文件路径在代码仓库中存在,则将匹配,作业将在管道中运行

```yaml
job:
  script: bundle exec rspec
  rules:
    - exists:
      - spec/**.rb
```

#### `rules:allow_failure`

> GitLab 12.8 新增

您可以在规则中使用 `allow_failure: true`,在不停止作业管道的情况下允许作业失败或手动作业等待操作.所有使用规则的作业默认为 `allow_failure: false`.

### `only/except`

`only` 和 `except` 是用于限制什么时候创建作业的两个参数.`only` 用于定义要运行的作业的分支和标签.`except` 恰好相反,用于定义不运行的作业的分支和标签.

- 它支持正则表达式匹配.
- 它支持直接指定存储库路径以便于过滤仓库 fork 的作业

`only` 和 `except` 允许使用如下关键字,且如果有作业没有关键字限定,则 `only` 默认为 `['branches', 'tags']`,`except` 默认为空.

值 | 描述
:---: | :---
`branches` | 当管道的 git 引用是分支时
`tags` | 当管道的 git 引用是标签时
`external` | 当时用 GitLab 以外的 CI 服务时
`pushes` | 由用户的 git push 触发
`schedules` | 用于自动调度的管道
`triggers` | 使用触发令牌创建的管道
`web` | 对于在 GitLab UI 中使用 "Run" 按钮创建的管道
`merge_requests` | 在创建或更新合并请求时

示例如下:

```yaml
job1:
  only:
    - /^issue-.*$/i  # 使用正则表达式,`i` 表示忽略大小写
  except:
    - branches  # 使用关键字,所有的分支都将被跳过

job2:
  only:
    - branches@gitlab-org/gitlab  # 仅对 branches 仓库执行作业,而不对其 fork 执行
  except:
    - master@gitlab-org/gitlab
    - /^release/.*$/@gitlab-org/gitlab
```

- `only:refs/except:refs`: 语法与上述一致
- `only:kubernetes/except:kubernetes`: 仅接受 `active` 参数,表示仅当 kubernetes 服务启动时才执行此作业
- `only:variables/except:variables`: 接受自定义变量表达式,以此决定是否创建作业
- `only:changes/except:changes`: 接受文件路径,并监控是否改变,以此决定是否创建作业

单个关键字中定义的条件列表是**或**的关系,有一个条件满足,则可以认为该关键字表示为 true.

对于 `only` 创建作业来说,以上关键字之间是**与**的关系,当定义的以上关键字均为 true 时候,才会创建作业.对于 `except` 不创建作业来说,以上关键字之间是**或**的关系,当定义的以上关键字任意一个为 true 时候,则不会创建作业.

示例如下:

```yaml
# 该作业在以下条件均满足时创建
# - 管道是被调度的或在 `master` 分支上运行
# - $CI_COMMIT_MESSAGE 变量正则匹配 /run-end-to-end-tests/
# - kubernetes 服务运行
test:
  script: npm run test
  only:
    refs:
      - master
      - schedules
    variables:
      - $CI_COMMIT_MESSAGE =~ /run-end-to-end-tests/
    kubernetes: active
```

```yaml
# 该作业任意条件满足时不创建
# - 管道是被调度的或在 `master` 分支上运行
# - README.md 文件被修改
test:
  script: npm run test
  except:
    refs:
      - master
    changes:
      - "README.md"
```

### `needs`

`needs` 关键字允许不按照 stages 顺序执行作业,而按照依赖顺序执行作业.这样无需考虑阶段顺序,可以让多个阶段同时运行.

示例:

```yaml
linux:build:
  stage: build

mac:build:
  stage: build

lint:
  stage: test
  needs: []

linux:rspec:
  stage: test
  needs: ["linux:build"]

linux:rubocop:
  stage: test
  needs: ["linux:build"]

mac:rspec:
  stage: test
  needs: ["mac:build"]

mac:rubocop:
  stage: test
  needs: ["mac:build"]

production:
  stage: deploy
```

- Linux 作业线: `linux:build` 作业完成后,将立即运行 `linux:rspec` 和 `linux:rubocop` 作业,而无需等待 `mac:build` 结束
- macOS 作业线: `mac:build` 作业完成后,将立即运行 `mac:rspec` 和 `mac:rubocop` 作业,而无需等待 `linux:build` 结束
- production 作业会等待 `deploy` 阶段之前的作业完成后再运行

### `tags`

`tags` 用于从允许运行该项目的所有 Runner 列表中选择特定的 Runner.在注册 Runner 时,您可以指定 Runner 的标签,例如 ruby,postgres,development 等.

`tags` 也是在不同平台上运行不同作业的好方法.例如，给定带有 `osx` 标签的 OS X Runner 和带有 `windows` 标签的 Windows Runner,以下作业将在各自的平台上运行:

```yaml
windows job:
  stage:
    - build
  tags:
    - windows
  script:
    - echo Hello, %USERNAME%!

osx job:
  stage:
    - build
  tags:
    - osx
  script:
    - echo "Hello, $USER!"
```

### `allow_failure`

`allow_failure` 关键字用于指定作业可以失败,而不会影响其余的作业.

设置为 true 后,如果该作业失败，该作业将在用户界面中显示橙色警告,管道的逻辑流程将认为作业成功/通过,并且不会被阻塞

### `when`

`when` 关键字用于在发生故障或发生故障时运行的作业.支持以下值

- `on_success`: 前面阶段的所有作业都成功时,才执行作业.是默认值
- `on_failure`: 仅在前面阶段至少一个作业失败时才执行作业.
- `always`: 总是执行作业,不管前面作业的执行状态
- `manual`: 手动执行作业时才执行此作业
- `delayed`: 一段时间后执行此作业.需要设置 `start_in` 关键字,并设置 **1 周**之内的延迟时间,用于表示在指定延迟时间后开始作业.支持 seconds,minutes,hours,days,week 单位,默认单位是 seconds

### 其它常用

- `retry` 允许你在失败的情况下重试作业的次数.重试值必须是一个整数,等于或大于0,但小于等于2
- `timeout` 设置指定作业的超时时间,
- `parallel` 配置并行运行的作业实例,名称为 `job_name 1/N` 到 `job_name N/N` (2<=N<=50)
- `trigger` 用于定义触发的下游管道的触发器.当定义 `trigger` 的作业完成时,将触发下一个作业的执行
- `variables` 用于在 `.gitlab-ci.yml` 中定义变量,然后在作业中使用.
