---
title: Python 反射机制
date: 2020/06/19
tags:
  - Python
  - 面试
categories:
  - Python
abbrlink: 
description: 本文主要介绍 Python 中反射机制的实现及使用,以作备忘.
---

Python 内置了一些有关反射机制的函数,方便

- `hasattr(obj,attr_name)`: 判断 obj 对象中是否有 attr_name 属性/方法
- `getattr(obj,attr_name)`: 获取 obj 中 attr_name 的属性/方法
- `setattr(obj,attr_name, some_attr)`: 设置 obj 中 attr_name 属性/方法为 some_attr
- `delattr(obj,attr_name)`: 删除指定 obj 的 attr_name 属性/方法

反射机制为动态导入模块或动态执行函数提供了方便.

如下示例是对 `Crawler` 类添加以 `crawl_` 开头的方法获取代理的示例,代码参考自 [崔庆才丨静觅 的 GitHub](https://github.com/Germey/ProxyPool)

```python
# crawler.py
class ProxyMetaclass(type):
    # 重写 `__new__` 方法后,会在新建的 `Crawler` 中添加'__CrawlFunc__'和'__CrawlFuncCount__' 属性,其中'__CrawlFuncCount__' 为新建Crawler对象的方法个数(本例中是以'crawl_'开头的方法),'__CrawlFunc__'是新建Crawler对象的方法列表
    # 如果添加或删除代理,只需要在 Crawler 类中添加以'crawl_'开头的方法即可
    def __new__(cls, name, bases, attrs):
        count = 0
        attrs['__CrawlFunc__'] = []
        for k, v in attrs.items():
            if 'crawl_' in k:
                attrs['__CrawlFunc__'].append(k)
                count += 1
        attrs['__CrawlFuncCount__'] = count
        return type.__new__(cls, name, bases, attrs)

class Crawler(object, metaclass=ProxyMetaclass): # 设置 metaclass(原类) 为 ProxyMetaclass
    def get_proxies(self, callback):
        proxies = []
        for proxy in eval("self.{}()".format(callback)): # 这里使用 `eval` 执行了参数内容,并返回表达式的值,很巧妙的传入 `callback` 参数,从而使得实例可以调用多个方法,从而从多个网站中获取代理
            print('成功获取到代理', proxy)
            proxies.append(proxy)
        return proxies

    # def crawl_daxiang(self):
    #     url = 'http://vtp.daxiangdaili.com/ip/?tid=559363191592228&num=50&filter=on'
    #     html = get_page(url)
    #     if html:
    #         urls = html.split('\n')
    #         for url in urls:
    #             yield url

    def crawl_daili66(self, page_count=4):
        """
        从代理 66 获取代理
        :param page_count: 页码
        :return: 代理
        """
        start_url = 'http://www.66ip.cn/{}.html'
        urls = [start_url.format(page) for page in range(1, page_count + 1)]
        for url in urls:
            print('Crawling', url)
            html = get_page(url)
            if html:
                doc = pq(html)
                trs = doc('.containerbox table tr:gt(0)').items()
                for tr in trs:
                    ip = tr.find('td:nth-child(1)').text()
                    port = tr.find('td:nth-child(2)').text()
                    yield ':'.join([ip, port])

    def crawl_proxy360(self):
        """
        从 Proxy360 获取代理
        :return: 代理
        """
        start_url = 'http://www.proxy360.cn/Region/China'
        print('Crawling', start_url)
        html = get_page(start_url)
        if html:
            doc = pq(html)
            lines = doc('div[name="list_proxy_ip"]').items()
            for line in lines:
                ip = line.find('.tbBottomLine:nth-child(1)').text()
                port = line.find('.tbBottomLine:nth-child(2)').text()
                yield ':'.join([ip, port])

# 使用
class Getter():
    def __init__(self):
        self.redis = RedisClient()
        self.crawler = Crawler()

    def is_over_threshold(self):
        """
        判断是否达到了代理池限制
        """
        if self.redis.count() >= POOL_UPPER_THRESHOLD:
            return True
        else:
            return False

    def run(self):
        print('获取器开始执行')
        if not self.is_over_threshold():
            for callback_label in range(self.crawler.__CrawlFuncCount__):
                callback = self.crawler.__CrawlFunc__[callback_label]
                # 获取代理
                proxies = self.crawler.get_proxies(callback)
                sys.stdout.flush()
                for proxy in proxies:
                    self.redis.add(proxy)
```
