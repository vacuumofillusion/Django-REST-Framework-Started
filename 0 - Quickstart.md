# 快速入门

我们将创建一个简单的API，允许管理员用户查看和编辑系统中的用户和组。

## 项目设置

创建一个名为`tutorial`的新Django项目，然后启动一个名为`quickstart`的新应用。

```bash
# 创建项目目录
mkdir tutorial
cd tutorial

# 创建一个虚拟环境以在本地隔离我们的包依赖
python3 -m venv env
source env/bin/activate  # 在Windows上使用 `env\Scripts\activate`

# 在虚拟环境中安装Django和Django REST框架
pip install djangorestframework

# 设置一个包含单个应用的新项目
django-admin startproject tutorial .  # 注意末尾的'.'字符
cd tutorial
django-admin startapp quickstart
cd ..
```

项目布局应该如下所示：

    $ pwd
    <some path>/tutorial
    $ find .
    .
    ./tutorial
    ./tutorial/asgi.py
    ./tutorial/__init__.py
    ./tutorial/quickstart
    ./tutorial/quickstart/migrations
    ./tutorial/quickstart/migrations/__init__.py
    ./tutorial/quickstart/models.py
    ./tutorial/quickstart/__init__.py
    ./tutorial/quickstart/apps.py
    ./tutorial/quickstart/admin.py
    ./tutorial/quickstart/tests.py
    ./tutorial/quickstart/views.py
    ./tutorial/settings.py
    ./tutorial/urls.py
    ./tutorial/wsgi.py
    ./env
    ./env/...
    ./manage.py

在项目目录中创建应用可能看起来不太寻常。使用项目的命名空间可以避免与外部模块发生名称冲突（这是一个超出快速入门范围的主题）。

现在，首次同步你的数据库：

```bash
python manage.py migrate
```

我们还将创建一个名为`admin`的初始用户并设置密码。稍后在示例中，我们将以该用户身份进行身份验证。

```bash
python manage.py createsuperuser --username admin --email admin@example.com
```

一旦你设置了数据库并且初始用户已经创建并准备就绪，打开应用的目录，我们开始编写代码...

## 序列化器

首先，我们将定义一些序列化器。让我们创建一个名为`tutorial/quickstart/serializers.py`的新模块，我们将用它来表示数据。

```python
from django.contrib.auth.models import Group, User
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```

请注意，在这种情况下，我们使用了带有`HyperlinkedModelSerializer`的超链接关系。你也可以使用主键和各种其他关系，但超链接是良好的RESTful设计。

## 视图

好的，接下来我们需要编写一些视图。打开`tutorial/quickstart/views.py`并开始编写代码。

```python
from django.contrib.auth.models import Group, User
from rest_framework import permissions, viewsets

from tutorial.quickstart.serializers import GroupSerializer, UserSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all().order_by('name')
    serializer_class = GroupSerializer
    permission_classes = [permissions.IsAuthenticated]
```

我们不是编写多个视图，而是将所有通用行为分组到称为`ViewSets`的类中。

如果我们需要，可以很容易地将它们分解为单独的视图，但使用视图集可以很好地组织视图逻辑，并且非常简洁。

## URLs

好的，现在让我们连接API的URLs。转到`tutorial/urls.py`...

```python
from django.urls import include, path
from rest_framework import routers

from tutorial.quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

因为我们使用了视图集而不是视图，我们可以通过简单地使用路由器类注册视图集来自动生成API的URL配置。

再次强调，如果我们需要更多对API URLs的控制，我们只需使用常规的基于类的视图，并明确编写URL配置。

最后，我们包含了默认的登录和注销视图，以与可浏览的API一起使用。这是可选的，但如果你的API需要身份验证并且你想使用可浏览的API，这将很有用。

## 分页

分页允许你控制每页返回的对象数量。要启用分页，请在`tutorial/settings.py`中添加以下行：

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

## 设置

将`'rest_framework'`添加到`INSTALLED_APPS`中。设置模块位于`tutorial/settings.py`

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

好了，我们完成了。

---

## 测试我们的API

现在，我们准备测试已经构建的API。让我们从命令行启动服务器。

```bash
python manage.py runserver
```

我们现在可以通过命令行使用`curl`等工具来访问我们的API...

```bash
curl -u admin -H 'Accept: application/json; indent=4' http://127.0.0.1:8000/users/
Enter host password for user 'admin':
{
    "count": 1,
    "next": null,
    "previous": null,
    "results": [
        {
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin",
            "email": "admin@example.com",
            "groups": []
        }
    ]
}
```

或者使用[httpie](https://httpie.io/docs#installation)命令行工具...

```bash
http -a admin http://127.0.0.1:8000/users/
http: password for admin@127.0.0.1:8000:: 
HTTP/1.1 200 OK
...
{
    "count": 1,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin"
        }
    ]
}
```

或者，你可以直接在浏览器中通过访问URL `http://127.0.0.1:8000/users/`来测试...

![快速入门图片](https://www.django-rest-framework.org/img/quickstart.png)

如果你通过浏览器工作，请确保使用右上角的控件登录。

太棒了，这很容易！

如果你想要更深入地了解REST框架是如何组合的，请前往[教程](https://www.django-rest-framework.org/tutorial/1-serialization/)，或者开始浏览[API指南](https://www.django-rest-framework.org/api-guide/requests/)。

[image]: ../img/quickstart.png
[tutorial]: 1-serialization.md
[guide]: ../api-guide/requests.md
[httpie]: https://httpie.io/docs#installation