[博客园wp](https://www.cnblogs.com/h0cksr/p/16189739.html) & [先知社区wp](https://xz.aliyun.com/t/11073)

仅仅能控制环境变量，参考p神的文章，本题是在可控环境变量的条件下执行sh -c "echo hfctf2022"，但是本题和p神的环境不一样。

### Nginx临时文件
Nginx用于反向代理时，从后端web应用接受响应并缓存到本地，然后发往客户端。响应较小时缓存到内存，如果响应过大则或发往客户端速度太慢导致内存缓冲区满，则会将一部分缓冲区内容写入临时文件中，腾出缓冲区部分空间。[source code分析](https://blog.csdn.net/kai_ding/article/details/21297101)
