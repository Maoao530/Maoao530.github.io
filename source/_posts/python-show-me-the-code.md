---
title: python每日编程训练
categories:
  - Python基础
tags:
  - Python
date: 2016-08-10 13:47:00
---

python 编程练习

python 每日编程训练。（本文使用Python 3.5）

来源于 - [https://github.com/Yixiaohan/show-me-the-code](https://github.com/Yixiaohan/show-me-the-code)


<!-- more -->

本文以问题加源码的形式发布。

# 第 0000 题

第 0000 题：将你的 QQ 头像（或者微博头像）右上角加上红色的数字，类似于微信未读信息数量那种提示效果。 类似于图中效果
![0000-image](/img/archives/python-coding-0000.png)

```python
from PIL import Image
from PIL import ImageDraw, ImageFont

image = Image.open('0000.png', 'r')
font = ImageFont.truetype('c:/Windows/Fonts/Arial.ttf', 36)
draw = ImageDraw.Draw(image)

w,h = image.size
#左上角
draw.text(xy=(10, 10), text='9', fill=(255,0,0,255), font=font)
image.save('0000-new.png', 'png')
```

# 第 0001 题

第 0001 题：做为 Apple Store App 独立开发者，你要搞限时促销，为你的应用生成激活码（或者优惠券），
使用 Python 如何生成 200 个激活码（或者优惠券）？

```python
import uuid

for i in range(200):
    # the str length is 32 bytes
    print(i,  ' ====> ' , ''.join(str(uuid.uuid4()).split('-')))
```

# 第 0002 题

第 0002 题：将 0001 题生成的 200 个激活码（或者优惠券）保存到 MySQL 关系型数据库中。

```python
import uuid
import sqlite3

# create database
conn = sqlite3.connect('0002.db')
cursor = conn.cursor()
sql = 'create table if not exists save_code (id varchar(20) primary key, save_code varchar(50), is_use varchar(10))'
cursor.execute(sql)

for i in range(200):
    # generate str, length is 32 bytes
    s = ''.join(str(uuid.uuid4()).split('-'))
    print(i,  ' ====> ' , s)
    # insert into sqlite3
    try:
        insert_sql = "insert into save_code (id, save_code, is_use) values ('%s', '%s', '%s')" %(str(i), s, 'no')
        cursor.execute(insert_sql)
    except:
        print('you had insert the save_code')
        break
# select and check
select_sql = 'select * from save_code'
rs = cursor.execute(select_sql).fetchall()
for r in rs:
    print(r)

# close resource
cursor.close()
conn.commit()
conn.close()
```

# 第 0003 题

第 0003 题：将 0001 题生成的 200 个激活码（或者优惠券）保存到 Redis 非关系型数据库中。

```python
import uuid
import redis

r = redis.Redis(host='127.0.0.1', port=6379, db=0)

for i in range(200):
    # generate str, length is 32 bytes
    s = ''.join(str(uuid.uuid4()).split('-'))
    print(i,  ' ====> ' , s)
    r.lpush('save_code',s)

r.flushdb()
```

# 第 0004 题

第 0004 题：任一个英文的纯文本文件，统计其中的单词出现的个数。

```python
import re

with open('doc.txt', 'r') as f:
    data = f.read()
    words = re.compile(r'([a-zA-Z]+)').findall(data)

dicts = {}
for word in words:
    if dicts.get(word) == None:
        dicts[word] = 1
    else:
        dicts[word] = dicts[word] + 1
    
for k in dicts:
    print(k, ' ==> ', dicts[k])
```

# 第 0005 题

第 0005 题：你有一个目录，装了很多照片，把它们的尺寸变成都不大于 iPhone5 分辨率的大小。

```python
import os
from PIL import Image

images = [ x for x in os.listdir('.') if os.path.splitext(x)[1] == '.png' or os.path.splitext(x)[1] == '.png']

iphone5_w = 64
iphone5_h = 113

for image in images:
    print()
    img = Image.open(image, 'r')
    w, h = img.size
    scaleXY = max(w / iphone5_w, h / iphone5_h)
    print(scaleXY)
    if scaleXY > 1.0 : 
        img.thumbnail((w/scaleXY, h/scaleXY))
        img.save(os.path.splitext(image)[0]+'_n'+os.path.splitext(image)[1], os.path.splitext(image)[1][1:])
```

# 第 0006 题

第 0006 题：你有一个目录，放了你一个月的日记，都是 txt，为了避免分词的问题，假设内容都是英文，请统计出你认为每篇日记最重要的词。

```python
import re
import os

DIR_NAME = 'dir'

docs = os.listdir(DIR_NAME)

for doc in docs:
    with open(os.path.join(DIR_NAME,doc), 'r') as f:
        data = f.read()
        words = re.compile(r'([a-zA-Z]+)').findall(data)
    dicts = {}
    for word in words:
        if dicts.get(word) == None:
            dicts[word] = 1 
        else:
            dicts[word] = dicts[word] + 1
    
    maxValue = max(dicts.values())
    
    for k in dicts:
        if dicts[k] == maxValue:
            print(doc, ' ==> ', k,dicts[k])
```

# 第 0007 题

第 0007 题：有个目录，里面是你自己写过的程序，统计一下你写过多少行代码。包括空行和注释，但是要分别列出来。

```python
import os

DIR_NAME = 'dir'
docs = os.listdir(DIR_NAME)

for doc in docs:
    lines = 0
    comment_lines = 0
    blank_lines = 0
    with open(os.path.join(DIR_NAME,doc), 'r', encoding = 'utf8', errors = 'ignore') as f:
        multi_comment_line_start = 0
        for line in f.readlines():    
            lines += 1
            # 忽略所有空格
            line = line.split()          
            if len(line) == 0:
                blank_lines += 1
            elif line[0].startswith('#'):
                comment_lines += 1
            elif line[0].startswith("'''") and multi_comment_line_start == 0:
                multi_comment_line_start = lines
            elif line[0].startswith("'''"):
                comment_lines = comment_lines + (lines - multi_comment_line_start + 1)
                multi_comment_line_start = 0 
    print('========================')       
    print(doc,' : ')
    print('lines',lines)
    print('comment_lines', comment_lines)
    print('blank_lines', blank_lines)
```

# 第 0008 题

第 0008 题：一个HTML文件，找出里面的正文。

```python
from bs4 import BeautifulSoup
import requests

url = 'http://www.baidu.com'
page = requests.get(url)
soup = BeautifulSoup(page.text, 'html.parser')
print(soup.getText())
```

# 第 0009 题

第 0009 题：一个HTML文件，找出里面的链接。

```python
from bs4 import BeautifulSoup
import requests

url = 'http://www.baidu.com'
page = requests.get(url)
soup = BeautifulSoup(page.text, 'html.parser')
for link in soup.find_all('a'):
    print(link.get('href'))
```

# 第 0010 题

第 0010 题：使用 Python 生成类似于下图中的字母验证码图片

```python
from PIL import Image, ImageDraw, ImageFont, ImageFilter
import random

# 随机字母:
def rndChar():
    return chr(random.randint(65, 90))

# 随机颜色1:用于填充背景
def rndColor():
    return (random.randint(64, 255), random.randint(64, 255), random.randint(64, 255))

# 随机颜色2:用于绘制字母
def rndColor2():
    return (random.randint(32, 127), random.randint(32, 127), random.randint(32, 127))

# 240 x 60:
width = 60 * 4
height = 60
image = Image.new('RGB', (width, height), (255, 255, 255))
# 创建Font对象:
font = ImageFont.truetype('c:/Windows/Fonts/Arial.ttf', 36)
# 创建Draw对象:
draw = ImageDraw.Draw(image)
# 填充每个像素:
for x in range(width):
    for y in range(height):
        draw.point((x, y), fill=rndColor())
# 输出文字:
for t in range(4):
    draw.text((60 * t + 10, 10), rndChar(), font=font, fill=rndColor2())
# 模糊:
image = image.filter(ImageFilter.BLUR)
image.save('code.jpg', 'jpeg')
```

# 第 0011 题

第 0011 题： 敏感词文本文件 filtered_words.txt，里面的内容为以下内容，当用户输入敏感词语时，则打印出 Freedom，否则打印出 Human Rights。

```python
filename = 'filtered_words.txt'

with open(filename, 'r') as f:
    words = f.read().split()

while True:
    s = input('请输入单词>>>')
    
    freedom = False
    for word in words:  
        if word in s:
            print('Freedom')
            freedom = True
            break
    if not freedom:
        print('Human Rights') 
```

# 第 0012 题

第 0012 题： 敏感词文本文件 filtered_words.txt，里面的内容 和 0011题一样，当用户输入敏感词语，则用 星号 * 替换，例如当用户输入「北京是个好城市」，则变成「**是个好城市」。

```python
filename = 'filtered_words.txt'

with open(filename, 'r') as f:
    words = f.read().split()

while True:
    s = input('请输入单词>>>')
    freedom = False
    for word in words:  
        if word in s:
            # ** 替换原有字符串 
            nPos = s.index(word)
            nStr = ''.join(['*' for c in word ])
            s = s.replace(word, nStr)
            freedom = True
            print(s)
            break
    if not freedom:
        print('Human Rights') 
```

# 第 0013 题

第 0013 题： 用 Python 写一个爬图片的程序，爬 这个链接里的日本妹子图片 :-)

```python
import requests
from bs4 import BeautifulSoup 

url = 'http://tieba.baidu.com/p/2166231880'
page = requests.get(url)
soup = BeautifulSoup(page.text, 'html.parser')

n = 0
for line in soup.find_all('img', attrs={'class':'BDE_Image'}):   
    img_url = line.get('src')
    img_content = requests.get(img_url).content
    filename = img_url[-10:]
    with open(filename, 'wb') as f:
        f.write(img_content)
    print(n, 'image %s is download completed.' % filename)
    n += 1  
```

# 第 0014 题

第 0014 题： 纯文本文件 student.txt为学生信息, 里面的内容（包括花括号）如下所示：

{
    "1":["张三",150,120,100],
    "2":["李四",90,99,95],
    "3":["王五",60,66,68]
}
请将上述内容写到 student.xls 文件中：

```python
import json
import xlwt

# jsonstr to python dict data
with open('student.txt') as f:  
    dicts = json.loads(f.read())

# create xls
xls = xlwt.Workbook()
table = xls.add_sheet('student')

# write xls
row = 0
col = 0

for k in sorted(dicts.keys()):
    col = 0
    table.write(row, col, k)
    col += 1
    for v in dicts[k]:
        table.write(row, col, v)
        col += 1
    row += 1    
# save to xls
xls.save('student.xls')

```

# 第 0015 题

第 0015 题： 纯文本文件 city.txt为城市信息, 里面的内容（包括花括号）如下所示：

{
    "1" : "上海",
    "2" : "北京",
    "3" : "成都"
}
请将上述内容写到 city.xls 文件中

```python
import json
import xlwt

# jsonstr to python dict data
with open('city.txt') as f:  
    dicts = json.loads(f.read())

# create xls
xls = xlwt.Workbook()
table = xls.add_sheet('city')

# write xls
row = 0
col = 0

for k in sorted(dicts.keys()):
    col = 0
    table.write(row, col, k)
    table.write(row, col + 1, dicts[k])
    row += 1    
# save to xls
xls.save('city.xls')

```

# 第 0016 题

第 0016 题： 纯文本文件 numbers.txt, 里面的内容（包括方括号）如下所示：
[
    [1, 82, 65535],
    [20, 90, 13],
    [26, 809, 1024]
]
请将上述内容写到 numbers.xls 文件中

```python
import json
import xlwt

# jsonstr to python dict data
with open('numbers.txt') as f:  
    lists = json.loads(f.read())

print(lists)
# create xls
xls = xlwt.Workbook()
table = xls.add_sheet('numbers')

# write xls
row = 0
col = 0

for k in lists:
    col = 0
    for v in k:
        table.write(row, col, v)
        col += 1
    row += 1    
# save to xls
xls.save('numbers.xls')
```

# 第 0017 题

第 0017 题：将 第 0014 题中的 student.xls 文件中的内容写到 student.xml 文件中，如下所示：

    <?xml version="1.0" encoding="UTF-8"?>
    <root>
    <students>
    <!-- 
        学生信息表
        "id" : [名字, 数学, 语文, 英文]
    -->
    {
        "1" : ["张三", 150, 120, 100],
        "2" : ["李四", 90, 99, 95],
        "3" : ["王五", 60, 66, 68]
    }
    </students>
    </root>

代码：


```python
import xml.etree.ElementTree as ET
import xlrd

# read xls
xls = xlrd.open_workbook('student.xls')
table = xls.sheet_by_name('student')

dicts = {}
for i in range(table.nrows):
    row = table.row_values(i)
    dicts[row[0]] = row[1:]

# json.dumps无法解决dict的排序问题,所以手动拼接json string
text = ''
text += '\n{\n'
for k in sorted(dicts.keys()):
    lists = dicts[k]
    s = "    \"%s\" : [\"%s\", %d, %d, %d],\n" % (k, lists[0], lists[1], lists[2], lists[3])
    text += s
text += '}\n'
text = text[::-1].replace(',','',1)[::-1]  # 翻转字符串，去掉第一个逗号，再翻转回来

#save to xml
root = ET.Element('root')
students = ET.SubElement(root, 'students')
students.append(ET.Comment(u"""学生信息表  "id" : [名字, 数学, 语文, 英文]""" ))
students.text = text
tree = ET.ElementTree(root)
tree.write('student.xml', encoding='utf-8',  xml_declaration=True)
```

# 第 0018 题

第 0018 题： 将 第 0015 题中的 city.xls 文件中的内容写到 city.xml 文件中，如下所示：

    <?xmlversion="1.0" encoding="UTF-8"?>
    <root>
    <citys>
    <!--
        城市信息
    -->
    {
        "1" : "上海",
        "2" : "北京",
        "3" : "成都"
    }
    </citys>
    </root>

```python
import xml.etree.ElementTree as ET
import json
import xlrd

xls = xlrd.open_workbook('city.xls')
table = xls.sheet_by_name('city')

# get json string
d = {}
for i in range(table.nrows):
    row = table.row_values(i)
    d[row[0]] = row[1]

json_str = json.dumps(d, ensure_ascii=False)

# save to xml
root = ET.Element('root')
citys = ET.SubElement(root, 'citys')
citys.append(ET.Comment('城市信息'))
citys.text = json_str
tree = ET.ElementTree(root)
tree.write('citys.xml', encoding = 'utf-8', xml_declaration = True)
```

# 第 0019 题

第 0019 题： 将 第 0016 题中的 numbers.xls 文件中的内容写到 numbers.xml 文件中，如下所示：

    <?xml version="1.0" encoding="UTF-8"?>
    <root>
    <numbers>
    <!--
        数字信息
    -->

    [
        [1, 82, 65535],
        [20, 90, 13],
        [26, 809, 1024]
    ]

    </numbers>
    </root>

代码：

```python
import xml.etree.ElementTree as ET
import json
import xlrd

xls = xlrd.open_workbook('numbers.xls')
table = xls.sheet_by_name('numbers')

# get json string
l = []
for i in range(table.nrows):
    row = table.row_values(i)
    l.append(row)

json_str = json.dumps(l, ensure_ascii=False)
print(json_str)

# save to xml
root = ET.Element('root')
citys = ET.SubElement(root, 'numbers')
citys.append(ET.Comment('数字信息'))
citys.text = json_str
tree = ET.ElementTree(root)
tree.write('numbers.xml', encoding = 'utf-8', xml_declaration = True)
```

# 第 0020 题

第 0020 题： 登陆中国联通网上营业厅 后选择「自助服务」 --> 「详单查询」，然后选择你要查询的时间段，点击「查询」按钮，查询结果页面的最下方，点击「导出」，就会生成类似于 2014年10月01日～2014年10月31日通话详单.xls 文件。写代码，对每月通话时间做个统计。

```python
import xlrd

xls = xlrd.open_workbook('src.xls')
table = xls.sheet_by_index(0)

total_time = 0
for i in range(1,table.nrows):
    row = table.row_values(i)
    total_time += int(row[3])

print('total time : %d s'  % total_time)
```



