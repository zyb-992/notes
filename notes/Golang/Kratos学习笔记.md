# Kratos学习笔记

## 代码

### 编译标记

```go
go 1.17后使用go:build作为新的编译标记，1.17前的使用+build标记
//go:build wireinject
// +build wireinject
```

### go generate

> https://yushuangqi.com/blog/2017/go-command-generate.html

go generate是Go语言中的一个命令，用于在Go源代码中执行自定义的命令或脚本，以生成代码或执行其他必要的构建任务。常用于代码生成工具的构建过程。通过在Go源代码中添加`//go:generate`注释，并定义相应的命令或脚本，可以方便地生成重复性、模板化或基于元数据的代码。

```go
逐行匹配以//go:generate 开头的行
//go:generate command arguments
```

注意点：

1. `go:generate`前面只能使用`//`注释，注释必须在行首，前面不能有空格且`//`与`go:generate`之间不能有空格！！！
2. `go:generate`可以在任何 Go 源文件中，最好在类型定义的地方。

### wire

> https://bennhuang.com/posts/wire/
>
> https://go.dev/blog/wire

两个基本概念，Provider、Injector，即构造器和注入器；wire.Build入餐中不用保证provider的顺序

```go
// three provider

// NewUserStore is the same function we saw above; it is a provider for UserStore,
// with dependencies on *Config and *mysql.DB.
func NewUserStore(cfg *Config, db *mysql.DB) (*UserStore, error) {...}

// NewDefaultConfig is a provider for *Config, with no dependencies.
func NewDefaultConfig() *Config {...}

// NewDB is a provider for *mysql.DB based on some connection info.
func NewDB(info *ConnectionInfo) (*mysql.DB, error) {...}

// injector
func initUserStore() (*UserStore, error) {
    // We're going to get an error, because NewDB requires a *ConnectionInfo
    // and we didn't provide one.
    wire.Build(UserStoreSet, NewDB)
    return nil, nil  // These return values are ignored.
}
```

TODO: 学习[Bin-Huang/newc](https://github.com/Bin-Huang/newc)仓库

### Makefile

- **find api -name *.proto**在zsh中无法匹配，需要替换为 **find api -name ".proto"**
- 执行*make api*时报错*make: protoc: No such file or directory，*需要提前安装protoc程序
  - 安装：brew install protobuf

### protoc命令参数含义

> Protobuf https://protobuf.dev/programming-guides/style/
>
> 

