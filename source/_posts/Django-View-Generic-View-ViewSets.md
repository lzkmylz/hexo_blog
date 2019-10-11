---
title: Django Rest Framework View/Generic View/ViewSets
date: 2019-06-30 14:31:42
tags: ['Django']
---

目前，在主流的前后端分离式协作模式下使用Python的Django作为后端时，通常使用Django Rest Framework来写Restful API以供前端调用。而在DRF中创建Restful API主要用的就是View/Generic View以及ViewSets这三种方式。

# View

用View来创建Restful API，一般需要创建一个继承了APIView的class，并override其`get()`和`post()`、`list()`、`update()`等方法来实现，或者是通过一个@APIView的function来实现。

```
from rest_framework.views import APIView

class ListUsers(APIView):
  authentication_classes = (authentication.TokenAuthentication,)
  permission_classes = (permissions.IsAdminUser,)

  def get(self, request, format=None):
    usernames = [user.username for user in User.objects.all()]
    return Response(usernames)
```

同时，View暴露了以下接口属性，可以通过覆盖这些属性来将如认证、权限管理等注入View中：

* renderer_classes
* parser_classes
* authentication_classes
* throttle_classes
* permission_classes
* content_negotiation_class
* metadata_class
* versioning_class

其中比较常用的是`authentication_classes`、`permission_classes`这两个。另外需要注意的是，基于View实现的API只能对应一个url的一系列对应不同HTTP Header Method。

# Generic View

每次都根据不同的Model重写View很容易产生重复代码。而Django提供了Mixin方式来扩展View————Generic View。Generic View继承了API View，并在其上扩展来有关queryset、paginate、serializer等。而后通过Mixin扩增得到自带关于不同HTTP Header Method的API。

* CreateAPIView = GenericAPIView + CreateModelMixin
* ListAPIView = GenericAPIView + ListModelMixin
* RetrieveAPIView = GenericAPIView + RetrieveModelMixin
* DestroyAPIView = GenericAPIView + DestroyModelMixin
* UpdateAPIView = GenericAPIView + UpdateModelMixin
* ListCreateAPIView = GenericAPIView + ListModelMixin + CreateModelMixin
* RetrieveUpdateAPIView = GenericAPIView + RetrieveModelMixin + UpdateModelMixin
* RetrieveDestroyAPIView = GenericAPIView + RetrieveModelMixin + DestroyModelMixin
* RetrieveUpdateDestroyAPIView = GenericAPIView + RetrieveModelMixin + UpdateModelMixin + DestroyModelMixin

# ViewSet

ViewSet可以将一系列API放在一个perfix下，从而将有关API更好的集中，并减少来URL设定。所有的ViewSet都是继承了ViewSetMixin和APIView扩增出来的。

* ViewSet = ViewSetMixin + APIView
* GenericViewSet = ViewSetMixin + GenericAPIView
* ReadOnlyModelViewSet = GenericViewSet + RetrieveModelMixin + ListModelMixin
* ModelViewSet = GenericViewSet + CreateModelMixin + RetrieveModelMixin + UpdateModelMixin + DestroyModelMixin + ListModelMixin

因此最佳的方式就是根据需要，继承GenericViewSet和相关的Mixin来自动构建简单API。在有特殊需求时，通过@action来构建对应的API/Detail API，此时API的调用形式为`/{prefix}/method_name/`。