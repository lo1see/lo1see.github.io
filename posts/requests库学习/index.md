# Requests库学习




# requests库使用

## requests方法

| 方法                                     | 描述                              |
| :--------------------------------------- | :-------------------------------- |
| `requests.get(url, params, **args)`      | `发送 GET 请求到指定 url`         |
| `requests.post(url, data, json, **args)` | `发送 POST 请求到指定 url`        |
| `requests.head(url, **args)`             | `发送 HEAD 请求到指定 url`        |
| `requests.patch(url, data, **args)`      | `发送 PATCH 请求到指定 url`       |
| `requests.delete(url, **args)`           | `发送 DELETE 请求到指定 url`      |
| `requests.put(url, data, **args)`        | `发送 PUT 请求到指定 url`         |
| `requests.request(method, url, **args)`  | `向指定的 url 发送指定的请求方法` |



## requests响应信息

| 属性或方法              | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `url`                   | `返回响应的 URL`                                             |
| `text`                  | `返回响应的内容，unicode 类型数据`                           |
| `status_code`           | `返回 http 的状态码，比如 404 和 200（200 是 OK，404 是 Not Found）` |
| `raise_for_status()`    | `如果发生错误，方法返回一个 HTTPError 对象`                  |
| `encoding`              | `解码 r.text 的编码方式`                                     |
| `apparent_encoding`     | `编码方式`                                                   |
| `content`               | `返回响应的内容，以字节为单位`                               |
| `headers`               | `返回响应头，字典格式`                                       |
| `close()`               | `关闭与服务器的连接`                                         |
| `cookies`               | `返回一个 CookieJar 对象，包含了从服务器发回的 cookie`       |
| `history`               | `返回包含请求历史的响应对象列表（url）`                      |
| `json()`                | `返回结果的 JSON 对象 (结果需要以 JSON 格式编写的，否则会引发错误)` |
| `next`                  | `返回重定向链中下一个请求的 PreparedRequest 对象`            |
| `ok`                    | `检查 &#34;status_code&#34; 的值，如果小于400，则返回 True，如果不小于 400，则返回 False` |
| `reason`                | `响应状态的描述，比如 &#34;Not Found&#34; 或 &#34;OK&#34;`                   |
| `request`               | `返回请求此响应的请求对象`                                   |
| `links`                 | `返回响应的解析头链接`                                       |
| `is_permanent_redirect` | `如果响应是永久重定向的 url，则返回 True，否则返回 False`    |
| `is_redirect`           | `如果响应被重定向，则返回 True，否则返回 False`              |
| `iter_content()`        | `迭代响应`                                                   |
| `iter_lines()`          | `迭代响应的行`                                               |
| `elapsed`               | `返回一个 timedelta 对象，包含了从发送请求到响应到达之间经过的时间量，可以用于测试响应速度。比如 r.elapsed.microseconds 表示响应到达需要多少微秒。` |

## get和post区别

`requests.get(url, params, **args)`通过`params(字符串)`进行传参

`requests.post(url, data, json, **args)`通过`data(字典)`或者`json(字典)`进行传参

## UA伪装

修改headers字典里的`User-Agent`参数，例如这样

```python
header = {&#39;User-Agent&#39;:&#39;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML like Gecko) Chrome/46.0.2486.0 Safari/537.36 Edge/13.10586&#39;}
r = requests.get(url, headers=header)
```

```python
headers = {
 &#39;Connection&#39;: &#39;keep-alive&#39;,
 &#39;Cache-Control&#39;: &#39;max-age=0&#39;,
 &#39;sec-ch-ua&#39;: &#39;&#34;Google Chrome&#34;;v=&#34;89&#34;, &#34;Chromium&#34;;v=&#34;89&#34;, &#34;;Not A Brand&#34;;v=&#34;99&#34;&#39;,
 &#39;sec-ch-ua-mobile&#39;: &#39;?0&#39;,
 &#39;Upgrade-Insecure-Requests&#39;: &#39;1&#39;,
 &#39;User-Agent&#39;: &#39;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.114 Safari/537.36&#39;,
 &#39;Accept&#39;: &#39;text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9&#39;,
 &#39;Sec-Fetch-Site&#39;: &#39;same-origin&#39;,
 &#39;Sec-Fetch-Mode&#39;: &#39;navigate&#39;,
 &#39;Sec-Fetch-User&#39;: &#39;?1&#39;,
 &#39;Sec-Fetch-Dest&#39;: &#39;document&#39;,
 &#39;Referer&#39;: &#39;https://bj.ke.com/&#39;,
 &#39;Accept-Language&#39;: &#39;zh-CN,zh;q=0.9&#39;,
}
response = requests.get(url, headers=headers)
```



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/requests%E5%BA%93%E5%AD%A6%E4%B9%A0/  

