源码一看是flask加上template，感觉和模板有关，总不能还是ssti吧。大致看了一眼app.py，好像没什么逻辑漏洞。
在Google的帮助下发现了FLAG位于base.html页脚，很不起眼的一个位置，所以我们要做的就是以admin身份登陆，即可得到flag。

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
我估计点击上传的时候自动会生成一个`/static/image/`开头的img_url然后再post给app.py，翻了js发现console还会log出img_url信息。
js会fetch /image/upload，app.py中如下：
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
流程是先`GET /image`，返回一个普通页面，选择图片后页面js去`POST /image/upload`，app.py执行`/image/upload`路由后返回结果给js，
js再`console.log(img_url)`，并使得上传按钮可用。用户点击上传后是`POST /image`，`/image`路由收到后验证前缀是否是`/static/image/`，
验证通过才存储图片。
