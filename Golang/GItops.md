

## GItops

> https://www.qikqiak.com/post/gitops-devops-on-k8s/
>
> https://www.cncf.io/blog/2025/06/09/gitops-in-2025-from-old-school-updates-to-the-modern-way/
>
> https://www.gitops.tech/
>
> GitOps is used to automate the process of provisioning infrastructure, especially modern cloud infrastructure. 

### 概念

GitOps 是一个概念，将软件的端到端描述放置到 Git 中，然后尝试着让集群状态和 Git 仓库持续同步

1. **软件的描述表示**：
2. **持续同步**：不断地检查 Git 仓库，将任何状态变化都反映到 Kubernetes 集群中

### 组成

1. `Git Repo`

   - Application  Code Repository：应用代码仓库
   - Environment  Config Repository：环境配置仓库
   - Container Image Repository：容器镜像仓库

   代码仓库至少包含两个：应用程序仓库和环境配置仓库。应用程序仓库包含应用程序的源代码以及用于部署应用程序的部署清单。环境配置仓库包含当前部署环境所需基础架构的所有部署清单。

2. `Kubernetes Cluster`

3. `Sync Agent (Operator)`

4. `CD pipeline`





### 问题

1. GitOps和DevOps区别