源码一看是flask加上template，感觉和模板有关，总不能还是ssti吧。大致看了一眼app.py，好像没什么逻辑漏洞。在Google的帮助下发现了FLAG位于base.html页脚，很不起眼的一个位置，所以我们要做的就是以admin身份登陆，即可得到flag。

翻了翻模板，感觉image.html有点怪：
```
{% extends "base.html" %}

{% block content %}
<div class="title is-3">{{ image.title }}</div>

<img src={{ image.url }} class="mb-3">
<input hidden id="imgId" value="{{ image.id }}">
{% if not shared %}
<div class="control">
    <button id="shareButton" class="button is-success">Share (to admin)</button>
</div>

{% endif %}

<script src="/static/script/main.js"></script>
{% endblock content %}
```
最开始我以为是`{{ image.title }}`，但是发现没问题，Google了一下才发现`<img src={{ image.url }} class="mb-3">`没加引号！

在app.py中寻找image.url是否可控，注意到这里：
```
@app.route('/image', methods=['GET', 'POST'])
@login_required
def create_image():
    if request.method == 'POST':
        title = request.form.get('title')
        img_url = request.form.get('img_url')
        
        if title and img_url:
            if not img_url.startswith('/static/image/'):
                flash('Image creation failed')
                return redirect(url_for('create_image'))

            image = Image(title=title, url=img_url, owner=current_user)
            db.session.add(image)
            db.session.commit()
            res = redirect(url_for('index'))
            res.headers['X-ImageId'] = image.id
            return res
        return redirect(url_for('create_image'))

    elif request.method == 'GET':
        return render_template('create_image.html')
```
我估计点击上传的时候自动会生成一个`/static/image/`开头的img_url然后再post给app.py，翻了js发现console还会log出img_url信息。js会fetch /image/upload，app.py中如下：
```
@app.route('/image/upload', methods=['POST'])
@login_required
def upload_image():
    img_file = request.files['img_file']
    if img_file:
        ext = os.path.splitext(img_file.filename)[1]
        if ext in ['.jpg', '.png']:
            filename = uuid4().hex + ext

            img_path = os.path.join(app.config.get('UPLOAD_FOLDER'), filename)
            img_file.save(img_path)

            return jsonify({'img_url': f'/static/image/{filename}'}), 200

    return jsonify({}), 400
```
流程是先`GET /image`，返回一个普通页面，选择图片后页面js去`POST /image/upload`，app.py执行`/image/upload`路由，保存图片到本地并返回一个img_url给前端js，js再`console.log(img_url)`并设置表单img_url，然后使得上传按钮可用。用户点击上传后是`POST /image`，内容包括了表单中刚刚设置的`img_url`等(可以尝试在这里抓包修改img_url)，`/image`路由收到后验证前缀是否是`/static/image/`，验证通过才存储图片。后续显示图片的话全是通过img_id来找到img_url，再加载图片。

因此抓包改img_url前缀必须是`/static/image/`才能被db顺利存储，后续访问才能找到这条记录

不会利用，Google一下。

注意到app.py中的Content-Security-Policy，图片不允许外链。

xsleaks放几个链接：[1](https://www.scuctf.com/ctfwiki/web/9.xss/xsleaks/) [2](https://xz.aliyun.com/t/11306) [3](https://www.anquanke.com/post/id/176049) [4](https://lists.archive.carbon60.com/apache/users/316239)

Scroll-To-Text Fragment：就是文本高亮，但是STTF有xsleaks侧信道攻击。别人的wp还用到了img标签[lazy-loading](https://www.zhangxinxu.com/wordpress/2019/09/native-img-loading-lazy/)属性。



