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

- `Stage`：stage按顺序执行，某个stage失败时后续的stage不会执行

  ​		`stage`下可以有0至多个`job`，既能并行执行，也可以在`.gitlab-ci.yml`自定义上下游`Job`执行过程间的关联关系、上游Job运行失败后是否继续运行等逻辑

- `Job`：流水线执行任务的最小执行单元，用于执行你真正需要工作的脚本，每个`job`独属于一个`stage`

  ​	     在`.gitlab-ci.yml`中定义`job`时需指定`stage`(未指定时Gitlab默认使用**test** stage)

<img src="D:\笔记\Go\图床\image-20250705154750899.png" style="zoom: 67%;" />

```yaml
# .gitlab-ci.yaml
stages:
  - init
  - test
  
job:unit_test:
  stage: test
  ...
  
job:gen_ut_report:
  stage: test
  needs:
    - job:unit_test
  ...

job:gen_increment_ut_report:
  stage: test
  needs:
    - job:unit_test
    - job:gen_ut_report
  ...
```



- **[`runner`](https://docs.gitlab.com/ci/runners/)**
  - `runner`可以简易理解为`gitlab server's client`，定期轮询并主动接收`gitlab server`派发的`job`，创建相应配置的`executor`执行任务，并在job执行完毕后上传执行过程中生成的`artifact(构建产物)`
  - `config.toml`
    - 配置文件路径: `/etc/gitlab-runner/config.toml` 
  - [advanced config](https://gitlab.cn/docs/runner/configuration/advanced-configuration.html)
- `executor`
  - 由`runner`在服务运行时动态生成，作为每个`job`执行脚本的运行环境
  - [executor of docker deployment](https://gitlab.cn/docs/runner/executors/docker.html)
  - [在 Docker 容器中运行 CI/CD 作业](https://gitlab.cn/docs/jh/ci/docker/using_docker_images.html#access-an-image-from-a-private-container-registry)

## 工作模式

### 架构

- [Gitlab Runner-Executor的工作模式](https://docs.gitlab.com/runner/#runner-execution-flow)
- [又拍云 CI/CD实践](https://opentalk-blog.b0.upaiyun.com/prod/2017-10-27/59c1c82f33c56c23422250ca45f52d31.pdf)
- [工作流](https://nexocode.com/blog/posts/understanding-principles-of-gitlab-ci-cd-pipelines/)

![](D:\笔记\Go\图床\image-20250705160215826.png)

## 编写gitlab-ci.yaml脚本

1. `stage`定义、`Job`间的执行关系

   - 设想需要多少个`satge`与边界?  系统性单元测试、ci-lint(检测服务代码标准的工具)

2. 自定义`Job`的执行镜像

   - 按需编写`Dockerfile`，上传到`harbor`

     ```dockerfile
     FROM golang:1.18
     
     # 配置公司内部环境相关的变量、系统性单元测试所需的执行文件
     RUN go env -w GOPRIVATE='git.wondershare.cn/*' &&  \
         go env -w GOINSECURE='git.wondershare.cn' &&  \
         go env -w GO111MODULE=on && \
         go env -w GOPROXY='https://goproxy.cn,direct' && \
         go install gotest.tools/gotestsum@v1.12.0 && \
         go install github.com/boumenot/gocover-cobertura@latest && \
         go install github.com/zyb-992/go-diff@latest && \
         git config --global url."git@git.wondershare.cn:".insteadOf "http://git.wondershare.cn/"
     ```

     <img src="D:\笔记\Go\图床\image-20250705161343584.png" alt="image-20250705161343584" style="zoom:50%;" />

   - 在`runner`容器中配置拉取镜像需要授权登录`harbor`的环境变量

     ```shell
     # 进行runner 容器
     [root@sz_sjzx_dev01_17_33:/home/sudo_root]# docker ps 
     CONTAINER ID   IMAGE                           COMMAND                  CREATED        STATUS        PORTS     NAMES
     503f45209c32   gitlab/gitlab-runner:v15.10.0   "/usr/bin/dumb-init …"   3 months ago   Up 3 months             qc-gitlab-runner
     
     [root@sz_sjzx_dev01_17_33:/home/sudo_root]# docker exec -it 503f45209c32 bash
     
     root@503f45209c32:/# cat /etc/gitlab-runner/config.toml
     # 配置文件 关注environment
     ...
     environment = ["DOCKER_AUTH_CONFIG={ \"auths\": { \"harbor.300624.cn\": { \"auth\": \"amVua2luczpZaGxCOEEwRnFiQXVTejdzTWM=\" } } }"]
     ...
     ```

   - 脚本中各个`job`指定镜像名称

     ```yaml
     variables:
       UNIT_TEST_IMAGE: harbor.300624.cn/base/qc-unit-test-go:v1
     
     job:unit_test:
       stage: test
       image: $UNIT_TEST_IMAGE
     ```

3. 脚本编写与制品上传

   ```yaml
   stages:
     - init
     - test
   
   # 全局变量
   variables:
     UNIT_TEST_IMAGE: harbor.300624.cn/base/qc-unit-test-go:v1
     INCREMENT_COVERAGE_THRESHOLD: 80
   
   # 缓存锚点
   .go-mod-cache:
     variables:
       GOPATH: "${CI_PROJECT_DIR}/.go"
     # 先extract cache(cache.zip), 再执行before_script
     before_script:
       - mkdir -p .go
     cache:
       key:
         # 根据前缀和文件内容哈希生成唯一键
         prefix: go_1-18_cache
         files:
           - go.sum
       paths:
         - .go/pkg/mod
   
   job:unit_test:
     stage: test
     image: $UNIT_TEST_IMAGE
     tags:
       - qc-exec-ut
     rules:
   	  # CI_PIPELINE_SOURCE 官方预制变量
       - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
     extends: .go-mod-cache
     script:
       # --后衔接的是go test命令的参数, -gcflags=all=-l：避免函数内联
       # gomonkey
       - gotestsum --junitfile report.xml --format pkgname --
         -gcflags=all=-l $(go list  -f '{{if len .TestGoFiles}} {{.ImportPath}}{{end}}' ./... | grep -v /vendor/)
         -coverprofile=coverage.out -covermode count -count=1
     artifacts:
       paths:
         - coverage.out
       when: on_success
   
   job:gen_ut_report:
     stage: test
     image: $UNIT_TEST_IMAGE
     tags:
       - qc-exec-ut
     needs:
       - job:unit_test
     rules:
       - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
     script:
       - go tool cover -func=coverage.out
       - go tool cover -html=coverage.out -o coverage_view.html
       - gocover-cobertura < coverage.out > coverage.xml
     # 用于在 MR  界面显示覆盖率，并参与 MR 的覆盖率计算
     coverage: '/total:\s+\(statements\)\s+\d+.\d+%/'
     artifacts:
       reports:
         coverage_report:
           # 两种format类型：1、cobertura -> 适用多数语言 2、JaCoCo -> 仅对Java生效
           coverage_format: cobertura
           path: coverage.xml
         junit: report.xml
       paths:
         - coverage.out
         - coverage_view.html
       expire_in: 1 week
       expose_as: 'coverage_review'
   
   job:gen_increment_ut_report:
     stage: test
     image: $UNIT_TEST_IMAGE
     needs:
       - job:unit_test
       - job:gen_ut_report
     rules:
       - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
     tags:
       - qc-exec-ut
     extends: .go-mod-cache
     coverage: '/total:\s+\(statements\)\s+\d+.\d+%/'
     script:    
       - git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
       - go-diff -file=coverage.out -branch=origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME
       - go tool cover -func=increment_coverage.out
       - go tool cover -html=increment_coverage.out -o increment_coverage_view.html
       # 检查增量覆盖率是否达到要求
       - |
         increment_coverage=$(go tool cover -func=increment_coverage.out | grep total | awk '{print $3}' | sed 's/%//'   | cut -d. -f1)
         if [ "$increment_coverage" -lt "$INCREMENT_COVERAGE_THRESHOLD" ]; then
           echo "increment_coverage $increment_coverage% is below threshold $INCREMENT_COVERAGE_THRESHOLD%"
           exit 1
         fi
         echo "increment_coverage $increment_coverage% meets threshold $INCREMENT_COVERAGE_THRESHOLD%"
     artifacts:
       paths:
         - increment_coverage.out
         - increment_coverage_view.html
       when: always
       expire_in: 1 week
       expose_as: 'increment_coverage_review'
   ```

4. 通过缓存优化脚本执行速度

   1. 删除相关缓存后,在部署`runner`容器的虚拟机上查看缓存相关的`volume`

      ![image-20250707161217287](D:\笔记\Go\图床\image-20250707161217287.png)

      重新启动流水线

   <img src="D:\笔记\Go\图床\image-20250707160539574.png" alt="image-20250707160539574" style="zoom: 33%;" />

   2. 发现多了三个新的目录

      ![image-20250707161318056](D:\笔记\Go\图床\image-20250707161318056.png)

   3. 进入某个`volume`，可以看到相关的缓存以zip格式存储在目录里

      <img src="D:\笔记\Go\图床\image-20250707161411111.png" alt="image-20250707161411111" style="zoom:67%;" />

   4. 对比有缓存的执行速度，接近快了一半
      <img src="D:\笔记\Go\图床\image-20250707162011159.png" alt="image-20250707162011159" style="zoom:67%;" />

   

#### Runner轮询Job原理

1. 部署`runner`命令

   ```shell
   # dind
   docker run -d --name gitlab-runner --restart always --env TZ=Asia/Shanghai \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v qc-gitlab-runner-ps-volume:/etc/gitlab-runner \
     gitlab/gitlab-runner:v15.10.0
     
   # container_id: 503f45209c32
   ```

2.  检查容器启动命令

   ```shell
   [root@sz_sjzx_dev01_17_33:/home/sudo_root]# docker inspect --format='{{.Config.Cmd}}' 503f45209c32
   [run --user=gitlab-runner --working-directory=/home/gitlab-runner]
   
   # 调用了`gitlab-runner run`命令
   ```

3. 下载[gitlab-runner仓库](https://gitlab.com/gitlab-org/gitlab-runner)查看源码，搜素关键字`RunCommand`

   1. `main.go`: 注册所有命令，包括`RunCommand`，当调用了`gitlab-runner run`命令，会执行`RunCommand.Execute`方法

   2. `commands/multi.go`:  

      ```go
      func (mr *RunCommand) Execute(_ *cli.Context) {
      	svcConfig := &service.Config{...}
      
      	svc, err := service_helpers.New(mr, svcConfig)
      
      	err = svc.Run()
      }
      
      // service_helpers.New
      func New(i service.Interface, c *service.Config) (service.Service, error) {
      	s, err := service.New(i, c)
      	...
      	return s, err
      }
      
      // service.New
      // github.com/kardianos/service
      func New(i Interface, c *Config) (Service, error) {
      	if len(c.Name) == 0 {
      		return nil, ErrNameFieldRequired
      	}
      	if system == nil {
      		return nil, ErrNoServiceSystemDetected
      	}
          // 根据操作系统控制
      	return system.New(i, c)
      }
      
      // svc.Run() -> 实际调用传入的mr.Start方法
      func (mr *RunCommand) Start(_ service.Service) error {
      	...
      	go mr.run()
      	return nil
      }
      
      func (mr *RunCommand) run() {
          // step1
      	runners := make(chan *common.RunnerConfig)
      	go mr.feedRunners(runners)
      
           // step2
      	startWorker := make(chan int)
      	stopWorker := make(chan bool)
      	go mr.startWorkers(startWorker, stopWorker, runners)
      
           // step3
      	workerIndex := 0
      	for mr.stopSignal == nil {
      		signaled := mr.updateWorkers(&workerIndex, startWorker, stopWorker)
      		if signaled != nil {
      			break
      		}
      
      		signaled = mr.updateConfig()
      		if signaled != nil {
      			break
      		}
      	}
          ...
      }
      ```

<img src="D:\笔记\Go\图床\image-20250705165214107.png" style="zoom:67%;" />

<img src="D:\笔记\Go\图床\image-20250705175837900.png" alt="image-20250705175837900"  />

#### difference between `artifacts` and `cache`

- `artifacts`

  - **reference**:
    - 
- `cache`
  - **reference**: 
    - [cache syntax in yaml](https://docs.gitlab.com/ci/yaml/#cache)
    - [Cache in Gitlab CI/CD](https://docs.gitlab.com/ci/caching/)
    - [use cache for caching go dependency](https://www.zhaowenyu.com/gitlab-doc/docs/213.html)
    - [archive/extract cache principle](https://docs.gitlab.com/ci/caching/#how-archiving-and-extracting-works)

`cache`存储依赖项，如软件包(`go mod` / `npm dependencies`)等，其下载的数据源存储在**部署`Gitlab Runner`的主机上**

`artifacts`由`Job`生成，用于在`Stage`之间传递`build result`(本质上也是通过Gitlab Server在发送Job时携带过去的)





### 实现获取增量覆盖率

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
6. 使用`golang.org/x/tools/cover`包解析`coverage.out`文件，这个覆盖率文件是在`CI阶段`执行`系统测试的job`生成的`artifact`，执行`增量测试覆盖率job`会通过`needs`指定`系统测试的job`完成后获取该`artifact`

<img src="D:\笔记\Go\图床\image-20250521164810497.png" alt="image-20250521164810497" style="zoom:50%;" />

<img src="D:\笔记\Go\图床\image-20250521164314617.png" alt="image-20250521164314617" style="zoom:50%;" />

## QA

1. `fatal: shallow file has changed since we read it`
   1. 在`runner's config.toml`中，`concurrent`=0，导致多个job并行执行时同时使用了同一个`executor`,修改`concurrent`=4(最大数)

2. 在单条流水线的多个Job中都执行覆盖率计算，导致MR界面看到的是这些job的覆盖率的平均值：[官方解释](https://gitlab.com/gitlab-org/gitlab/-/issues/15399)

​	
