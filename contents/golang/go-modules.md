
# go modules 私有仓库依赖

go mod 作为官方正牌的依赖管理工具，常规操作均是从开源仓库进行依赖。而对于私有仓库则需要一些额外配置。

## 目前的问题

如果需要使用公司内部依赖（跨工程）主要存在两个问题：

1. go mod 无法通过 https 方式访问到 https://git.ghostcloud.cn 因为是公司仓库仅支持 http 和 ssh。
1. go mod 访问私有仓库默认没有配置认证相关参数。访问被拒绝。

## 解决方式

参考[Why does "go get" use HTTPS when cloning a repository?](https://golang.org/doc/faq#git_https)

使用 ssh 的方式，首先需要配置 gitlab 的 ssh 认证, [GitLab and SSH keys](https://docs.gitlab.com/ee/ssh/).

配置 git 重写规则，让 https 的请求使用 ssh 认证方式：

```sh
git config --global url."ssh://git@git.ghostcloud.cn:".insteadOf "https://git.ghostcloud.cn"
```

~~由于是 http 所以需要增加参数 `-insecure`:~~

	```sh
	$ GOPROXY=direct go get -insecure  -v  git.ghostcloud.cn/newben/newben
	get "git.ghostcloud.cn/newben/newben": found meta tag get.metaImport{Prefix:"git.ghostcloud.cn/newben/newben", VCS:"git", RepoRoot:"http://git.ghostcloud.cn/newben/newben.git"} at 			//git.ghostcloud.cn/newben/newben?go-get=1
	go: finding git.ghostcloud.cn/newben/newben latest
	```

可以看到 go.mod 内容已经增加了该私有仓库了。

## 问题排除

```sh
$ go get -v  git.ghostcloud.cn/newben/newben
go get git.ghostcloud.cn/newben/newben: module git.ghostcloud.cn/newben: reading https://goproxy.io/git.ghostcloud.cn/newben/@v/list: 404 Not Found
```

如果配置了 GOPROXY 则会出现以下输出,在使用私有仓库的时候临时关闭代理。在使用私有仓库的时候临时关闭代理。

```sh
$ GOPROXY=direct 
$ go get -v  git.ghostcloud.cn/newben/newben
go get git.ghostcloud.cn/newben/newben: unrecognized import path "git.ghostcloud.cn/newben/newben" (https fetch: Get https://git.ghostcloud.cn/newben/newben?go-get=1: dial tcp 182.92.83.19:443: connect: connection refused)
```

go get 会默认加参数`go-get=1`并以`https`协议访问`https://{host}/{path}?go-get=1`,以获取包名重定向，有兴趣的可以尝试访问一下 [https://k8s.io/?go-get=1](https://k8s.io/?go-get=1) 就明白了。

所以如果要以`http`方式访问则需要增加 `-insecure` 参数。
