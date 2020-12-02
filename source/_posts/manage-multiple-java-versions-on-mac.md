---
title: Mac上的JDK多版本管理
date: 2019-09-24 15:27:36
tags: [JDK, jEnv]
categories: [Java]
---
![jEnv](/images/jenv.png)
我的Mac上已经有一个JDK8的版本了，这不[JDK13](http://openjdk.java.net/projects/jdk/13/)刚发布（2019-09-17），想快速的尝一尝鲜，就得安装多个版本的JDK了。这个对Node、Ruby、Python的使用者来说，已经不是个什么新鲜话题了，但是对于Java的使用者来说，似乎没有那么多的人受到过多版本的折磨（我是通过GitHub上[`nvm`](https://github.com/nvm-sh/nvm)、[`rbenv`](https://github.com/rbenv/rbenv)、[`pyenv`](https://github.com/pyenv/pyenv)、[`jenv`](https://github.com/jenv/jenv)的Star数量臆测出这个结论的 :P）。

<!-- more -->

* nvm
<iframe src="https://ghbtns.com/github-btn.html?user=nvm-sh&repo=nvm&type=star&count=true" frameborder="0" scrolling="0" width="200px" height="20px" style="vertical-align:middle"></iframe>
* rbenv
<iframe src="https://ghbtns.com/github-btn.html?user=rbenv&repo=rbenv&type=star&count=true" frameborder="0" scrolling="0" width="200px" height="20px" style="vertical-align:middle"></iframe>
* pyenv
<iframe src="https://ghbtns.com/github-btn.html?user=pyenv&repo=pyenv&type=star&count=true" frameborder="0" scrolling="0" width="200px" height="20px" style="vertical-align:middle"></iframe>
* jenv
<iframe src="https://ghbtns.com/github-btn.html?user=jenv&repo=jenv&type=star&count=true" frameborder="0" scrolling="0" width="200px" height="20px" style="vertical-align:middle"></iframe>

## 安装JDK 13

通过Homebrew 安装JDK 13，可以先通过`brew cask info java`查看目前Java的版本：
```bash
java: 13,33:5b8a42f3905b406298b72d750b6919f6
https://openjdk.java.net/
Not installed
From: https://github.com/Homebrew/homebrew-cask/blob/master/Casks/java.rb
==> Name
OpenJDK Java Development Kit
==> Artifacts
jdk-13.jdk -> /Library/Java/JavaVirtualMachines/openjdk-13.jdk (Generic Artifact)
```
这里显示的是JDK13，正好是我想要安装的JDK版本，如果不是你想要的版本可以自己搜索相应的 Homebrew Tap。接下来直接安装:
```bash
$ brew cask install java
$ java -version

openjdk version "13" 2019-09-17
OpenJDK Runtime Environment (build 13+33)
OpenJDK 64-Bit Server VM (build 13+33, mixed mode, sharing)
```
这就说明JDK13已经安装好了。

但是另一个问题来了，我电脑上原来安装的JDK8去哪呢？我如何在不同的版本中随意切换呢？比如像Node的`nvm`，Ruby的`rvm`，Python的`pyenv`等。答案是我们可以通过`jenv`来实现相同的效果。

## 安装 jEnv

1. 安装 jEnv

```bash
$ brew install jenv
$ exec $SHELL -l
```
安装完成之后，然后检查是否安装成功。
```bash
$ jenv doctor
[OK]	No JAVA_HOME set
[ERROR]	Java binary in path is not in the jenv shims.
[ERROR]	Please check your path, or try using /path/to/java/home is not a valid path to java installation.
	PATH : /usr/local/Cellar/jenv/0.5.2/libexec/libexec:/Users/xxx/.cargo/bin:/Users/xxx/.pyenv/shims:/Users/username/.pyenv:/Users/xxx/.nvm/versions/node/v8.11.4/bin:/Users/xxx/bin:/usr/local/bin:/Users/xxx/.cargo/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/go/bin:/Users/xxx/Documents/Projects/golang/bin
[ERROR]	Jenv is not loaded in your zsh
[ERROR]	To fix : 	cat eval "$(jenv init -)" >> /Users/xxx/.zshrc
```
在这里如果按照提示执行`cat eval "$(jenv init -)" >> /Users/xxx/.zshrc`：可能会得到如下错误：
```bash
cat eval "$(jenv init -)" >> /Users/xxx/.zshrc
cat: eval: No such file or directory
cat: export PATH="/Users/xxx/.jenv/shims:${PATH}"
export JENV_SHELL=zsh
export JENV_LOADED=1
unset JAVA_HOME
source '/usr/local/Cellar/jenv/0.5.2/libexec/libexec/../completions/jenv.zsh'
jenv rehash 2>/dev/null
jenv() {
  typeset command
  command="$1"
  if [ "$#" -gt 0 ]; then
    shift
  fi

  case "$command" in
  enable-plugin|rehash|shell|shell-options)
    eval `jenv "sh-$command" "$@"`;;
  *)
    command jenv "$command" "$@";;
  esac
}: No such file or directory
```
经过一番搜索，得到如下的解决办法，主要就是将`cat`替换为`echo`，这里我已经给jEnv提了个[PR](https://github.com/jenv/jenv/pull/265)，以消除这个干扰。
* Bash用户
```bash
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(jenv init -)"' >> ~/.bash_profile
$ exec $SHELL -l
```
* Zsh用户
```zsh
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(jenv init -)"' >> ~/.zshrc
$ exec $SHELL -l
```

然后再次执行`jenv doctor`，得到如下信息：
```bash
[OK]	No JAVA_HOME set
[ERROR]	Java binary in path is not in the jenv shims.
[ERROR]	Please check your path, or try using /path/to/java/home is not a valid path to java installation.
	PATH : /usr/local/Cellar/jenv/0.5.2/libexec/libexec:/Users/xxx/.jenv/shims:/Users/xxx/.cargo/bin:/Users/xxx/.pyenv/shims:/Users/username/.pyenv:/Users/xxx/.cargo/bin:/Users/xxx/.pyenv/shims:/Users/username/.pyenv:/Users/xxx/.nvm/versions/node/v8.11.4/bin:/Users/xxx/bin:/usr/local/bin:/Users/xxx/.cargo/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/go/bin:/Users/xxx/Documents/Projects/golang/bin:/Users/xxx/Documents/Projects/golang/bin
[OK]	Jenv is correctly loaded
```
为了能够正确的设置`JAVA_HOME`，最好开启`export`插件：
```bash
$ jenv enable-plugin export
$ exec $SHELL -l
```
如果你是Maven用户，建议开启Maven插件，使得Maven能够使用正确的JDK版本：

```shell
$ jenv enable-plugin maven
$ exec $SHELL -l
```



## 管理不同版本的JDK

### 添加JDK

添加最新安装的JDK：
```bash
$ jenv add $(/usr/libexec/java_home)
```
如果`/usr/libexec/java_home`所指的位置不是你想要的，也可以手动指定目录：
```bash
$ jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/
```

### 查看JDK版本

执行`jenv versions`：
```bash
  system
* 1.8 (set by JENV_VERSION environment variable)
  1.8.0.191
  13
  openjdk64-13
  oracle64-1.8.0.191
```
默认情况下，system指的是系统中安装的最新版本的JDK。

### 切换JDK版本

* Global
设置全局模式下的JDK版本：
```bash
$ jenv global 13
$ exec $SHELL -l 
$ java -version
```

* Local
在某个工作目录下设置JDK版本，会在当前目录下创建一个`.java-version`的文件：
```bash
$ jenv local 1.8
$ exec $SHELL -l 
$ java -version
```

* Shell
设置当前Shell session中的JDK版本：
```bash
$ jenv shell 1.8
$ java -version
```

## 参考链接

> * http://www.jenv.be/
> * https://github.com/jenv/jenv
> * https://emcorrales.com/blog/install-oracle-jdk-macos-homebrew