看了源码，发现是jwt伪造，但是带secret_key的话必须要得到secret_key才能正确伪造jwt。

在`/`路由下可以看到`tpl, err := template.New("").Parse("Logged in as " + acc.id)`，这里是直接拼接，且id字段用户完全可控，因此可能存在go ssti。
go ssti种传入模板的是一整个结构体变量，可以直接访问该变量内部的字段，可以参考[这里](https://tyskill.github.io/posts/gossti/)

先register，id设置为`{{.secret_key}}`，然后用注册的账号去auth，得到jwt token，然后拿jwt token访问root，获得secret_key，然后伪造jwt token并访问flag路由即可。

[jwt](https://jwt.io/)

buu环境开不起来，先不复现了。
