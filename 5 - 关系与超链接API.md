# 教程 5: 关系和超链接API

目前，我们的API中的关系是通过使用主键来表示的。在本教程的这一部分中，我们将通过使用超链接来表示关系，从而改进API的连贯性和可发现性。

## 创建API根目录的端点

目前我们有“代码片段”和“用户”的端点，但我们没有API的单一入口点。为了创建一个入口点，我们将使用常规的函数式视图和我们之前介绍的`@api_view`装饰器。在您的`snippets/views.py`中添加：

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

这里有两点需要注意。首先，我们正在使用REST框架的`reverse`函数来返回完整的URL；其次，URL模式是通过在后续的`snippets/urls.py`中声明的方便名称来识别的。

## 创建高亮代码片段的端点

我们的pastebin API中另一个明显缺失的是代码高亮显示的端点。

与我们所有的其他API端点不同，我们不希望使用JSON，而是只呈现HTML表示。REST框架提供了两种HTML渲染器，一种用于处理使用模板渲染的HTML，另一种用于处理预渲染的HTML。对于这个端点，我们想使用第二种渲染器。

在创建代码高亮视图时，我们需要考虑的另一件事是，没有现成的具体泛型视图可以使用。我们不是返回一个对象实例，而是返回一个对象实例的属性。

因此，我们不会使用具体的泛型视图，而是使用表示实例的基类，并创建我们自己的`.get()`方法。在您的`snippets/views.py`中添加：

```python
from rest_framework import renderers

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = [renderers.StaticHTMLRenderer]

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

像往常一样，我们需要将我们创建的新视图添加到我们的URL配置中。
我们将在`snippets/urls.py`中添加一个新API根目录的URL模式：

```python
path('', views.api_root),
```

然后为代码片段高亮显示添加一个URL模式：

```python
path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view()),
```

## 为我们的API添加超链接

处理实体之间的关系是Web API设计中更具挑战性的方面之一。我们可以选择多种不同的方式来表示关系：

* 使用主键。
* 在实体之间使用超链接。
* 在相关实体上使用唯一的标识符slug字段。
* 使用相关实体的默认字符串表示。
* 将相关实体嵌套在父表示中。
* 其他一些自定义表示。

REST框架支持所有这些样式，并且可以在正向或反向关系、或跨自定义管理器（如通用外键）中应用它们。

在这种情况下，我们希望在实体之间使用超链接样式。为此，我们将修改序列化器以扩展`HyperlinkedModelSerializer`而不是现有的`ModelSerializer`。

`HyperlinkedModelSerializer`与`ModelSerializer`有以下区别：

* 它默认情况下不包括`id`字段。
* 它包含一个`url`字段，使用`HyperlinkedIdentityField`。
* 关系使用`HyperlinkedRelatedField`，而不是`PrimaryKeyRelatedField`。

我们可以轻松地重写现有的序列化器以使用超链接。在您的`snippets/serializers.py`中添加：

```python
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ['url', 'id', 'highlight', 'owner',
                    'title', 'code', 'linenos', 'language', 'style']


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ['url', 'id', 'username', 'snippets']
```

注意，我们还添加了一个新的`'highlight'`字段。这个字段与`url`字段的类型相同，只是它指向的是`'snippet-highlight'` URL模式，而不是`'snippet-detail'` URL模式。

因为我们包含了带有格式后缀的URL，如`'.json'`，所以我们还需要在`highlight`字段上指示，它返回的任何带有格式后缀的超链接都应该使用`'.html'`后缀。

## 确保我们的URL模式被命名

如果我们打算拥有一个超链接的API，我们需要确保我们的URL模式被命名。让我们来看看哪些URL模式需要被命名。

* 我们的API的根目录指向`'user-list'`和`'snippet-list'`。
* 我们的代码片段序列化器包含一个字段，该字段指向`'snippet-highlight'`。
* 我们的用户序列化器包含一个字段，该字段指向`'snippet-detail'`。
* 我们的代码片段和用户序列化器包含`'url'`字段，该字段默认情况下将指向`'{model_name}-detail'`，在本例中将是`'snippet-detail'`和`'user-detail'`。

在将这些名称添加到我们的URL配置之后，我们最终的`snippets/urls.py`文件应该看起来像这样：

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    path('', views.api_root),
    path('snippets/',
        views.SnippetList.as_view(),
        name='snippet-list'),
    path('snippets/<int:pk>/',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    path('snippets/<int:pk>/highlight/',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    path('users/',
        views.UserList.as_view(),
        name='user-list'),
    path('users/<int:pk>/',
        views.UserDetail.as_view(),
        name='user-detail')
])
```

## 添加分页

用户和代码片段的列表视图最终可能会返回相当多的实例，因此我们真的需要确保对结果进行分页，并允许API客户端逐个遍历每一页。

我们可以通过稍微修改`tutorial/settings.py`文件来更改默认的列表样式以使用分页。添加以下设置：

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

请注意，REST框架中的设置都位于一个名为`REST_FRAMEWORK`的单个字典设置中，这有助于将它们与您的其他项目设置很好地分隔开。

如果我们需要，还可以自定义分页样式，但在这个例子中，我们将坚持使用默认值。

## 浏览API

如果我们打开浏览器并导航到可浏览的API，您会发现现在只需跟随链接即可在API中自由浏览。

您还将在代码片段实例上看到“highlight”链接，这些链接将带您进入突出显示的代码HTML表示。

在教程的[第6部分](https://www.django-rest-framework.org/tutorial/6-viewsets-and-routers/)中，我们将了解如何使用ViewSets和Routers来减少构建API所需的代码量。