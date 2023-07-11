[博客园wp](https://www.cnblogs.com/h0cksr/p/16189739.html) & [先知社区wp](https://xz.aliyun.com/t/11073)

仅仅能控制环境变量，参考p神的文章，本题是在可控环境变量的条件下执行sh -c "echo hfctf2022"，但是本题和p神的环境不一样。

### Nginx临时文件
Nginx用于反向代理时，从后端web应用接受响应并缓存到本地，然后发往客户端。响应较小时缓存到内存，如果响应过大则或发往客户端速度太慢导致内存缓冲区满，则会将一部分缓冲区内容写入临时文件中，腾出缓冲区部分空间。[source code分析](https://blog.csdn.net/kai_ding/article/details/21297101)

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
当目录下没有test.txt时，测试结果如图：
![nil](https://github.com/local-h0st/CTF/blob/30ede03357edb15579ffb06cb3adb9d341bf3576/writeups/pics/test_unlink_01.png)

如果预先存在test.txt则不会存在/proc/pid/fd/3这个软链接，且a.out本身输出buff为空，如图：
![nil](https://github.com/local-h0st/CTF/blob/main/writeups/pics/test_unlink_02.png)
我发现，在getchar挡住，close未执行时，目录下test.txt就已经消失不见了。
