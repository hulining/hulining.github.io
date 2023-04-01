---
title: GitLab CI 使用 SSH keys
date: 2022-12-10 11:43:19
tags:
  - GitLab
  - CI/CD
categories:
  - CI/CD
abbrlink:
description: 本文章主要包含 GitLab CI 中使用 SSH keys 的方法
---

GitLab目前还没有在构建环境(GitLab Runner 运行的地方)中管理SSH密钥的内置支持。

在以下场景中，您可能需要使用 SSH密钥:

- 使用包管理器下载私有包。如 Bundler
- 将应用程序部署到您自己的服务器上。如 Heroku
- 从构建环境向远程服务器执行SSH命令
- 将文件从构建环境 Rsync 到远程服务器

目前使用最多的方法是通过扩展 `.gitlab-ci.yml` 将SSH密钥注入构建环境。它是一个适用于任何类型的 Runner (例如Docker或shell)的解决方案。

### 实现

1. 使用 `ssh-keygen` 在本地创建新的 ssh 密钥对
2. 添加 private key 中的内容到 Gitlab 仓库的环境变量中，将公钥复制到您想要访问的服务器上(一般为 `~/.ssh/authorized_keys` 文件中 )，或者如果您正在访问私有的GitLab存储库，则将其添加为[部署密钥](https://docs.gitlab.com/ee/user/project/deploy_keys/index.html)。
3. 在 job 执行期间运行  `ssh-agent` 加载私钥

可以在 `.gitlab-ci.yml` 中添加如下内容来运行 `ssh-agent` 加载私钥。

```yaml
before_script:
  ## 确认 ssh-agent 已经安装了，如果使用的是 rpm 格式的包，可以使用 yum 包管理器
  - 'command -v ssh-agent >/dev/null || ( apt-get update -y && apt-get install openssh-client -y )'
  #- 'command -v ssh-agent >/dev/null || ( yum update -y && yum install openssh-client -y )'
  
  ## 运行 ssh-agent (inside the build environment)
  - eval $(ssh-agent -s)
  
  ## 将保存在 SSH_PRIVATE_KEY 变量中的 SSH 密钥添加到 ssh-agent 中
  ## 我们使用 tr 删除行结束符，使 ed25519 加密密钥没有额外的 base64 编码: https://gitlab.com/gitlab-examples/ssh-private-key/issues/1#note_48526556
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -

  ## 创建 ~/.ssh 目录，并配置正确的权限
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  
  ## 可选的 git 配置
  # - git config --global user.email "user@example.com"
  # - git config --global user.name "User name"
  ##
  ## 如果配置了 SSH_KNOWN_HOSTS，则可以取消对下面两行的注释。
  - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts

  ##
  ## Alternatively, use ssh-keyscan to scan the keys of your private server.
  ## Replace example.com with your private server's domain name. Repeat that
  ## command if you have more than one server to connect to.
  ##
  # - ssh-keyscan example.com >> ~/.ssh/known_hosts
  # - chmod 644 ~/.ssh/known_hosts

  ##
  ## You can optionally disable host key checking. Be aware that by adding that
  ## you are susceptible to man-in-the-middle attacks.
  ## WARNING: Use this only with the Docker executor, if you use it with shell
  ## you will overwrite your user's SSH config.
  ##
  # - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >> ~/.ssh/config'
  


```

在 ssh 登录时有时需要对远程主机添加信任。GitLab 提供了以下3种方式，可以补充在 `.gitlab-ci.yml` 中以保证 ci 流程正确执行

```yaml
before_script:
  ## 方式1. 如果配置了 SSH_KNOWN_HOSTS，则可以使用如下方式完善 known_hosts 文件
  ## 其中 SSH_KNOWN_HOSTS 变量内容可 通过 `ssh-keyscan remote-server` 获得
  - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  
  ## 方式2. 直接执行 `ssh-keyscan remote-server` 将生成的密钥保存到 known_hosts 文件中
  # - ssh-keyscan example1.com >> ~/.ssh/known_hosts
  # - ssh-keyscan example2.com >> ~/.ssh/known_hosts
  # - chmod 644 ~/.ssh/known_hosts
  
  ## 方式3. 可以直接禁用主机密钥的校验。要注意这种方式容易受到中间人攻击
  ## 警告: 请只在 Docker executor 中使用这种方式，否则这种方式可能会修改本地的 ssh config
  # - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >> ~/.ssh/config'
```

### 参考

- [Using SSH keys with GitLab CI/CD](https://docs.gitlab.com/ee/ci/ssh_keys/)
