---
title: Django中contenttypes组件的使用
date: 2019-09-22 17:21:23
categories: 
- Django
tags: 
- Django
- contenttypes组件
toc: true
---
contenttypes是Django内置的一个非常有用的组件，它对当前项目中所有基于Django驱动的model提供了更高层次的接口，简单点说，它可以追踪项目中所有的应用与model的对应关系并将这些关系记录在ContentType表中，每当我们创建了新的model并执行数据库迁移指令后，ContentType表中就会自动新增一条不同应用和这个应用中的model之间对应关系的记录；更重要的是，我们不仅可以使用contenttypes提供的`GenericForeignKey类`与`GenericReleation类`去避免因大量复杂的ORM查询导致的程序效率降低，而且在实际项目中使用contenttypes还会优化项目数据库表结构的设计，避免数据库中产生许多无用的数据。
本文用一个简单的业务为例详细为大家介绍一下使用contenttypes组件有何优点、组件的具体用法及其使用场景等等。<!-- more -->
### `Django中的contenttypes组件`
假如说，公司A的主要业务是培训青少年人工智能与机器人技术，主要分类是`基础课程`与`高级课程`，而这两个大的分类中又有其他具体的课程。每个课程根据培训周期的不同培训费用的计算方式也不一样。也许没有做过相关业务的同学还没有get到相关的点，这里我先粗略的画一下表结构，后面再详细阐述。
#### `初版表结构设计`
如果不用contenttypes，大致的表结构会这么设计：
![image-20190922200038809](http://whw.pythonav.cn/contenttypes1.png)
由上图我们可以看到基础课程与高级课程下有多个不同的子课程。
接下来看一下`价格策略`这张表，每个小课程（比如Python初级与Python高级）由于培训周期的不同价格是不一样的`（这里只是模拟，请勿对号入座）`，并且PricePolicy表中`journeycourse_id`与`seniorcourse_id`是与两个课程表建立外键的字段——此时聪明的你立刻就发现问题了：初级课程的周期最多周期只有21天，而高级课程的培训周期是按月来计算的！如果这样会导致数据库中产生许多无用的空数据！
上面这个问题这是一个很典型的`一张表与多张表动态的创建ForeignKey关系`的问题，而要解决上面产生无用数据的问题`可以选择使用Django中的contenttypes`。
#### `使用contenttypes的表结构`
![image-20190922202135890](http://whw.pythonav.cn/contenttypes2.png)
**详细说明如下：**
1. 新增了一个ContentType表（settings中默认注册了contenttypes组件，Django启动时会自动生成），这个表中存放了model的名称；
2. 价格策略表中的content_type_id字段是`PricePolicy`表与`ContentType`表之间外键关联的字段；
3. 价格策略表中的object_id字段对应的是`基础课程`或者`高级课程`中具体的子类课程的id；
4. 在价格策略表中，我们可以根据大课程的分类`content_type_id`、子课程的id`object_id`以及培训的周期`period`确定一条唯一的记录！
5. 特别注意`object_id`只是一个普通的字段，它并没有与任何表建立外键关联！
这样设计的话就不会出现上面那种空的无效数据了！
#### `contenttypes的具体实现`
老规矩，先上代码，后面给出具体的说明！
项目应用中的models.py文件如下：
```python
from django.db import models
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey,GenericRelation

class JuniorCourse(models.Model):
    """初级课程"""
    name = models.CharField(max_length=64,verbose_name='初级课程名称')
    course_img = models.CharField(max_length=255, verbose_name="课程缩略图")
    brief = models.TextField(verbose_name="初级课程简介", )
    
    # 不会在数据库中生成列，只是帮我们添加与查询数据用的
    # 建立GenericRelation的字段如果被删除的话，PricePolicy中对应的记录也会被删掉！
    policy_lst = GenericRelation('PricePolicy')

class SeniorCourse(models.Model):
    """高级课程"""
    name = models.CharField(max_length=64, verbose_name='进阶级课程名称')
    course_img = models.CharField(max_length=255, verbose_name="课程缩略图")
    brief = models.TextField(verbose_name="进阶级课程简介", )

    # 不会在数据库中生成列，只是帮我们添加与查询数据用的
    # 建立GenericRelation的字段如果被删除的话，PricePolicy中对应的记录也会被删掉！
    policy_lst = GenericRelation('PricePolicy')


class PricePolicy(models.Model):
    """价格策略——课程有效期与价格表"""
    # 与ContentType表做关联 —— 找到大的分类
    content_type = models.ForeignKey(to=ContentType,on_delete=models.CASCADE)
    # 对应"大分类中的课程的id"，用正整数表示 —— 名字必须叫object_id
    object_id = models.PositiveIntegerField()

    # 不会在数据库中生成列，只是帮我们添加与查询查询数据用的
    content_object = GenericForeignKey('content_type','object_id')

    valid_period_choices = (
        (1,'1天'),
        (3,'3天'),
        (7,'1周'),
        (14,'2周'),
        (21,'3周'),
        (30,'1个月'),
        (60,'2个月'),
        (90,'3个月'),
    )
    valid_period = models.SmallIntegerField(choices=valid_period_choices,verbose_name='课程周期')
    price = models.DecimalField(max_digits=8,decimal_places=1)
```
**详细说明如下：**
**1、**大家首先要注意一点：由于项目的settings.py文件中默认注册了`django.contrib.contenttypes`这个组件，所以我们在`执行数据库迁移指令`的时候，Django会自动为我们创建一张铭文`django_content_type`的表，存放项目应用与应用里面model对应关系：
![image-20190922203753923](http://whw.pythonav.cn/contenttypes3.png)
可以看到，最下面那三条记录就是我上面的代码执行`数据库迁移指令`后生成的——我的应用名叫testapp。
**2、**引用这张`django_content_type`表的方式是：
```python
from django.contrib.contenttypes.models import ContentType
```
注意不要改这张表！我们从里面查数据就好了——基本上都是查model。
**3、**我们来看一下`PricePolicy`这个类：首先它与ContentType表建立了外键关联，`content_object`这个属性是`GenericForeignKey`类实例化的对象，参数是上面的`content_type`与`object_id`属性——这步操作不会在数据库中创建新的列，是为了方便我们后续添加与查询数据用的。我们在知道了价格策略的条件下可以根据`GenericForeignKey`对应的属性查到具体的课程。
**4、**`GenericRelation`写在了具体课程的类下面，我们可以它实例化的属性，拿课程对象去查询这个课程对应的价格策略都有哪些——但是要注意一点：GenericRelation对应字段的对象如果被删除的话，PricePolicy中对应的记录也会被删掉！也就是说，根据上面建立的model关系，如果我删除了一条seniorcourse中的记录，那与这条记录对应的pricepolicy中的记录也会被删除！
#### `视图函数中的增删改查`
上面介绍了在`models.py`中如何用contenttypes，接下来我们来看看如何在视图函数中使用它。
##### `向价格策略表中添加数据`
由于上面的`models.py`文件中我们在`PricePolicy`类中定义了`content_object`这个`GenericForeignkey`类的实例化对象，在创建数据的时候直接指定`content_object`就可以了：
```python
# 与seniorcourse表的关系
models.PricePolicy.objects.create(
    valid_period=7,
    price=123.45,
    # 直接根据SeniorCourse对象添加
    content_object = models.SeniorCourse.objects.get(pk=2),
)
# 与juniorcourse表的关系
models.PricePolicy.objects.create(
    valid_period=3,
    price=35.66,
    # # 直接根据JuniorCourse对象添加
    content_object = models.JuniorCourse.objects.get(pk=1),
```

其他两张课程表直接添加数据就好了，没有任何的约束。

##### `根据某个价格策略对象去找与它对应的表的数据`

这里其实还是利用了`GenericForeignKey`，自动`跨表查询`：

```python
from django.contrib.contenttypes.models import ContentType

# 在ContentType表中找到具体的表
contenttype_obj = ContentType.objects.get(model='seniorcourse')
# 找到价格策略对象
# 注意get是查询唯一对象的方法，所以给出的条件必须保证只查到一个对象！
obj=models.PricePolicy.objects.get(content_type=contenttype_obj,object_id=1,valid_period=)
    ### 直接“跨表查询”就好了
    print(obj.content_object.name)
```
##### `找到某个课程相关的所有价格策略***`
这个在实际中用的还是很多的！这里用到了`GenericRelation`，也就是上面`models.py`文件中定义在高级课程与初级课程类中的那两个属性。
```python
obj = models.SeniorCourse.objects.get(pk=1)
	# 注意这里必须用all()方法
    for i in obj.policy_lst.all():
        print(i.pk,i.valid_period,i.price)
```
##### `课程中如果用了GenericRelation类实例化出来的对象作为属性的话，删除课程中的数据时，价格策略中与之相关的数据也会一起被删除！`
```python
# 删除了SeniorCourse中的一个课程后，PricePublic中与之相关的数据都删除了！！！
models.SeniorCourse.objects.get(pk=1).delete()
```
### `contenttypes使用小结`
根据上面的介绍，其实contenttypes的使用大致分为这几步：
1. 创建ContentType表（Django的settings中默认注册了contenttypes应用，启动时会自动生成）；
2. 价格策略表与ContentType表建立外键关联关系；
3. 定义`GenericForeignKey`属性，根据价格策略表中的记录去查询具体课程的信息或者根据课程对象创建价格策略的记录；
4. 定义`GenericRelation`的属性，根据具体的课程对象在价格策略表中查找与这个课程相关的所有的价格策略；
5. 定义了`GenericRelation`的属性的话需要注意：删除课程中的一条记录价格策略表中与这个课程相关的所有记录也都会被删除掉！
### `contenttypes的使用场景`
在实际中，contenttypes绝大多数都用在`一张表与N张表动态的创建ForeignKey关系`的场景下。
其实实际中contenttypes的使用场景还是很多的，比如`评论功能`：如果一个网站的文章、视频、图片等都需要加评论的话，我们可以把评论这张表与文章、图片、视频都建立向上面那样的`外键关联`；再拿上面的例子来说，我们可以为不同分类下的没门课程加入`优惠券`，优惠券这张表跟`价格策略表`性质一样，也需要与那两张课程表建立外键关系，此时使用contenttypes会十分高效！
### `更多参考文章`
[官方文档](https://docs.djangoproject.com/en/2.1/ref/contrib/contenttypes/)
[https://blog.csdn.net/laughing2333/article/details/53014267](https://blog.csdn.net/laughing2333/article/details/53014267)
[https://blog.csdn.net/weixin_42134789/article/details/80690182](https://blog.csdn.net/weixin_42134789/article/details/80690182)
