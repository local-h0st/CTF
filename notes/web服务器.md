## 关于web server
Nginx、Apache等web server用于静态资源的分发，动态内容是处理不了的，用户请求某个静态资源，web server找到后发回给用户，如果请求动态资源就需要转发给WSGI(Python web应用)或CGI程序(通用)

## 关于中间件
以下web server用Nginx举例，当然Apache也是一样的，不过反向代理负载均衡什么的不如Nginx
### WSGI
Python本身是支持HTTP的，但是效率很低，生产环境直接Python或Python+Nginx都是性能不佳，需要中间件，例如Python+uWSGI+Nginx

WSGI(Python Web Server Gateway Interface)是一套接口标准协议/规范，Gateway即网关，所以WSGI是转换不同协议的一套规范，用于连接web服务器(如Nginx)和Python web应用/web框架(诸如Flask等)，以保证不同Web服务器可以和不同的Python程序之间相互通信。Flask，webpy，Django、CherryPy等框架自带了WSGI，但是性能都不好。Gunicorn和uWSGI等程序实现了WSGI协议

uWSGI是程序，是WSGI Server，支持包括WSGI、HTTP在内的多种协议，另外它内部还有一种消息传递的协议叫uwsgi，和WSGI完全是两种不相干的东西。uWSGI对动态内容的处理性能很好，其实单独Python+uWSGI没有Nginx，也能工作，但是Nginx对静态内容响应很好，还能负载均衡，因此最好是Nginx做web server，处理静态资源等，并把动态内容转发给uWSGI，从而达到更快的响应。

### CGI & FastCGI
CGI(Common Gateway Interface)和WSGI一样，也是一种网关协议，不过WSGI专用于Python，CGI适用于几乎所有语言实现的web应用。FastCGI是对CGI的改进，提高了性能。

用户请求动态资源时，web server将请求转发给CGI程序，CGI程序启动对应的进程(例如启动一个Python解释器/php解释器并运行我们编写的web应用)去处理来自web server的消息，把结果返回给CGI程序，CGI程序再转发给web server

PHP-CGI和PHP-FPM分别是实现了CGI协议和FastCGI协议的中间件程序，连接PHP脚本和web server。用户请求某个php脚本时请求就会被web server转发给中间件，中间件就会开个进程/线程并执行该脚本，得到结果返回给web server。

### Go + Nginx
go本身内置了高性能web服务框架，无需中间件，仅需要使用Nginx提高其负载均衡等能力
