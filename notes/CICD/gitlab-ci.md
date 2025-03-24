# Gitlab-CI

> [Get started with GitLab CI/CD](https://archives.docs.gitlab.com/17.8/ee/ci/index.html)
>
> [Gitlab latest CI/CD.yaml Syntax Reference](https://archives.docs.gitlab.com/17.8/ee/ci/yaml/index.html)
>
> [Gitlab v15.10.0 CI/CD.yaml Syntax Reference](https://archives.docs.gitlab.com/15.10/ee/ci/yaml/)
>
> [在容器中运行GItlab Runner](https://gitlab.cn/docs/runner/install/docker.html)
>
> [自定义执行器](https://gitlab.cn/docs/runner/executors/custom.html)
>
> [ssh](https://www.techtarget.com/searchsecurity/definition/Secure-Shell#:~:text=SSH%20(Secure%20Shell%20or%20Secure,that%20implement%20the%20SSH%20protocol.))

我遇到的同样的问题：https://gitlab.com/gitlab-org/gitlab/-/issues/431258

找寻gitlab-runner config.toml文件：sudo find /srv /etc -type f -path '*gitlab-runner*' -name 'config.toml'

报告支持类型：https://archives.docs.gitlab.com/15.10/ee/ci/yaml/artifacts_reports.html#artifactsreportscoverage_report 

## 介绍

### 术语

- Pipeline
- Stage
- Job
- Runner
- Executor

### 不同单元之间的联系

#### stage to stage

#### job to job

默认情况下没有直接的数据传递，但在编写脚本(.gitlab-ci.yml)中可通过如下关键字定义来共享数据:

- `artifacts`
- `cache`

##### difference between `artifacts` and `cache`

`cache`存储依赖项，如软件包等，其下载的数据源存储在`Gitlab Runner上`

`artifacts`由Job生成，用于在Stage之间传递`build result`

## .gitlab-ci.yml script

```

```

## QA

1. Q: 在Gitlab CI/CD中，我使用的Gitlab Runner是docker创建的self-managed Runner，同时Executor也是通过Docker创建的，我想知道commit在push到repo后，Gitlab server触发Runner执行Pipeline的过程是怎么样的，即Gitlab server、Gitlab Runner、Gitlab Executor的交互以及交互过程中双方传递的是什么数据，并且在执行Pipeline的Job中，每个stage下的job之间有什么关联吗 以及不同stage之间有数据传递吗

   ```
   A: 
   4. 交互流程
   4.1 Commit 推送到仓库
   用户推送代码:
   
   用户将本地更改推送到 GitLab 仓库，触发 commit。
   例如，使用 git push origin main。
   GitLab Server 检测到新 commit:
   
   GitLab Server 通过 Webhook 或内置机制检测到新 commit 推送。
   Pipeline 创建:
   
   GitLab Server 读取项目根目录下的 .gitlab-ci.yml 文件。
   根据 .gitlab-ci.yml 文件定义的 stages 和 jobs，生成一个 Pipeline。
   记录 Pipeline 的基本信息（如 ID、触发人、触发时间等）。
   4.2 Pipeline 分配 Job
   任务分配:
   
   GitLab Server 查找可用的 Runner，并将生成的 Job 分配给合适的 Runner。
   任务分配基于 Runner 的标签、可用性等因素。
   Job 传递给 Runner:
   
   GitLab Server 通过 HTTP API 将 Job 的详细信息发送给 Runner。
   传递的数据包括：
   Job ID
   项目 ID 和名称
   Git 仓库地址
   Branch 或 Tag 名称
   Commit ID
   环境变量
   作业脚本（执行步骤）
   所需的构建镜像（Docker 镜像）
   其他相关信息
   4.3 Runner 执行 Job
   Runner 接收 Job:
   
   GitLab Runner 从 GitLab Server 接收到分配的 Job 信息。
   Runner 解析 Job 信息，确定所需的环境配置和执行步骤。
   准备 Docker 环境:
   
   Docker Executor 根据 Job 定义的镜像（例如 image: golang:latest）创建一个新的 Docker 容器。
   容器会挂载必要的卷（如缓存目录、密钥等），并设置环境变量。
   容器会克隆项目的 Git 仓库、检出对应的 commit 或 branch。
   执行 Job 脚本:
   
   在准备好的 Docker 环境中，Docker Executor 执行 Job 定义的脚本步骤。
   例如，编译代码、运行测试、构建部署包等。
   执行过程中，会将日志发送回 GitLab Server，以便用户可以实时查看日志。
   Job 结果报告:
   
   Job 执行完成后，Docker Executor 将 Job 的结果（成功/失败）以及日志通过 HTTP API 发送回 GitLab Server。
   GitLab Server 更新 Pipeline 和 Job 的状态，并记录执行时间、日志等信息。
   清理资源:
   
   Docker Executor 停止并删除用于执行 Job 的 Docker 容器，释放资源。
   5. Pipeline 结构与 Job 关联
   5.1 Stages 和 Jobs
   Stages: 定义了一个 Pipeline 的不同阶段，每个阶段可以包含一个或多个 Jobs。
   Jobs: 在每个 Stage 中定义具体的构建、测试、部署任务。
   关联: Stage 之间按顺序执行，每个 Stage 的所有 Jobs 完成后才开始下一个 Stage。
   ```

   

2. Executor的Docker环境与Job的Docker环境的区别

3. 



go help testflag

-cover
            Enable coverage analysis.
            Note that because coverage works by annotating the source
            code before compilation, compilation and test failures with
            coverage enabled may report line numbers that don't correspond
            to the original sources.

 -covermode set,count,atomic
            Set the mode for coverage analysis for the package[s]
            being tested. The default is "set" unless -race is enabled,
            in which case it is "atomic".
            The values:
                set: bool: does this statement run?
                count: int: how many times does this statement run?
                atomic: int: count, but correct in multithreaded tests;
                        significantly more expensive.
            Sets -cover.

-coverpkg pattern1,pattern2,pattern3
            Apply coverage analysis in each test to packages matching the patterns.
            The default is for each test to analyze only the package being tested.
            See 'go help packages' for a description of package patterns.
            Sets -cover.

- Executor拉取代码时有远程仓库代码依赖问题

- 错误

  ```
  Host key verification failed.
  	fatal: Could not read from remote repository.
  	
  	Please make sure you have the correct access rights
  	and the repository exists.
  ```

  

  ```
  Warning: Permanently added the RSA host key for IP address '' to the list of known hosts. git@: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password). fatal: Could not read from remote repository.
  ```

  ```
  @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
  @       WARNING: POSSIBLE DNS SPOOFING DETECTED!          @
  @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
  The ECDSA host key for git.wondershare.cn has changed,
  and the key for the corresponding IP address 192.168.13.157
  is unknown. This could either mean that
  DNS SPOOFING is happening or the IP address for the host
  and its host key have changed at the same time.
  @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
  @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
  @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
  IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
  Someone could be eavesdropping on you right now (man-in-the-middle attack)!
  It is also possible that a host key has just been changed.
  The fingerprint for the ECDSA key sent by the remote host is
  SHA256:wN0v82FBqdcZVkxK7fBDHfDMleryKrY8Y4SldZZE+gU.
  Please contact your system administrator.
  Add correct host key in /root/.ssh/known_hosts to get rid of this message.
  Offending ECDSA key in /root/.ssh/known_hosts:13
  ECDSA host key for git.wondershare.cn has changed and you have requested strict checking.
  Host key verification failed.
  
  
  # 解决方式
  ssh-keygen -f "/root/.ssh/known_hosts" -R git.wondershare.cn
  ```

  

## SSH

### Part of SSH

- sshd
- ssh-agent
- ssh-keyscan

### Encrypt Algorithm

- RSA \ DSA \ ECDSA \ ED22519

### Command

```
ssh hostname/IP




```

