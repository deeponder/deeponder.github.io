---
layout:     post
title:      google文档api使用
subtitle:   
date:       2020-03-6
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - 谷歌文档
---

# 一、背景
1. 通过谷歌文档管理自己的股票池，并希望能自定义条件，主动通知
2. 工作上， 需要能在跑完压测后，把压测数据自动同步到谷歌文档，并展示出来 

# 二、谷歌文档函数
提供了一系列的官方云函数/公式，可实现类似爬虫的功能，详见[谷歌文档函数列表](https://support.google.com/docs/table/25273?hl=en&ref_topic=3105411)
这里举两个我用到的函数。
1. GOOGLEFINANCE。这个函数可以实时通过谷歌财经实时获取到指定股票的数据，包括价格、市盈率等，例如`=GOOGLEFINANCE("NASDAQ:GOOGL","pe")`可获取到google公司实时的市盈率
2. IMPORTHTML。可以爬取任意网站的表格。例如`=IMPORTHTML("http://en.wikipedia.org/wiki/Demographics_of_India","table",4)`可获取到该网站下， 第四个table的内容

# 三、官方脚本--AppScript
AppScript有个类支持访问和操作谷歌文档--[Class Spreadsheet](https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet)
这里我想达到的效果是， 定时去跑某个函数， 然后当谷歌文档中的数据，满足某个条件的话，即给我发一封提醒邮件。 定时执行脚本可以用触发器配执行时机的策略，脚本参考如下：
```
function sendEmail(content) {
  var message = 'stock attack!! please check: https://docs.google.com/spreadsheets/d/1iLPQ8igJyfCK8uncMJ1p6XsnnkCayZ8U8KMk9HM4Dyg/edit#gid=0'; // Second column
  var subject = content;
  MailApp.sendEmail("577002615@qq.com", subject, message);
}


function checkPEG() {
  // 1. fetch sheet data
  var stockSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("stocks");
  var pegRange = stockSheet.getRange("B12:X12"); 
  var pegValues = pegRange.getValues()[0];
  var buyFlag=0;
  // 2. traverse the valuse i care 
  for (var i=0; i<pegValues.length;i++) {
    if (pegValues[i]<=0.6) {
      Logger.log(pegValues[i]);
      buyFlag=1;
    }
  }
  
  // 3. find one, send me a mail
  if(buyFlag) {
    sendEmail("buy signal!")
  }
  
  var pegSellRange=stockSheet.getRange("B15:X15");
  var pegSellValues=pegSellRange.getValues()[0];
  
  var sellFlag=0;
  for (var i=0; i<pegSellValues.length;i++) {
    if (pegSellValues[i]==1) {
      Logger.log(pegSellValues[i]);
      sellFlag=1;
    }
    //Logger.log(pegValues[i]);
  }
  
  if(sellFlag) {
    sendEmail("sell signal!")
  }
  
}
```

# 四、自主管理谷歌文档--base on python
更进一步地，我们希望能用我们自己的脚本去操作谷歌文档。 
1. 首先，需要到[Google Developers Console](https://console.developers.google.com/cloud-resource-manager)创建一个项目，并注册一个service account和下载对应的凭证， 可参考[Using OAuth2 for Authentication](https://gspread.readthedocs.io/en/latest/oauth2.html)
2. 接着， [安装python环境](https://deeponder.github.io/2020/03/20/python%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2/), 引入oauth2client`pip install --upgrade oauth2client`
4. 最后引入开源的gspread`pip install gspread`， 参照[gspread官方文档](https://gspread.readthedocs.io/en/latest/)就可以愉快地蹂躏、操作谷歌文档了
最后附上小小的例子， 实现的功能是往指定的google文档追加一行记录
```
...
scope = ['https://spreadsheets.google.com/feeds']

credentials = ServiceAccountCredentials.from_json_keyfile_name('lgame-8570bdc1de6e.json', scope)

gc = gspread.authorize(credentials)

# Open a worksheet from spreadsheet with one shot
wks = gc.open_by_key("1l78DlUkTpsqLRFj-1BMMMzlnKNrE09eZVP7ivpK4Gn4").worksheet("ZoneSvr")

# wks.insert_row([7, 15, 18])
wks.append_row([7, 15, 18])
print wks.get_all_records()
...
```
