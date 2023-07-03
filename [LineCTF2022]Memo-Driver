看源码，看到了Jinja2，大致感觉是Python-SSTI，不过框架不是Flask。看了一圈没什么大问题，感觉和starlette有点关系
网上一搜starlette ssti，找到了https://github.com/encode/starlette/issues/1325和CVE-2021-23336
这样的话读取文件的路径就是可控的了，过滤了`.`导致不能通过`../`返回上层目录，尝试构造url：
```
/view?c224685b7f75bfccb50ee46dd3f807ec=flag;%2F%2E%2E
```
解决
```
def view(request):

    try:
        clientId = "id"
        print("request.url:", request.url)
        print("request.url.query", request.url.query)
        print("params:", request.query_params)
        print("unquote params:", unquote(request.query_params[clientId]))
        if '&' in request.url.query or '.' in request.url.query or '.' in unquote(request.query_params[clientId]):
            raise
        
        filename = request.query_params[clientId]
        print("filename:", filename)
        print("keys:", request.query_params.keys())
        path = './memo/' + "".join(request.query_params.keys()) + '/' + filename
        print("path:", path)
    
    except:
        pass
    
    return JSONResponse({"a":1})
```
unquote之前都是raw的url，if里面request.query_params[clientId]是正常解析的，request.query_params.keys()对分号解析有问题。

另外还有一种改Host的，见https://github.com/aszx87410/huli-blog/blob/master/source/_posts/linectf-2022-writeup.md

结果发现不是ssti，不过为什么%2E用.就不行呢？
