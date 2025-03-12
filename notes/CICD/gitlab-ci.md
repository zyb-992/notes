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

