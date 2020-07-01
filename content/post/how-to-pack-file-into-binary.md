---
title: "如何将静态文件打包进入golang二进制程序中"
date: 2020-06-30T15:23:13+08:00
draft: true
tags: ["golang"]
categories: ["技术"]
---

# 文章由来

这篇文章源于一个实际的项目需求，当项目中的变量需要根据环境而变化时，会引入环境变量来根据不同环境适配。

在这个项目中需要用一个变量来设置cdn的地址，同时我当心cdn的地址会随时调整，所以把他设置成`.env`配置文件放在根目录下，文件内容如下这样：

```
CDN=http://cdn.xx.com/xxx
```

然后在项目中获取文件中对应key的值。但是这样操作之后有一个问题，就是配置文件没有打包进入编译后的binary中去，所以在程序中直接获取的话会出现can't open file .env之类的错误，搜索之后发现已经有很多开源包实现了打包的功能，下面就介绍一下其中一个第三方包`pkger`



# 如何使用pkger

- 安装

  ```shell
  go get -u -v github.com/markbates/pkger/cmd/pkger
  ```

- 生成映射文件

  pkger支持两种方式指定需要打包的文件或目录

  - 在命令行添加 `-include`参数

  - 在项目中调用如下方法，pkger会通过解析器判断哪些文件是需要被加入的

    > The [`github.com/markbates/pkger/parser#Parser`](https://godoc.org/github.com/markbates/pkger/parser#Parser) works by statically analyzing the source code of your module using the [go/parser](https://godoc.org/go/parser) to find a selection of declarations. The following declarations in your source code will tell the parser to embed files or folders

    

    - `pkger.Dir("<path>")`
    - `pkger.Open("<path>")` 
    - `pkger.Stat("<path>")`
    - `pkger.Walk("<path>", filepath.WalkFunc)`
    - `pkger.Include("<path>")` 

    

  ```shell
  pkger -include=yourmodule/.env
  ```

  执行之后会在根目录生成一个叫做`pkger.go`的中间文件，这个文件由`pkger`命令自动生成，且不需要修改

- 编译程序

  ```shell
  go build pkger.go main.go -o main
  ```



这样一通操作下来.env就已经加入到binary中了



#  嵌入原理

打开打包后的文件`pkger.go`查阅发现有一串如下代码

```golang
var _ = pkger.Apply(mem.UnmarshalEmbed([]byte(`1f8b08000000000000ffe....
```

`UnmarshalEmbed`方法的参数decode之后内容大概如下

```json
{
    "infos": {
      "yourmodule": {
        "Dir": "$HOME/project/path",
        "ImportPath": "yourmodule",
        "Name": "main",
        "Doc": "",
        "Target": "$GOPATH/bin/yourmodule",
        "Root": "$HOME/project/path",
        "Match": [
          "."
        ],
        "Stale": true,
        "StaleReason": "stale dependency: github.com/markbates/pkger",
        "GoFiles": [
          "main.go",
          "pkged.go"
        ],
        "Imports": [
        ],
        "Deps": [
         
        ],
        "TestGoFiles": null,
        "TestImports": null,
        "Module": {
          "Path": "yourmodule",
          "Main": true,
          "Dir": "$HOME/project/path",
          "GoMod": "$HOME/project/path/go.mod",
          "GoVersion": "1.14"
        }
      }
    },
    "files": {
        
    },
    "here": {}
  }
```

pkger大概做了这么几件事情来生成pkger.go

- 分析你的go.mod文件找出依赖包以及相关信息
- 分析你的代码或者cli参数，找出需要额外加载的文件或目录



生成文件之后，在文件中调用`pkger.Apply`方法会将上述信息写入一个内部变量之中

```go
func Apply(pkg pkging.Pkger, err error) error {
	if err != nil {
		panic(err)
		return err
	}
	gil.Lock()
	defer gil.Unlock()
	current = pkging.Wrap(current, pkg) // current就是一个内部单例，用来做路径映射
	return nil
}
```

这样后续我们调用pkger的相关方法例如`open`就会通过一定规则从`current`中找到对应的文件



# 实现简化版pkger

https://github.com/tim0991/pack-file-demo



# 参考

https://levelup.gitconnected.com/how-i-embedded-resources-in-go-514b72f6ef0a




