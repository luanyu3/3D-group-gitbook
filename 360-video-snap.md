## 环境
- 操作系统
- python依赖 `win32` (pywin32 窗口模块)

![image](https://github.com/user-attachments/assets/8b8d9b37-8a9c-4fe4-b2e7-9ab7e7eadde9)

## 用法
- 方法：通过调用窗口模块批量自动截图（360视频的一种提取图片的方法）

![image](https://github.com/user-attachments/assets/afc9dd21-5540-4fc5-8913-59a7087cb709)

- 将shot.py文件与main.py文件放在同级目录，main.py导入win32模块；

![image](https://github.com/user-attachments/assets/fb6047b4-0474-4717-969f-82b28a861339)

- 执行main.py文件，然后播放视频，保持视频窗口不被最小化。视频播放完成之后需要自己手动暂停播放器再关闭程序。

![image](https://github.com/user-attachments/assets/7b35b90f-51a5-4735-ad05-58df24e0d967)


## 代码
```python
# main.py 代码
import win32api
import win32gui
import win32con
import time
import shot # 导入shot.py文件

wdname = u'f025f6ae25dC1794e7195efa80555b5f.mp4 - PotPIayer' # 只需要更换视频文件名即可
handle = win32gui.FindWindow(0, wdname) # 获取取窗囗问柄
while True:
  time.sleep(l)
  shot.screen_shot(handle)
```

```python
# shot.py 代码
import os

import win32gui, win32ui, win32con
from ctypes import windll
from PIL import Image
import cv2
import numpy
import time

def screen_shot(hWnd):
    # 获取后台窗口的句柄，注意后台窗口不能最小化
  #  hWnd = win32gui.FindWindow(0, name)  # 窗口的类名可以用Visual Studio的SPY++工具获取
    # 获取句柄窗口的大小信息
    left, top, right, bot = win32gui.GetWindowRect(hWnd)
    width = right - left
    height = bot - top
    # 返回句柄窗口的设备环境，覆盖整个窗口，包括非客户区，标题栏，菜单，边框
    hWndDC = win32gui.GetWindowDC(hWnd)
    # 创建设备描述表
    mfcDC = win32ui.CreateDCFromHandle(hWndDC)
    # 创建内存设备描述表
    saveDC = mfcDC.CreateCompatibleDC()
    # 创建位图对象准备保存图片
    saveBitMap = win32ui.CreateBitmap()
    # 为bitmap开辟存储空间
    saveBitMap.CreateCompatibleBitmap(mfcDC, width, height)
    # 将截图保存到saveBitMap中
    saveDC.SelectObject(saveBitMap)
    # 保存bitmap到内存设备描述表
    saveDC.BitBlt((0, 0), (width, height), mfcDC, (0, 0), win32con.SRCCOPY)

    timestamp = int(time.time())
    t = "pic/"+str(timestamp) + ".jpg"
    saveBitMap.SaveBitmapFile(saveDC,t)
    # 使用时间戳作为文件名

    win32gui.DeleteObject(saveBitMap.GetHandle())
    saveDC.DeleteDC()
    mfcDC.DeleteDC()
    win32gui.ReleaseDC(hWnd, hWndDC)
```

## 结果
最终获得按照时序排列的截图

![a39f95a64e917e86928006d949bbedd](https://github.com/user-attachments/assets/7bdb8420-e7d5-4215-ac66-919ccf6b52cc)
