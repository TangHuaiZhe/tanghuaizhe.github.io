### 1. 背景：

方便开发人员确认某个App安装包中是否包含对应的代码

### 2. 解决方案

比较传统的方法是反编译app，直接用jadx或jd-gui等工具看代码是否包含。

然而比较麻烦，app加固之后也不能方便的反编译。



最后决定在app中增加一个后门对话框，参考google android开发者模式，2秒内点击5次就可以弹出对话框。

对话框中记录最后一次Git commit信息，版本号，编译时间等等信息。



#### 2秒内点击5次的判断逻辑:

```java
  private void something() {
    getmTitleBar()
        .setOnClickListener(
            new View.OnClickListener() {
              final int COUNTS = 5;
              final long DURATION = 2 * 1000;
              long[] mHits = new long[COUNTS];
 
              @Override
              public void onClick(View v) {
                // 实现左移
                System.arraycopy(mHits, 1, mHits, 0, mHits.length - 1);
                mHits[mHits.length - 1] = SystemClock.uptimeMillis();
                if (mHits[0] >= (SystemClock.uptimeMillis() - DURATION)) {
                  // show dialog
                }
              }
            });
  }
```



#### 编译时获取最后一次git commit信息：

这里需要注意由于Android无法解析windows对汉字的GBK编码，会出现乱码，因此需要直接对字节流`utf-8`处理，而不能直接拿文字信息;

如果团队都是mac和linux系统，

可以直接使用`def gitCommit = cmd.execute().text.trim()`获取命令的执行结果，在Android上展示无乱码问题，考虑兼容性，还是需要转换一次

```groovy
def inputStream = cmd.execute().getIn()
def gitLog = IOGroovyMethods.getText(new BufferedReader(new InputStreamReader(inputStream, "utf-8")))
```



**完整代码如下:**

```groovy
import org.codehaus.groovy.runtime.IOGroovyMethods

def getGitCommit() {
    def gitDir = new File("${rootDir}/.git")
    println("gitdir is ${gitDir}")
    if (!gitDir.isDirectory()) {
        return 'non-git-build'
    }

    def cmd = "git log --pretty=format:%cn_%s_%h_%cd --date=format:'%Y-%m-%d_%H:%M' -1"
    def inputStream = cmd.execute().getIn()
    def gitLog = IOGroovyMethods.getText(new BufferedReader(new InputStreamReader(inputStream, "utf-8")))
    if (gitLog != null && gitLog.length() > 0) {
        return gitLog
    } else {
        return "unknown"
    }
}
```

#### 在Gradle编译时插入相关字段:

```groovy
    variant.buildConfigField 'String', 'LAST_CI', "\"${getGitCommit()}\""
    variant.buildConfigField 'String', 'BUILD_TIME', "\"${releaseTime()}\""
```

#### 在Java中引用:

```java
String message = BuildConfig.LAST_CI
```

得到的结果类似:

`Tony_MOBILE-2722 优化环境配置_abea5547_2019-07-31_09:46`

格式为：git 提交者+ git commit message + git hash + 时间



#### Git log的格式自定义:

`git log --pretty=format:%cn_%s_%h_%cd --date=format:'%Y-%m-%d_%H:%M' -1`

具体自定义可参考[Git log](https://git-scm.com/docs/git-log)

