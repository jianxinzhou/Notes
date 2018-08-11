# Flask 基本原理与核心知识

## 1. 响应对象：Response

+ 在浏览器中对如下 url 进行请求时，浏览器会显示「Hello, world!」

```python
@app.route('/hello')
def hello():
    return 'Hello, world!'
```

+ 同样对如下 url 进行请求，这次浏览器什么都不会显示，原因在于
  + server 除了返回 `'<html></html>'` 字符串外，还会返回一系列的附加信息：
    + status code，例如 200
    + content-type（位于 http headers 中），作用是告诉浏览器如何解析返回的主体内容，也就是说视图函数返回的永远是一串字符串，但是浏览器如何对其解析，需要通过 content-type 这个字段来告知，
  + content-type 默认值是 `text/html`，因此浏览器对返回的内容会以 HTML 格式来解析，那么对于以下例子显示为空，就再自然不过了  

```python
@app.route('/hello')
def hello():
    return '<html></html>'
```

+ flask 视图函数到底返回了什么？
  + flask 会将返回的主体内容，以及之前提过的 status code，content type 等内容封装成一个 Response 对象返回
  + 视图函数本质上返回的永远是 Response 对象

```python
@app.route('/hello')
def hello():
    # 此时会显示，因为我们设置了 content type 是普通文本
    headers = {
        'content-type': 'text/plain'
    }
    response = make_response('<html></html>', 404)
    response.headers = headers
    return response
```

+ 重定向例子

```python
@app.route('/hello')
def hello():
    headers = {
        'content-type': 'text/plain',
        'location': 'http://www.baidu.com'
    }
    response = make_response('<html></html>', 302)
    response.headers = headers
    return response
```

+ 当作为 API 时，提供 Json 格式的字符串
  + content type 设置为 `application/json`

+ 简单创建 Response 对象的方式

```python
@app.route('/hello')
def hello():
    headers = {
        'content-type': 'text/plain',
        'location': 'http://www.baidu.com'
    }
    
    # 本质在于当返回元组时，flask 内部会将其转换为 Response 对象返回
    return '<html></html>', 302, headers
```

