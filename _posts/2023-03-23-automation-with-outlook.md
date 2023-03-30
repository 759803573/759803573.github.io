---
layout: post
title: '3min阅读: 基于Outlook做办公自动化'
date: 2023-03-30
author: Calvin Wang
# cover: '/assets/img/202209/saml.drawio.png'
tags: outlook python automation
---

> 最近有同事问我怎么结合Outlook来做自动化，简单聊聊


## 遇到了什么问题？
同事有一些例行任务需要用到Outlook, 比如：
1. 不定期从Vendor通过Email收到数据文件， 进行校验和处理。
2. 不定时收到服务器配置，需要在本地更新
3. 特定邮件要备份在Outlook以外

## 解决方式呢？
1. 在Outlook中配置收件规则， 收到特定邮件触发VBA脚本执行。
   ![](/assets/img/2023/e1.png)
2. 在VBA脚本中获取EmailItem的EntryID，调用外部脚本并将EntryID作为参数参数
   ![](/assets/img/2023/e2.png)
3. 外部脚本通过EntryID重新获取EmailItem， 接下来就是用擅长的语言解决简单的问题了
   ![](/assets/img/2023/e3.png)


```python
import argparse
import sys
import time

import win32com.client
from winotify import Notification

parser = argparse.ArgumentParser()
parser.add_argument(f'--id')
parsed, _ = parser.parse_known_args(sys.argv[1:])


stoast = Notification(app_id="outlook automation",
                     title="收到配置更新邮件",
                     msg=f"收到配置邮件")

stoast.show()

outlook = win32com.client.Dispatch("Outlook.Application").GetNamespace("MAPI")
message = outlook.GetItemFromID(parsed.id)


etoast = Notification(app_id="outlook automation",
                     title="配置更新完成",
                     msg=f"邮件正文: {message.Body}")
etoast.show()
```

## 这样做的好处呢？
1. 少干点重复的事情
