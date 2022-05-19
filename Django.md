# Django

Django的**5项基础核心技术**包括模型（Model）的设计，URL的配置，View（视图）的编写，Template（模板）的设计和Form(表单）的使用



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