# BBS+Blog项目开发

## 1.项目需求

``````
1 基于ajax和用户认证组件实现登录验证
2 基于ajax和form组件实现注册功能
3 系统首页文章列表的渲染
4 个人站点页面设计
5 文章详细页的继承
6 点赞与踩灭
7 评论功能
8 富文本编辑器的使用
9 防止xss攻击
``````

![](https://images2018.cnblogs.com/blog/1356841/201808/1356841-20180830154316945-1613552450.png)

## 2.项目详情

### 2.1 数据库设计

核心代码：

```
1.继承AbstractUser
2.中介模型
3.联合唯一
             ]
```



![](https://images2018.cnblogs.com/blog/1356841/201808/1356841-20180830154447123-1865221472.png)

``````python
from django.db import models

# Create your models here.


from django.contrib.auth.models import AbstractUser


class UserInfo(AbstractUser):
    """
    用户信息
    """
    nid = models.AutoField(primary_key=True)
    telephone = models.CharField(max_length=11, null=True, unique=True)
    avatar = models.FileField(upload_to='avatars/', default="avatars/default.png")
    create_time = models.DateTimeField(verbose_name='创建时间', auto_now_add=True)

    blog = models.OneToOneField(to='Blog', to_field='nid', null=True, on_delete=models.CASCADE)

    def __str__(self):
        return self.username


class Blog(models.Model):
    """
    博客信息
    """
    nid = models.AutoField(primary_key=True)
    title = models.CharField(verbose_name='个人博客标题', max_length=64)
    site_name = models.CharField(verbose_name='站点名称', max_length=64)
    theme = models.CharField(verbose_name='博客主题', max_length=32)

    def __str__(self):
        return self.title


class Category(models.Model):
    """
    博主个人文章分类表
    """
    nid = models.AutoField(primary_key=True)
    title = models.CharField(verbose_name='分类标题', max_length=32)
    blog = models.ForeignKey(verbose_name='所属博客', to='Blog', to_field='nid', on_delete=models.CASCADE)

    def __str__(self):
        return self.title


class Tag(models.Model):
    nid = models.AutoField(primary_key=True)
    title = models.CharField(verbose_name='标签名称', max_length=32)
    blog = models.ForeignKey(verbose_name='所属博客', to='Blog', to_field='nid', on_delete=models.CASCADE)

    def __str__(self):
        return self.title


class Article(models.Model):
    nid = models.AutoField(primary_key=True)
    title = models.CharField(max_length=50, verbose_name='文章标题')
    desc = models.CharField(max_length=255, verbose_name='文章描述')
    create_time = models.DateTimeField(verbose_name='创建时间', auto_now_add=True)
    content = models.TextField()

    comment_count = models.IntegerField(default=0)
    up_count = models.IntegerField(default=0)
    down_count = models.IntegerField(default=0)

    user = models.ForeignKey(verbose_name='作者', to='UserInfo', to_field='nid', on_delete=models.CASCADE)
    category = models.ForeignKey(to='Category', to_field='nid', null=True, on_delete=models.CASCADE)
    tags = models.ManyToManyField(
        to="Tag",
        through='Article2Tag',
        through_fields=('article', 'tag'),
    )

    def __str__(self):
        return self.title


class Article2Tag(models.Model):
    nid = models.AutoField(primary_key=True)
    article = models.ForeignKey(verbose_name='文章', to="Article", to_field='nid', on_delete=models.CASCADE)
    tag = models.ForeignKey(verbose_name='标签', to="Tag", to_field='nid', on_delete=models.CASCADE)

    class Meta:
        unique_together = [
            ('article', 'tag'),
        ]

    def __str__(self):
        v = self.article.title + "---" + self.tag.title
        return v


class ArticleUpDown(models.Model):
    """
    点赞表
    """

    nid = models.AutoField(primary_key=True)
    user = models.ForeignKey('UserInfo', null=True, on_delete=models.CASCADE)
    article = models.ForeignKey("Article", null=True, on_delete=models.CASCADE)
    is_up = models.BooleanField(default=True)

    class Meta:
        unique_together = [
            ('article', 'user'),
        ]


class Comment(models.Model):
    """

    评论表

    """
    nid = models.AutoField(primary_key=True)
    article = models.ForeignKey(verbose_name='评论文章', to='Article', to_field='nid', on_delete=models.CASCADE)
    user = models.ForeignKey(verbose_name='评论者', to='UserInfo', to_field='nid', on_delete=models.CASCADE)
    content = models.CharField(verbose_name='评论内容', max_length=255)
    create_time = models.DateTimeField(verbose_name='创建时间', auto_now_add=True)
    parent_comment = models.ForeignKey('self', null=True, on_delete=models.CASCADE)

    def __str__(self):
        return self.content

``````

### 2.2 站点管理

```python
from django.contrib import admin

# Register your models here.
from blog import models

admin.site.register(models.UserInfo)
admin.site.register(models.Article)
admin.site.register(models.Article2Tag)
admin.site.register(models.ArticleUpDown)
admin.site.register(models.Blog)
admin.site.register(models.Category)
admin.site.register(models.Comment)
admin.site.register(models.Tag)
```

### 2.3 注册

核心代码：

1.form组件

`````python
1.form组件
    class RegForm(forms.Form):pass
    局部钩子 全局钩子

2.上传头像 avatar
    图像预览
        var reader = new FileReader();
    上传文件
        formdata = new FormData();

3.用户文件配置
    avatar = models.FileField(upload_to='avatars/', default='avatars/default.png')
    MEDIA_ROOT = os.path.join(BASE_DIR, 'blog', 'media')
    MEDIA_URL = '/media/'
    re_path(r'media/(?P<path>.*)$',serve,{'document_root':settings.MEDIA_ROOT})

`````



![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215202023-639451677.png)

### 2.4 登录

核心代码：

``````python
1.验证码
    随机生成5个字符，0-9 a-z A-Z

    pip install pillow
    from PIL import Image, ImageDraw, ImageFont
        image = Image.new()

    在内存中生成图片直接返回
    from io import BytesIO
        f = BytesIO()

2.request.session['valid_str'] = valid_str
    存在session中，为了之后登录，验证是否通过

3.验证码点击刷新：
    $('#valid_img').click(function () {
        $(this)[0].src += '?'
    });

4.认证组件
    valid_str = request.session.get('valid_str')
    if valid_str.upper() == valid_code.upper():
        user = auth.authenticate(username = user, password = pwd)
        if user:
            auth.login(request, user)
``````

![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215140660-111765331.png)

 ### 2.5 网站首页

核心代码：

``````
1.bootstrap搭建页面

2.导航条
    登录：   username / 注销
    未登录： 登录 / 注册

3.for循环
     {% for article in article_list %}
     {% endfor %}
``````

![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215236923-1894987618.png)

网页底部分页器：

![深度截图_选择区域_20180911200719](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215251033-1672110239.png)

### 2.6 个人站点

``````
1.文章列表，分类列表，标签列表，日期归档列表
    文章列表：     /blog/egon/
    分类列表：     /blog/egon/cate/python
    标签列表：     /blog/egon/tag/生活
    日期归档列表： /blog/egon/archive/2018-06

2.模板继承
    {% extends 'base.html' %}

    {% block content %}
    {% endblock content%}}

3.自定义标签
    /blog/templatetags/my_tag.py

    @register.inclusion_tag('menu.html')
    def get_menu(username):
        ...
        return {} # 去渲染 menu.html

4.分组查询 .annotate() / extra()应用
    多表分组
        tag_list = Tag.objects.filter(blog=blog).annotate(
            count = Count('article')).values_list('title', 'count')

    单表分组 / DATE_FORMAT() /  extra()
        date_list = Article.objects.filter(user=user).extra(
            select={"create_ym": "DATE_FORMAT(create_time,'%%Y-%%m')"}).values('create_ym').annotate(
            c = Count('nid')).values_list('create_ym', 'c')

5. 时间、区域配置
     TIME_ZONE = 'Asia/Shanghai'
     USE_TZ = False
``````

![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215310812-577258899.png)

### 2.7 文章详情

核心代码：

``````
1.模板继承
    article = Article.objects.filter(pk=article_id).first()
    {% extends 'base.html' %}
    {% block content %}
         ...
        {{ article.articledetail.content|safe }}
    {% endblock content %}
``````

![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215337094-341474826.png)

### 2.8 后台管理

核心代码：

``````python
1.支持文章编辑
2.支持富文本编辑器（支持渲染已有文章）
3.支持删除文章
4.防止Xss攻击（基于BS4）
 # 防止xss攻击,过滤script标签
        soup = BeautifulSoup(content, "html.parser")
        for tag in soup.find_all():

            if tag.name == "script":
                tag.decompose()	
``````



![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215442402-358164894.png)



### 2.9 批量建立测试数据

核心代码：

``````python
1.定制建立测试数据
2.初始化得到100条用户信息，以及为100个客户分别添加文章、博客的详细信息
def supreme(request):
    """
    这是数据初始化视图，将会生成99条测试数据，用户名为管理员输入的字符后，加1_99.
    密码为管理员填入的密码
    :param request:
    :return:
    """
    tag_title = ['BASIC', 'C',
                 'C++', 'PASCAL', 'FORTRAN', 'LISP', 'Prolog', 'CLIPS', 'OpenCyc', 'Fazzy', 'Python', 'PHP', 'Ruby',
                 'Lua']
    category_titles = ['非技术区',
                       '软件测试',
                       '代码与软件发布',
                       '计算机图形学',
                       '游戏开发',
                       '程序人生',
                       '求职面试',
                       '读书区',
                       '转载区',
                       'Windows',
                       '翻译区',
                       '开源研究',
                       'Flex']
    userlist = []
    bolglist = []
    article_list = []
    tag_list = []
    cat_list = []
    article2tag_list = []

    if request.is_ajax():

        form = UserForm(request.POST)
        response = {'user': None, 'msg': None}
        if form.is_valid():

            response['user'] = form.cleaned_data.get('user')
            # 生成一条用户记录
            user = form.cleaned_data.get('user')

            pwd = form.cleaned_data.get('pwd')
            email = form.cleaned_data.get('email')
            avatar_obj = request.FILES.get('avatar')
            extra = {}
            if avatar_obj:
                extra['avatar'] = avatar_obj
                with transaction.atomic():
                    for i in range(1, 100):
                        Blog.objects.create(title='blog%s' % i)

                        UserInfo.objects.create_user(username=user + str(i), password=pwd, email=email, **extra)

                        UserInfo.objects.filter(username=user + str(i)).update(blog_id=i)

                        for title in tag_title:
                            tag_list.append(Tag(title=title, blog_id=i))

                        for cat in category_titles:
                            cat_list.append(Category(title=cat, blog_id=i))
                        import random
                        for c in range(1, 7):
                            with open('/home/hyc/PycharmProjects/cnblog/static/superme/%s.txt' % c, 'r',
                                      encoding='utf-8') as f:
                                content = f.read()

                            soup = BeautifulSoup(content, "html.parser")
                            for tag in soup.find_all():
                                if tag.name == "script":
                                    tag.decompose()
                            desc = soup.text[0:150] + "..."
                            c_id = 13 * (i - 1) + random.randrange(1, 9)

                            article_list.append(
                                Article(desc=desc, title='article%s' % c, content=content, user_id=i, category_id=c_id))

                    Category.objects.bulk_create(cat_list)
                    Tag.objects.bulk_create(tag_list)
                    Article.objects.bulk_create(article_list)

        else:
            response['msg'] = form.errors
        return JsonResponse(response)

    my_from = UserForm

    return render(request, 'supreme.html', {'from': my_from})

``````

![](https://images2018.cnblogs.com/blog/1356841/201809/1356841-20180911215522692-421655458.png)