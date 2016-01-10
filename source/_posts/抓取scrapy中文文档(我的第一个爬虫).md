title: '抓取scrapy中文文档(我的第一个爬虫)'
date: 2014-05-22 19:31
tags: 爬虫
---


##1.新建scrapy项目
<!--more-->

      说明:本文档默认大家已经安装scripy,如果还没,请参考[http://doc.scrapy.org/en/latest/intro/install.html](http://doc.scrapy.org/en/latest/intro/install.html)

```python
scrapy startproject scrapyDoc
```


##2.新建spider

    1) 进入项目 
```python
cd scrapyDoc
```

    2) 新建spider

```python
scrapy genspider ScrapyDoc scrapy-chs.readthedocs.org
```
即: name = 'ScrapyDoc'
allowed_domains = ['scrapy-chs.readthedocs.org']   

##3)编写代码

   1)item.py
 
```python
# Define here the models for your scraped items
#
# See documentation in:
# http://doc.scrapy.org/topics/items.html

from scrapy.item import Item, Field

class ScrapydocItem(Item):
    title = Field()
    link = Field()
    url = Field()
```
   2)ScrapyDocSpider.py

```python
from scrapy.spider import BaseSpider
from scrapy.selector import HtmlXPathSelector
from scrapy.http import Request

from ScrapyDoc.items import ScrapydocItem
import os

#设置默认编码
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

class ScrapyDocSpider(BaseSpider):
    name = 'ScrapyDoc'
    allowed_domains = ['scrapy-chs.readthedocs.org']

    start_urls = ['http://scrapy-chs.readthedocs.org/zh_CN/latest']

    def parse(self,response):

        if response.url.split("/")[-1] == '':
            filename = response.url.split("/")[-2]
        else :
            dirname = response.url.split("/")[-2]
            #判断是否有此目录,如果没有就新建
            if os.path.isdir(dirname) == False:
                os.mkdir(dirname)
            filename = '/'.join(response.url.split("/")[-2:])

        #保存文件
        open(filename,'wb').write(response.body)

        sel = HtmlXPathSelector(response)
        sites = sel.select('//li[@class="toctree-l1"]')
        for site in sites:
            item = ScrapydocItem()
            item['title'] = site.select('a/text()').extract()

            #生成连接 begin   ,因为从页面提取的连接都是相对地址
            link = site.select('a/@href').extract()[0]
            url = response.url

            #地址形式是否为 ../spiders.html 这种形式,需要回到上级地址
            if link.split('/')[0] == '..':
                url2 = '/'.join(url.split('/')[0:-2]) + '/' + '/'.join(link.split('/')[1:])
            else:
                url2 = '/'.join(url.split('/')[0:-1]) + '/' + link

            item['link'] = [url2]
            #生成连接 end 
            yield item

            #返回多个request
            yield Request(url=url2,callback=self.parse)

        return


```


##4)运行程序

    1) 回到scrapy项目主目录
    2) 新建一个文件夹(我习惯将所有下载的文件保存在一个单独文件夹下)

```python
mkdir doc
```
    3)进入doc文件夹  

```python
cd doc
```
    4) 运行程序

```html
scrapy crawl ScrapyDoc 
```



现在终于将scrapy中文文档保存在本地了,不过提取到的item还没存储,可以输入如下命令行保存item

```html
scrapy crawl ScrapyDoc -o s.json -t json
```




















