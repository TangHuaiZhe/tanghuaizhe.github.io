

####背景

统一项目代码格式，增加代码可读性

#### 传统的解决方法是:

依赖代码提交者自觉将需要提交上传的代码整理好，或者每隔一段时间统一整理代码，统一提交。

显然这两种方案都不是很靠谱。



#### 目前比较自动化的方案:

集成[spotless gradle plugin](https://github.com/diffplug/spotless/tree/master/plugin-gradle)

插件集成后会将格式检查任务关联到build任务，格式有问题会直接导致编译报错，开发人员可以得到相关提示。



#### IDEA代码格式设置

首先设置IDEA的默认代码格式:

File→Settings→Editor→Code Style→Java→Scheme→Import Scheme 选择 [google-java-format.xml](https://raw.githubusercontent.com/google/styleguide/gh-pages/intellij-java-google-style.xml)

之后再使用快捷键调整代码格式就以google-java-format为准.



#### 在Android中的集成:

以`googleJavaFormat`格式为例:

在`rootProject`目录下的`build.gradle`中:

```groovy
buildscript {
	dependencies {
    	classpath 'com.diffplug.spotless:spotless-plugin-gradle:3.23.1'
	}
}	


allprojects {
    apply plugin: 'com.diffplug.gradle.spotless'
    spotless {
        java {
            target '**/*.java'
            googleJavaFormat("1.7")
        }
    }
}
```



此时`spotless`已经和项目的`build`任务关联

如果项目存在违反`googleJavaFormat`格式的`java`代码,

当开发人员运行`./gradlew build`时，编译将直接终止，并报错如下类似提示:

```
* What went wrong:
Execution failed for task ':spotlessJava'.

> The following files had format violations:
      app/src/com/sdpopen/demo/ui/MainActivity.java
          @@ -30,14 +30,13 @@
           import·com.sdpopen.wallet.home.code.activity.PaymentCodeActivity;
           import·com.sdpopen.wallet.user.bean.UserHelper;
           import·com.wifipay.common.security.Base64;
          -import·org.json.JSONObject;
          -
           import·java.net.URLEncoder;
           import·java.security.KeyFactory;
           import·java.security.PrivateKey;
           import·java.security.Signature;
           import·java.security.spec.PKCS8EncodedKeySpec;
           import·java.util.*;
          +import·org.json.JSONObject;
           
           public·class·MainActivity·extends·Activity·{
           ··//
      app/src/com/opensdk/test/PrePayOrderActivity.java
          @@ -277,16 +277,16 @@
           ······isLunckyMoney·=·false;
           
           ······//······PayTool.getInstance().startPay(this,·mRespone,·request);
          -··················testH5Pay(request);
          +······testH5Pay(request);

  Run 'gradlew spotlessApply' to fix these violations.
```



这时候运行`./gradlew spotlessApply`则可以直接自动修复代码格式问题;

再编译则可以通过。



这在`java`项目中已经足够使用，但是在`Android`项目中，由于常用快速的编译的命令是`./gradlew assembleFlavorbuildType`,因此还需要一些定制化的操作:

在项目的`build.gradle`中增加任务:

```
task format(
        dependsOn: 'spotlessApply',
        group: 'Verification')

task formatCheck(
        dependsOn: 'spotlessCheck',
        group: 'Verification')

//开启则每次编译时检查代码格式
//preBuild.dependsOn formatCheck
```

若要每次运行`./gradlew assembleFlavorbuildType`时检查代码格式，直接开启第10行即可。



#### hook git pre-commit:

考虑到我们实际上在意的是开发人员提交代码的时候的格式，而不是每次编译的时候，

这种每次编译就去检查代码格式的频率太高了，因为开发新功能的时候必然是多次编译重复验证的，此时对代码格式大可不必关心。



因此可以考虑将此格式检查任务和git commit的流程关联起来，每次开发人员git commit之前，去执行`formatCheck`任务，如果格式有问题，终止commit，调整格式后再commit即可。

参考[Git hook在android gradle中的简单使用](https://tanghuaizhe.github.io/2019/07/06/Git-hook%E5%9C%A8Android-gradle%E4%B8%AD%E7%9A%84%E7%AE%80%E5%8D%95%E4%BD%BF%E7%94%A8.html)：

在项目下新建`hooks`目录，新建`pre-commit`文件:

```shell
#!/bin/sh
#
# An example hook script to verify what is about to be committed.
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.
#
# To enable this hook, rename this file to "pre-commit".

./gradlew formatCheck
result=$?
printf "the spotlessCheck result code is $result"
if [[ "$result" = 0 ]] ; then
    echo "\033[32m
    ....
    ....
    SpotlessCheck Pass!!
    ....
    ....
    \033[0m"
    exit 0
else
    ./gradlew format
    echo "\033[31m
    ....
    ....
    SpotlessCheck Failed!!
    代码格式有问题;
    ....
    已经自动调整格式,review代码后再git add . && git commit
    ....
    ....
    \033[0m"
    exit 1
fi
```

当格式有问题的时候,在中断commit之前,先自动调整一把格式,开发人员只需再次git add . &&  git commit即可。



在项目`build.gradle`中:

```grovvy
task copyHooks(type: Copy) {
    println 'copyHooks task'
    from("hooks") {
        include "**"
    }
    into ".git/hooks"
}

preBuild.dependsOn copyHooks
```

这样当开发人员至少进行了一次编译任务之后，我们的`pre-commit`就会hook住git commit的流程。





参考:

[spotless](https://github.com/diffplug/spotless)

[google-java-format](https://github.com/google/google-java-format)











