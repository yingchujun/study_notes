#### 1. 环境安装：
* pip install scrapy 

#### 2. 整体结构图
##### 2.1 官方架构 
![](note_img/Pasted%20image%2020230513130338.png)
##### 2.2 理解
![](note_img/Pasted%20image%2020230513130723.png)
#### 3. 基本使用：
* 创建项目： scrapy stratproject  【项目名】
	* cd 【项目名】
 
``` python
文件目录：
scrapy.cfg ：项目的配置文件
mySpider/ ：项目的Python模块，将会从这里引用代码
mySpider/items.py ：项目的目标文件
mySpider/pipelines.py ：项目的管道文件
mySpider/settings.py ：项目的设置文件
mySpider/spiders/ ：存储爬虫代码目录>)
```

* 创建爬虫： scrapy genspider  【爬虫名】 【允许爬取的域名(之后可修改)】
	* 在到新创建的爬虫文件写需要的爬虫程序
* 运行爬虫： scrapy crawl 【爬虫名】
##### 31. 其他指令
* 清楚日志信息： 
	* scrapy crawl 【爬虫名】--nolog
	* 在settings.py 文件设置 LOG_LEVEL = “等级” ：选择日志输出的信息
		* CRITICAL - 严重错误
		* ERROR - 一般错误
		* WARNING - 警告信息
		* INFO - 一般信息
		* DEBUG - 调试信息
	  * 配置文件其他日志设置
		* LOG_ENABLED 默认: True，启用logging
		*  LOG_ENCODING 默认: ‘utf-8’，logging使用的编码
		* LOG_FILE 默认: None，在当前目录里创建logging输出文件的文件名
#### 4. 持久化存储：
* 基于终端指令的持久话存储
	* 要求：只可以将parse方法的返回值存储到本地文件中
	* 指令：scrapy crawl 【爬虫名】-o 【路径】
 * 基于管道
	 * 流程：
		* 数据解析
		*  将解析的数据封装存储到item类型的对象中(items.py)
		*  在item.py中定义相关属性
		* 将item类型对象提交给管道进行持久化存储的操作(pipelines.py)
			* process_item 的return item表示将item传递给下一个即将执行的管道  
		* 在管道类的process_item中要将其接收到的item对象中存储的数据进行持久化存储
		*  在配置文件中开启管道  
#### 5. 基于spider的全站数据爬取：
* 就是将网站中某板块下的全部页码对应的页面数据进行爬取
* 将所有的页码对应的url添加到start_urls 【不推荐】
* 自行手动进行请求发送 【推荐】
``` python
class MovieSpider(scrapy.Spider):
    name = 'movie'
    # allowed_domains = ['www.521609.com']
    start_urls = ['https://www.meijutt.tv/file/list2.html']

    # 生成一个通用的url模板(不可变)
    page_num = 2
    url = 'https://www.meijutt.tv/file/list2-{}.html'

    def parse(self, response):
        movie_list = response.xpath('//ul[@class="list_20"]/li/a')

        for movie in movie_list:
            movie_name = movie.xpath('./text()').extract_first()
            print(movie_name)

        if self.page_num %3C= 10:
            new_url = self.url.format(self.page_num)
            self.page_num += 1
            # 手动请求发送 callback 回调函数用作于数据解析
            yield scrapy.Request(url=new_url, callback=self.parse)
```







### scrapy yield的作用：
**scrapy中的yield还是生成器。  
通过 yield 来发起一个请求，并通过 callback 参数为这个请求添加回调函数，在请求完成之后会将响应作为参数传递给回调函数。 scrapy框架会根据 yield 返回的实例类型来执行不同的操作：**
* a. 如果是 scrapy.Request 对象，scrapy框架会去获得该对象指向的链接并在请求完成后调用该对象的回调函数。
* b. 如果是 scrapy.Item 对象，scrapy框架会将这个对象传递给 pipelines.py做进一步处理。

**先说scrapy的spider：**
* Scrapy为Spider的 start_urls 属性中的每个URL创建了 scrapy.Request 对象，并将 parse 方法作为回调函数(callback)赋值给了Request。
* Request对象经过调度，执行生成 scrapy.http.Response 对象并送回给spider parse() 方法。
**当parse中使用了yield，parse就会被当成一个生成器。**
* 1. 因为使用的yield，而不是return。parse函数将会被当做一个生成器使用。scrapy会逐一获取parse方法中生成的结果，并判断该结果是一个什么样的类型； 
* 2. 如果是request则加入爬取队列，如果是item类型则使用pipeline处理，其他类型则返回错误信息。 
* 3. scrapy取到第一部分的request不会立马就去发送这个request，只是把这个request放到队列里，然后接着从生成器里获取； 
* 4. 取尽第一部分的request，然后再获取第二部分的item，取到item了，就会放到对应的pipeline里处理； 
* 5. parse()方法作为回调函数(callback)赋值给了Request，指定parse()方法来处理这些请求 scrapy.Request(url, callback=self.parse) 
* 6. Request对象经过调度，执行生成 scrapy.http.response()的响应对象，并送回给parse()方法，直到调度器中没有Request（递归的思路） 
* 7. 取尽之后，parse()工作结束，引擎再根据队列和pipelines中的内容去执行相应的操作； 
* 8. 程序在取得各个页面的items前，会先处理完之前所有的request队列里的请求，然后再提取items。 
* 9. 这一切的一切，Scrapy引擎和调度器将负责到底。