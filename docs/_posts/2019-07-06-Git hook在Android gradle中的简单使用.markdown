### 1. 背景：防止开发人员将编译出错的代码提交到git仓库

### 2. 如何编写git hook:

观察项目`.git/hooks/`目录下的钩子:

- applypatch-msg.sample     
- pre-applypatch.sample     
- pre-receive.sample
- commit-msg.sample         
- pre-commit.sample         
- prepare-commit-msg.sample
- fsmonitor-watchman.sample 
- pre-push.sample           
- update.sample
- post-update.sample       
- pre-rebase.sample

只要将文件后缀.sample删除，钩子就会启用。

选择`pre-push`钩子，也就是git push之前会执行的脚本，脚本如下:

```shell
#!/bin/sh

./gradlew WKDD && ./gradlew LXDD

status=$?

if [ "$status" = 0 ] ; then
    echo "
    ....
    ....
    Build Check Pass
    ....
    ....
    "
    exit 0
else
    echo "
    ....
    ....
    Build Check Failed!!
    ....
    ....
    "
    exit 1
fi
```

这里的`./gradlew WKDD && ./gradlew LXDD`是我们项目的编译命令，自行修改所需命令，为了尽快验证，不作`clean`操作。



### 3. 如何同步git钩子:

由于`.git/hooks/`目录为每个工程本地私有的目录，并不会参与版本控制，因此需要一个机制在组内同步git hook设置。

首先在项目下新建`hooks`目录，将`pre-push`文件放在此目录下,此目录参与git版本控制；

然后在项目的build.gradle中新建如下task，每次编译之前都会执行一遍，将`hooks`目录下的文件复制到项目的`.git/hooks`目录下:

```groovy
task copyHooks(type: Copy) {
    println 'copyHooks task'
    from("hooks") {
        include "**"
    }
    into ".git/hooks"
}

preBuild.dependsOn copyHooks
```

如果使用maven，原理也是一致的，关键是将任务插入到正确的生命周期。



### 4. 验证后mac和windows都能支持