公司的AT测试需要打开web页面验证登陆， 久而久之模拟器上就会堆积N多的页面，尝试过 adb clear chrome，但这样做的同时输入法，浏览器的设置都需要重新确认，最后发现adb forward可以帮我们完美的解决这个问题，代码看完就懂了。



```python
import json
from time import sleep
import requests
import os
if __name__ == '__main__':
    # adb forward转发端口到本地, http://localhost:9222/json 观察是否成功转发
    os.popen("adb forward tcp:9222 localabstract:chrome_devtools_remote")
    # 启动chrome
    os.popen("adb shell am start -n com.android.chrome/com.google.android.apps.chrome.Main")
    sleep(3)
    # 读取web tab页面
    response = requests.get("http://localhost:9222/json/list")
    # print(response.text.encode('utf-8'))
    json_data = json.loads(response.text.encode('utf-8'))
    for link in json_data:
        print(link['id'])
        # close all tabs
        response = requests.get("http://localhost:9222/json/close/" + link['id'])
        print(response.text)
```

