# 教程 6: ViewSets 与 Routers

REST framework 包含了处理 `ViewSets` 的抽象，允许开发者专注于建模 API 的状态和交互，而将 URL 的构建留给基于通用约定的自动处理。

`ViewSet` 类几乎与 `View` 类相同，不同之处在于它们提供如 `retrieve` 或 `update` 这样的操作，而不是如 `get` 或 `put` 这样的方法处理器。

`ViewSet` 类仅在最后一刻被绑定到一组方法处理器上，当它被实例化为一组视图时，这通常是通过使用 `Router` 类来完成的，该类会为您处理定义 URL 配置的复杂性。

## 重构以使用 ViewSets

让我们将当前的视图集重构为视图集。

首先，让我们将 `UserList` 和 `UserDetail` 类重构为一个单一的 `UserViewSet` 类。在 `snippets/views.py` 文件中，我们可以删除这两个视图类，并用一个单一的 ViewSet 类替换它们：

```python
from rest_framework import viewsets


class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    This viewset automatically provides `list` and `retrieve` actions.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

在这里，我们使用了 `ReadOnlyModelViewSet` 类来自动提供默认的“只读”操作。我们仍然像使用常规视图时一样设置 `queryset` 和 `serializer_class` 属性，但我们不再需要将相同的信息提供给两个单独的类。

接下来，我们将替换 `SnippetList`、`SnippetDetail` 和 `SnippetHighlight` 视图类。我们可以删除这三个视图，并再次用一个类替换它们。

```python
from rest_framework import permissions
from rest_framework import renderers
from rest_framework.decorators import action
from rest_framework.response import Response


class SnippetViewSet(viewsets.ModelViewSet):
    """
    This ViewSet automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                            IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

这次我们使用了 `ModelViewSet` 类来获取完整的默认读写操作集。

请注意，我们还使用了 `@action` 装饰器来创建一个名为 `highlight` 的自定义操作。此装饰器可用于添加任何不符合标准 `create`/`update`/`delete` 风格的自定义端点。

使用 `@action` 装饰器的自定义操作将默认响应 `GET` 请求。如果我们希望某个操作响应 `POST` 请求，则可以使用 `methods` 参数。

自定义操作的 URL 默认取决于方法名本身。如果您想更改 URL 的构建方式，可以在装饰器中包含 `url_path` 作为关键字参数。

## 明确将 ViewSets 绑定到 URL

处理方法仅在定义 URLConf 时才绑定到操作上。
为了了解背后的工作原理，让我们首先从 ViewSets 中显式创建一组视图。

在 `snippets/urls.py` 文件中，我们将 `ViewSet` 类绑定到一组具体的视图中。

```python
from rest_framework import renderers

from snippets.views import api_root, SnippetViewSet, UserViewSet

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```

请注意，我们是如何将每个 `ViewSet` 类中的 HTTP 方法绑定到每个视图所需的操作上来创建多个视图的。

现在我们已经将资源绑定到具体的视图上，我们可以像往常一样将这些视图注册到 URL 配置中。

```python
urlpatterns = format_suffix_patterns([
    path('', api_root),
    path('snippets/', snippet_list, name='snippet-list'),
    path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail')
])
```

## 使用 Routers

因为我们使用的是 `ViewSet` 类而不是 `View` 类，所以我们实际上不需要自己设计 URL 配置。使用 `Router` 类可以自动处理将资源连接到视图和 URL 的约定。我们只需要将适当的视图集注册到路由器上，然后让它完成其余的工作。

以下是重新连接后的 `snippets/urls.py` 文件。

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from snippets import views

# Create a router and register our ViewSets with it.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet, basename='snippet')
router.register(r'users', views.UserViewSet, basename='user')

# The API URLs are now determined automatically by the router.
urlpatterns = [
    path('', include(router.urls)),
]
```

将 ViewSet 注册到路由器与提供 url 模式类似。我们需要包含两个参数 - 视图的 URL 前缀和视图集本身。

我们使用的 `DefaultRouter` 类还会自动为我们创建 API 根视图，因此我们现在可以从 `views` 模块中删除 `api_root` 函数。

## 视图与 ViewSet 之间的权衡

使用 ViewSet 可以是一个非常有用的抽象。它有助于确保您的 API 中的 URL 约定保持一致，减少您需要编写的代码量，并允许您专注于 API 提供的交互和表示，而不是 URL 配置的具体细节。

但这并不意味着它总是正确的做法。与使用基于类的视图而不是基于函数的视图时需要考虑的权衡类似，使用 ViewSet 相比单独构建 API 视图来说，不那么直观。