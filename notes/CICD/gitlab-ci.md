# Gitlab-CI

> [Get started with GitLab CI/CD](https://archives.docs.gitlab.com/17.8/ee/ci/index.html)
>
> [Gitlab v15.10.0 CI/CD.yaml Syntax Reference](https://archives.docs.gitlab.com/15.10/ee/ci/yaml/)
>
> [ssh](https://www.techtarget.com/searchsecurity/definition/Secure-Shell#:~:text=SSH%20(Secure%20Shell%20or%20Secure,that%20implement%20the%20SSH%20protocol.))
>
> [基于Gitlab的CI实践](https://moelove.info/2018/08/05/%E5%9F%BA%E4%BA%8E-GitLab-%E7%9A%84-CI-%E5%AE%9E%E8%B7%B5/)

## 介绍

### 术语

- `Pipeline`：宏观上作为串联整个CI/CD的流水线所定义的名词
- `Stage`：类似**命名空间**概念，`Stage`下可以有0至多个`Job`，每个`Job`可以并行执行，也可以在`.gitlab-ci.yml`自定义上下游`Job`执行过程间的关联关系、上游Job运行失败后是否继续运行等逻辑；[参考](#job to job)
- `Job`：用于执行你真正需要工作的脚本，每个`Job`独属于一个`Stage`，在`.gitlab-ci.yml`中定义`Job`时需指定`Stage`(未指定时Gitlab默认使用**test** stage)



- **[Runner](https://docs.gitlab.com/ci/runners/)**
  - `config.toml`
    - path in linux: `/etc/gitlab-runner/config.toml` 
  - [advanced config](https://gitlab.cn/docs/runner/configuration/advanced-configuration.html)
- **Executor**
  - 决定每个作业的运行环境
  - [docker executor](https://gitlab.cn/docs/runner/executors/docker.html)
  - [在 Docker 容器中运行 CI/CD 作业](https://gitlab.cn/docs/jh/ci/docker/using_docker_images.html#access-an-image-from-a-private-container-registry)

## 工作模式

### 架构

- [Gitlab Runner-Executor的工作模式](https://docs.gitlab.com/runner/#runner-execution-flow)
- [又拍云 CI/CD实践](https://opentalk-blog.b0.upaiyun.com/prod/2017-10-27/59c1c82f33c56c23422250ca45f52d31.pdf)
- [工作流](https://nexocode.com/blog/posts/understanding-principles-of-gitlab-ci-cd-pipelines/)

### 不同单元之间的联系

#### stage to stage

#### job to job

默认情况下没有直接的数据传递，但在编写脚本(.gitlab-ci.yml)中可通过如下关键字定义来共享数据:

#### difference between `artifacts` and `cache`

- `artifacts`

  - **reference**:
    - 

- `cache`

  - **reference**: 

    - [cache syntax in yaml](https://docs.gitlab.com/ci/yaml/#cache)
    - [Cache in Gitlab CI/CD](https://docs.gitlab.com/ci/caching/)
    - [use cache for caching go dependency](https://www.zhaowenyu.com/gitlab-doc/docs/213.html)

  - storage: be archive in a single `cache.zip` file,by default, the cache is stored on the machine where GitLab Runner is installed. The location also depends on the type of executor.

  - [archive/extract cache principle](https://docs.gitlab.com/ci/caching/#how-archiving-and-extracting-works)

    

    <img src="D:\笔记\Go\图床\image-20250407203158389.png" alt="image-20250407203158389" style="zoom:67%;" /><img src="D:\笔记\Go\图床\image-20250407203212238.png" alt="image-20250407203212238" style="zoom:67%;" />

`cache`存储依赖项，如软件包等，其下载的数据源存储在`Gitlab Runner上`

`artifacts`由`Job`生成，用于在`Stage`之间传递`build result`

##### 运行Job的前置条件



## .gitlab-ci.yml script

```
# 没有定义时，默认采用config.toml中[runners.docker]使用的executor镜像
image:

default:

stage:

job:

coverage:
```

### 实践

#### 脚本

> [单元测试覆盖率可视化](https://archives.docs.gitlab.com/15.10/ee/ci/testing/test_coverage_visualization.html#troubleshooting)

##### artifacts

- `reports`
  - `coverage_report`
  - `junit`
- 

####　使用私域镜像作为Job运行环境



#### 实现获取增量覆盖率

1. 用`go`+`git diff`编写脚本实现
2. 脚本中执行`git diff {branch} HEAD --unified=0 --diff-filter=ACMR`，`{branch}`由执行脚本时传入参数获取
3. 获取到系统差异文件内容后，遍历内容每一行，在遍历到`diff --git`前缀开头时，使用切片记录后续的由`@@ -n1,n2 +n3,n4 @@`开头的行；针对新增的只需要关注`n3,n4`
   1. `n3`：代表新增的首行
   2. `n4`：代表增量的行数，n3+n4得到结尾行
4. 获取系统mod名称，替换`diff --git`开头的行的末尾的文件路径
   1. 如`b/test/utils.go`，mod名称为`github.com/zhengyb/task`，则文件名称替换为`github.com/zhengyb/task/test/utils.go`
   2. 作这个操作主要是为了后续遍历`coverage.out`时判断文件名称进行匹配
5. `diff`内容遍历完后，得到的结构是一个`map`
   1. `key`: 第4步操作里得到的文件名称
   2. `value`: 切片，切片元素里的数据保存了起始行和末尾行（设置切片类型是由于`diff`内容里一个文件可能包含多条`@@ -n1,n2 +n3,n4 @@`行）
6. 使用`golang.org/x/tools/cover`包解析`coverage.out`文件，这个覆盖率文件是在`CI`时执行`系统测试的job`生成的`artifact`，执行`增量测试覆盖率job`会通过`needs`指定`系统测试的job`完成后获取该`artifact`

<img src="D:\笔记\Go\图床\image-20250521164810497.png" alt="image-20250521164810497" style="zoom:50%;" />

<img src="D:\笔记\Go\图床\image-20250521164314617.png" alt="image-20250521164314617" style="zoom:50%;" />

## QA

1. commit在push到repo后，Gitlab server触发Runner执行Pipeline的过程是怎么样的，即Gitlab server、Gitlab Runner、Gitlab Executor的交互以及交互过程中双方传递的是什么数据，并且在执行Pipeline的Job中，每个stage下的job之间有什么关联吗 以及不同stage之间有数据传递吗

   - Runner: self-managed Runner(docker)
   - Executor: docker type

   ```
   4. 交互流程
   4.1 Commit 推送到仓库
   用户推送代码:
   
   用户将本地更改推送到 GitLab 仓库，触发 commit。
   例如，使用 git push origin main。
   GitLab Server 检测到新 commit:
   GitLab Server 通过 Webhook 或内置机制检测到新 commit 推送。
   
   Pipeline 创建:
   GitLab Server 读取项目根目录下的 .gitlab-ci.yml 文件。根据 .gitlab-ci.yml 文件定义的 stages 和 jobs，生成一个 Pipeline。记录 Pipeline 的基本信息（如 ID、触发人、触发时间等）。
   
   4.2 Pipeline 分配 Job
   任务分配:
   GitLab Server 查找可用的 Runner，并将生成的 Job 分配给合适的 Runner。
   任务分配基于 Runner 的标签、可用性等因素。
   
   Job 传递给 Runner:
   GitLab Server 通过 HTTP API 将 Job 的详细信息发送给 Runner。传递的数据包括：
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
   GitLab Runner 从 GitLab Server 接收到分配的 Job 信息。Runner 解析 Job 信息，确定所需的环境配置和执行步骤。
   
   4.4 准备 Docker 环境:
   Docker Executor 根据 Job 定义的镜像（例如 image: golang:latest）创建一个新的 Docker 容器。容器会挂载必要的卷（如缓存目录、密钥等），并设置环境变量。容器会克隆项目的 Git 仓库、检出对应的 commit 或 branch。
   
   4.5 执行 Job 脚本:
   在准备好的 Docker 环境中，Docker Executor 执行 Job 定义的脚本步骤。例如，编译代码、运行测试、构建部署包等。执行过程中，会将日志发送回 GitLab Server，以便用户可以实时查看日志。
   
   4.6 Job结果报告:
   Job 执行完成后，Docker Executor 将 Job 的结果（成功/失败）以及日志通过 HTTP API 发送回 GitLab Server。
   GitLab Server 更新 Pipeline 和 Job 的状态，并记录执行时间、日志等信息。
   
   4.7 清理资源:
   Docker Executor 停止并删除用于执行 Job 的 Docker 容器，释放资源。
   ```

   

2. Executor的Docker环境与Job的Docker环境的区别

3. go module包管理执行问题

4. `fatal: shallow file has changed since we read it`

   1. 在`runner's config.toml`中，`concurrent`=0，导致多个job并行执行时同时使用了同一个`executor`,修改`concurrent`=4(最大数)

5. 在单条流水线的多个Job中都执行覆盖率计算，导致MR界面看到的是这些job的覆盖率的平均值：[官方解释](https://gitlab.com/gitlab-org/gitlab/-/issues/15399)



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

​	
