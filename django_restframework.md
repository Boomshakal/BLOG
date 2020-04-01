# django restframework

## APIView

1. 用户请求到django,首先经过wsgi,中间件,然后到url路由系统,执行视图类中继承APIView执行as_view方法

2. 在源码中可以看到VPIView继承了django的View类,通过super执行View中的as_view方法[详细看文章](http://www.cnblogs.com/supery007/p/8432324.html),最终返回执行self.dispatch()

3. 按照django类中查找顺序现从自己的方法中找,如果自己没有dispatch方法再从继承的父类中找,从APIView中找dispatch方法

4. 在dispatch中首先将request执行self.initialze_request重新封装request,
   之后执行self.initial方法,这个方法一共执行4步操作:
> 首先对request版本进行验证
```
版本控制执行自己类中的determine_version方法,最终是versioning_class的属性,可以通过在settings.py文件中进行配置
```
>  第二步进行用户认证

```
执行perform_authentication方法,在这个方法中执行了request.user方法,按照python类查找顺序先到APIView中进行查找,没有网View中查找都没有,往回看重新看一下之前的request封装操作,

查到restframework.request中有个Request类重新封装request,在Request中查找user方法,最后找到request.user最后返回authentication_classes实例化并返回认证列表
```

> 第三步进行权限控制

```
  执行check_permissions方法,按照上面的执行的流程,最后返回permission_classes实例化并返回权限列表,然后循环实例化对象中has_permission(必须存在)方法进行判断,如果定义了权限类,
  
  has_permission必须有返回值,可以返回布尔值或raise一个报错信息,True表示有权限不做操作,False没有权限执行permission_denied方法,首先进行判断是否进行认证,如果没有认证则raise一个没有认证错误信息,
  
  如果有认证则raise一个没有权限错误信息。
```
> 第四步进行用户访问频率限制
```
执行check_throttles方法,按照上面的执行的流程,最后返回throttle_classes实例化并节流列表,循环执行allow_request方法,源码中如果没有定义allow_request方法则restframework会返回raise错误必须重写allow_request方法,

重写allow_request,比如对匿名用户访问做限制1分钟只能访问10次超过10次休息1分钟返回false不让访问,执行到wait(必须定义)进行重写,将访问时间进行计算然后返回下次访问时间,

也可以继承restframework已经写好的类SimpleRateThrottle,AnonRateThrottle,UserRateThrottle,ScopedRateThrottle
```

## 解析器组件

```python
class LoginView(APIView):
		def get(self,request):
			pass
        def post(self,request):
        	request.data		#新的request对象 @property data()
        	return 
			
from rest_framework.request import Request
class APIView(View):
		def as_view(cls,**initkwargs):
			pass
			super(APIView,cls).as_view(**initkwargs)
            def initialize_request(self, request, *args, **kwargs):
        """
        Returns the initial request object.
        """
        parser_context = self.get_parser_context(request)

        return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )
        def dispatch(self):
        	pass
        	request = self.initialize_request(request, *args, **kwargs)
        	self.request = request
```

1. views.LoginView.as_view()
2. LoginView里面没有as_view方法，到父类APIView去找
3. 执行View里面的as_view()方法，返回view函数
```python
        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.setup(request, *args, **kwargs)
            if not hasattr(self, 'request'):
                raise AttributeError(
                    "%s instance has no 'request' attribute. Did you override "
                    "setup() and forget to call super()?" % cls.__name__
                )
            return self.dispatch(request, *args, **kwargs)
```
4. url和视图函数之间的绑定关系建立完毕{"login": view},等待用户请求

5. 接收到用户请求：login，到建立好的绑定关系里面执行对应的试图函数：view(request)

6. 视图函数的执行结果是什么就返回给用户什么：self.dispatch(),self.dispatch()的执行结果是什么，就返回给用户什么

7. 此时的self代表的是LoginView的实例化对象

8. 开始找dispatch方法，self里面没有，Login View里面也没有，在APIView里面有

9. 开始执行APIView里面的dispatch

10. 最后找到http方法（GET,POST,PUT,DELETE等），根据请求类型查找request.method.lower()

11. 开始执行找到的方法（GET），self.get() ,self此时代表LoginView的实例化对象
	11.1 假设接收到的是POST请求，执行request.data 
	11.2 根据分析，所有的解析工作都在request.data里面实现，且data是一个方法 被装饰的（@property）
	
	```python
    @property
    def data(self):
        if not _hasattr(self, '_full_data'):
            self._load_data_and_files()
	    return self._full_data
	```
	11.3 开始执行request.data，执行self._load_data_and_files
	
	11.4 执行`self._data`, `self._files` = self._parse()
	
	11.5 执行parser = self.negotiator.select_parser(self, self.parsers)
	
	11.6  self.parsers就是引用的parse_classes = api_settings.DEFAULT_PARSER_CLASSES (未找到变量执行`__getattr__`方法)
	11.7 动态导入模块val = perform_import(val, attr)
	
	11.8 解析数据parsed = parser.parse(stream, media_type, self.parser_context)
	
	11.9 return (parsed.data, parsed.files)就是我们想要的数据
	
	11.10 DRF将`self._full_data` = `self._data`
	
	11.11  request.date
	
12. 在LoginView里面找到了对应的方法，执行改方法，最后返回给用户

- **DRF的所有功能都是在as_view()和dispatch里面重写的**
- **而解析器组件在dispatch方法里面重写的**

## 序列化组件

<table>
    <tr>
        <th>主要内容</th>
        <th>小标题</th>
        <th>serializers</th>
        <th>ModelSerializer</th>
    </tr>
    <tr>
        <th rowspan='3'>field</th>
        <th>常用的field</th>
        <th colspan='2'>CharField、BooleanField、IntegerField、DateTimeField</th>
    </tr>
    <tr>
        <th>Core arguments</th>
        <th colspan='2'>read_only、write_only、required、allow_null/allow_blank、lable、help_text、style</th>
    </tr>
    <tr>
        <th>HiddenField</th>
        <th colspan='2'>HiddenField的值不需要用户自己post数据过来，也不会显示返回给用户。配合CurrentUserDefault()可以实现获取到请求用户</th>
    </tr>
    <tr>
        <th rowspan='3'>save instance</th>
        <th rowspan='3'></th>
        <th colspan='2'>serializer.save()，它会调用了serializer的create或update方法</th>
    </tr>
    <tr>
        <th>当有post请求，需要重写serializer.create方法</th>
        <th colspan='2'>ModelSerializer已经封装了这两个方法，如果额外自定义，也可以进行重载</th>
    </tr>
    <tr>
        <th>当有patch请求，需要重写serializer.update方法</th>
    </tr>
    <tr>
        <th rowspan='3'>Validation自定义验证逻辑</th>
        <th>单独的validate</th>
        <th colspan='2'>对某个字段进行自定义验证逻辑，重载valizte_+字段名</th>
    </tr>
    <tr>
        <th>联合validate</th>
        <th colspan='2'>重写validate()方法，对多个字段进行验证或者对read_only的字段进行操作</th>
    </tr>
    <tr>
        <th>Validators</th>
        <th colspan='2'>1.单独作用于某个field，作用于重载validate_+字段名相似。
            2.UniqueValidator:指定某一个对象是唯一的，直接作用于某个field。
            3.UniqueTogetherValidator:联合唯一，需要咋Meta属性中设置</th>
    </tr>
    <tr>
        <th rowspan='2'>ModelSerializer专属</th>
        <th>validate</th>
        <th></th>
        <th>重载validate可以实现删除用户提交的字段，单改字段不存在指定model,避免save()出错</th>
    </tr>
    <tr>
        <th>SerializerMethodField</th>
        <th></th>
        <th>SerializerMethodField实现将model不存在字段或者获取不到的数据序列化返回给用户</th>
    </tr>
    <tr>
        <th rowspan='3'>外键的serializers</th>
        <th rowspan='2'>正向</th>
        <th>PrimaryKeyRelatedField,不关心外键具体内容</th>
        <th>通过field已经映射，不关心外键具体内容</th>
    </tr>
    <tr>
        <th colspan='2'>嵌套serializer，获取外键具体内容</th>
    </tr>
    <tr>
        <th>反向</th>
        <th colspan='2'>Model设置related_name,通过related_name嵌套serializer</th>
    </tr>
</table>



1. Django原生serializers  django.core.serializers
```python
from django.core.serializers import serialize
class BookView(APIView):
	serialized_data = serialize('json',Course.objects.all())
    return HttpResponse(serialized_data)
```
2. rest_framework  serialzers 
- 需要手动插入数据（必须自定义create）
- 手动序列化需要的字段

```python
# serializers.py
form rest_framework import serializers
class BookSerializer(serializers.Serializer):
    publish_name = serializers.CharField(read_only=True, source='publish.name')
    authors_list = serializers.SerializerMethodField()
    
    
    def get_authors_list(self,book_obj):
        authors = list()
        for author in book_obj.authors.all()
        	authors.append(author.name)
            
            return authors
    
    def create(slef,validated_data):
        validated_data['publish_id'] = validated_data.pop('publish')
        book = Book.objects.create(**validated_data)
        return book
    
        
# view.py 
class BookView(APIView):
    def get(self,request):
        book_obj = Book.objects.all()

        serialized_data = BookSerializer(book_obj,many=True)

        return Response(serialized_data.data)

    def post(self,request):
        validated_data = BookSerializer(data=request.data)

        if validated_data.is_valid():
            book = validated_data.save()
            authors = Author.objects.filter(nid_in=request.data['authors'])
            book.authors.add(*authors)
            return Response(validated_data.data)
        else:
            return Response(validated_data.errors)
```
3.  为了解决上面的问题，我们建议使用serializers.ModelSerializer

```python
# serializers.py
class BookSerializer(serializers.ModelSerializer):
    # 外键字段显示__str__方法的返回值
    publish_name = serializers.CharField(read_only=True, source='publish.name')
    authors_list = serializers.SerializerMethodField()
    
    # 多对多字段需要自己手动获取数据
    def get_authors_list(self,book_obj):
        authors =list()
        for author in bokk_obj.authors.all():
        	authors.append(author.name)
        return authors
    class Meta:
        model = Book
        fields = {'field_name',} #'__all__'
        extra_kwargs = {'publish':{"write_only":True},}  
# view.py
class BookFilterView(APIView):
    def get(self,request,nid):
        book_obj = Book.objects.get(pk=nid)
        
        serialized_data = BookSerializer(book_obj)
        
        return Response(serialized_data.data)
    
    def delete(self,request,nid):
        
        Book.objects.get(pk=nid).delete()
        
        return Response()
    
    def put(self,request,nid):
        book_obj = Book.objects.get(pk=nid)
        
        verified_data = BookSerializer(data=request.data, instance=book_obj,many=False)
        
        if verified_data.is_valid():
            verified_data.save()
            return Response(verified_data.data)
        else:
            return Response(verified_data.errors)
```

## 视图组件

1. 导入**mixin**： form rest_framework.mixins

- ListModelMixin
- CreateModelMixin
- RetrieveModelMixin
- UpdateModelMixin
- DestroyModelMixin

2. mixin源码剖析
- Django程序启动，开始初始化，读取url.py settings,读取视图类
- 执行as_view() BookView没有，需要到父类中找
- 几个ModelMixin也没有，GenericAPIView没有，GenericAPIView(APIView)中找
- 找到了，并且与之前的逻辑是一样的，同时我们发现GenericAPIView中定义了查找queryset和serialzer_class类的方法
- as_view()方法返回重新封装的视图函数，开始建立url和视图函数之间的映射关系
- 等待用户请求
- 接收到用户请求，根据url找到视图函数
- 执行视图函数的dispatch方法(因为视图函数的返回值是return self.dispatch())
- dispatch分发请i去，茶轴打哦试图类的五个方法中的其中一个
- 开始执行比如post请求，返回self.create()，视图类本身没有，则会去父类中查找
- 最后在CreateModelMixin中找到
- 执行create()方法，获取qureyset和serializer_class
- 返回数据
3. 导入**GenericAPIView**: form rest_framework.generics

```python
class BookView(ListModelMixin,CreateModelMixin,GenericAPIView):
       queryset = Book.objects.all()
       serializer_class = BookSerializer
       def get(self, request, *args, **kwargs):
               return self.list(self, request, *args, **kwargs)
       def post(self, request, *args, **kwargs):
               return self.create()
   class BookFilterView(RetrieveModelMixin,DestroyModelMixin,UpdateModelMixin,GenericAPIView):
       queryset = Book.objects.all()
       serializer_class = BookSerializer
       def get(self, request, *args, **kwargs):
           return self.retrieve(self, request, *args, **kwargs)
       def put(self, request, *args, **kwargs):
           return update(self, request, *args, **kwargs)
       def delete(self, request, *args, **kwargs):
           return self.destroy(self, request, *args, **kwargs)
```
4. 封装合并view  from rest_framework import generics
```python
class BookView(generics.ListCreateAPIView):
		queryset = Book.objects.all()
		serializer_class = BookSerializer
class BookFilterView(generics.RetrieveUpdateDestroyAPIView):
		queryset = Book.objects.all()
		serializer_class = BookSerializer
```
5. **ViewSet**    from rest_framework.ViewSets import ModelViewSet
```python
# url.py
urlpatterns = [
    url(r'books/$', view.BooksViewSet.as_view({
    	'get':'list',
    	'post':'create'
    })),
    url(r'books/(?P<pk>\d+)/$', view.BooksViewSet.as_view({
    	'get':'retrieve',
    	'put':'update',
    	'delete':'destroy'
    })), 
]
# view.py
class BooksViewSet(ModelViewSet):
	queryset = Book.objects.all()
	serializer_class = BookSerializer
```
4. viewset源码剖析
- Django程序启动，开始初始化，读取urls.py settings、读取视图类
- 执行as_view()BookViewSet没有，需要到父类(ModelViewSet)中找
- ModelViewSet继承了mixins的几个ModelMixin和GenericViewSet,显然ModelMixin也没有，只有GenericViewSet中有
- GenericViewSet没有任何代码，只继承了ViewSetMixin和generics GenericAPIView
- 继续去ViewSetMixin中查找，找到as_view类方法，在重新封装view函数的过程中，有一个self.action_map = actions
- 这个actions就是我们给as_view()传递的参数
- 绑定url和属兔函数actions之间的映射关系
```python
self.action_map = actions
for method, action in actions.items():
	handler = getattr(self, action)
	setattr(self, method, handler)
    # setattr(self,'get',self.list)
    # setattr(self,'post',self.create)
    # setattr(self,'get',self.retrieve)
    # setattr(self,'put',self.update)
    # setattr(self,'delete',self.destroy)
```
- 等待用户请求
- 接收到用户请求，根据url找到视图函数
- 执行视图函数的dispatch()方法（因为视图函数的返回值是：return self.dispatch()）
- dispatch分发请求，找到视图类的五个方法中的其中一个
- 开始执行，比如post请求，返回self.create(),视图类本身没有，则回到父类中查找
- 最后在CreateModelMixin中查找到
- 执行create()方法，获取queryset和serializer_class
- 返回数据

5. 视图继承图
 ```mermaid
    graph TD
        
        ModelViewSet-->CreateModelMixin
        ModelViewSet-->RetrieveModelMixin
        ModelViewSet-->UpdatelMixin
        ModelViewSet-->DestroyModelMixin
        ModelViewSet-->ListModelMixin
        ReadOnlyModelViewSet-->ListModelMixin
        ReadOnlyModelViewSet-->GenericViewSet
        ReadOnlyModelViewSet-->RetrieveModelMixin
        GenericViewSet-->GenericAPIView-->APIView-->View
        GenericViewSet-->ViewSetMixin
        ViewSet-->APIView
        ViewSet-->ViewSetMixin
 ```

## 认证组件

```python
# 定义认证类
class UserAuth(BaseAuthentication):
    #def authenticate_header(self, request):
    #    pass
    def authenticate(self, request):
        user_token = request.query_params.get('token')
        try:
            token = UserToken.objects.get(token=user_token)
            return token.user.username, token.token
        except Exception:
            raise APIException('没有认证')
# 在需要认证的数据接口指定认证类      
authentication_classes = [UserAuth]
```
源码解剖分析
多个认证类请在最后一个认证类返回值

1. self.dispatch()  #self BookView 的实例化对象
2. self.initial()   #self BookView 的实例化对象
3. 		self.perform_authentication() #self BookView 的实例化对象
4. 			request.user			  # self Request的实例化对象
5. 				self._authentiacte()
6. 				self.user, self.auth = user_auth_tuple

全局认证

- settings文件里面指定

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES":(
    "utils.app_authes.UserAuth"
    )
}
```
DRF 认证类
```
from rest_framework.authentication import TokenAuthentication
```

## 权限组件

1. 权限组件的使用

```python
# 定义权限类
class UserPerm():
    def has_permission(self,request,view):
        if has:
            return True
        else:
            return False
# 指定权限类
class BookView(ModelViewSet):
    permission_classes = [UserPerm]
```

DRF的权限类

```
from rest_framework.permissions import IsAuthenticated
```

## 频率组件

```python
# 定义频率类
class RateThrottle():
    def allow_request(self,request):
        if allow:
            return True
        else:
            return False
class BookView(ModelViewSet):
    throttle_classes = [RateThrottle]
```
全局配置
```python
class RateThrottle(SimpleRateThrottle):
    scope = 'visit_rate'
    
    def get_cache_key(self,request,view):
        return self.get_ident(request)		#获取IP地址
    
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES":('utils.app_throttles.RateThrottle',),
    "DEFAULT_THROTTLE_RATES":{
        "visit_rate":"15/m"
    }
}
```

DRF的频率类

```
from rest_framework.throttling import SimpleRateThrottle
```

## url 注册器组件

```python
# 导入模块 
import rest_framework import routers
# 生成一个注册器实列对象
router = routers.DefaultRouter()
# 开始生成url
router.register(r'books', views.BookView, base_name='Book')
urlpatterns = [···]
urlpatterns += router.urls  
```

## 响应器组件

```python
from rest_framework.renderers import JsonRenderer,BrowsableAPIRenderer

renderer_classes = [JsonRenderer,BrowsableAPIRenderer]
```

## 分页器组件

```python
# 导入模块
from rest_framework.pagination import PageNumberPagination

class BookView(APIView):
    def get(self,request):
        # 获取数据
        books = Book.objects.all()
        # 创建分页器
        paginater = PageNumberPagination()
        # 开始分页
        paged_books = paginater.paginate_queryset(books,request)
        # 序列化
        serialized_books = BookSerializer(paged_books, many=True)
        # 返回数据
        return Response(serialized_books.data)
```

```python
class BooksPagination(PageNumberPagination):
    page_size = 12
    page_size_query_param = 'page_size'
    page_query_param = 'page'
    max_page_size = 20
    
    
class BookView(ModelViewSet):
    pagination_class = GoodsPagination 
```

## Django ContentType
- 在model中定义ForeignKey字段，并关联到ContentType表。通常这个字段命名为“content_type”

- 在model中定义PositiveIntegerField字段，用来存储关联表中的主键。通常这个字段命名为“object_id”

- 在model中定义GenericForeignKey字段，传入上述两个字段的名字。

.model_class()  表示ContentType表model字段 的类
注意：ContentType只运用于1对多的关系！！！并且多的那张表中有多个ForeignKey字段。

```python
class Electrics(models.Model):
    name = models.CharField(max_length=32)
    price = models.IntegerField(default=100)
    coupons = GenericRelation(to='Coupon')  # 用于反向查询，不会生成表字段

    def __str__(self):
        return self.name
    
class Coupon(models.Model):
    """
    Coupon
    id    name                       content_type_id       object_id_id
    1     美的满减优惠券				9（电器表electrics）    3
    2     猪蹄买一送一优惠券           10                     2
    3     南极被子买200减50优惠券      11                     1
    """
    name = models.CharField(max_length=32)

    content_type = models.ForeignKey(to=ContentType) # step 1
    object_id = models.PositiveIntegerField() # step 2
    content_object = GenericForeignKey('content_type', 'object_id') # step 3

    def __str__(self):
        return self.name
```

## Django Forms 和modelForm

```python
model.py
class Book(models.Model):
    title=models.CharField(max_length=32)
    price=models.DecimalField(max_digits=8,decimal_places=2)
    pub_date=models.DateField()
    publish=models.ForeignKey("Publish")
    authors=models.ManyToManyField("Author")
    def __str__(self): 
        return self.title

forms.py
#Modelform将一个model转化成一个form组件
class BookModelForm(forms.ModelForm):
     class Meta:
        model=models.Book
        fields="__all__"
这一步做的事情相当于下面的代码
'''
class BookModelForm(form.Form):
     title=forms.CharField(max_length=32)
     price=forms.IntegerField()
     pub_date=forms.DateField()

'''
```

## stark 组件

