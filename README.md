# Django风格指南

Django风格指南，给Django使用者的一些关于目录结构、使用的建议。该指南中所有的例子你都可以在DeerU([https://github.com/gojuukaze/DeerU](https://github.com/gojuukaze/DeerU))项目中找到。  

项目地址：https://github.com/gojuukaze/django-styleguide

## project结构

project结构建议是这样的（[demo](https://github.com/gojuukaze/DeerU)）

```
 app1/
 	...
 app2/
 	...
 html_app/
 	...
 project_name/
 	...
 tool/
 	...

```

`app1`，`app2`，`project_name`三个目录是标准的django目录结构，除此之外多了`html_app`与`tool`。   
 
* `html_app`：这是用来专门存放前端代码的目录，虽然你可以把前端代码放到不同的app中，但更加建议你单独放在前端专属的app中，在此app中用不同的目录来区分不同app的前端代码。
* `tool` ： 用来存放一些每个app都会用到的公用函数，比如：时间处理函数，Exception类等

## 子project目录结构
项目下的project_name目录是存放项目`settings`的地方，有两种结构可供选择。

### 第一种，不提交settings.py

[demo](https://github.com/gojuukaze/DeerU/tree/master/deeru) （demo中并没有严格按照这个目录结构，因为demo需要支持git升级且不影响已修改的settings配置。  ）  


```bash
# * 表示不提交到仓库

project_name/
	* settings.py
	settings_common.py
	settings_dev.py
	settings_test.py
	settings_prod.py
	* settings_local.py

```

* `settings_common.py` ： 建议首先需要一个`settings_common.py`，它用来保存一些在dev、test、product三个环境用到公用的配置。
* `settings.py ` ：Django的settings文件，之所以不建议提交是为了防止把开发时的修改提交了上去，造成bug。在实际上线时根据不同环境把不同环境的settings文件拷贝为`settings.py`，或者手动指定使用的settings
* `settings_dev.py` ：dev环境的专有配置，比如数据库地址、密码，缓存地址等，它里面引用了settings_common，如：

  ```python
  from xxx.settings_common import *
  ```
* `settings_local.py` ：这个用于本地开发的settings，因为开发时会对settings配置进行修改，建议此文件不提交代码仓库，防止对其他人的开发造成干扰。

### 第二种，提交settings.py

```bash
# * 表示不提交到仓库

project_name/
	settings.py
	settings_common.py
	settings_dev.py
	settings_test.py
	settings_prod.py
	* settings_local.py

```

* 此结构提交settings.py，在settings.py中判断引用哪个环境的settings。
  settings.py的代码如下：
  
  ```
  import os
  from detox_web.settings_common import *
  ENV=os.getenv('ENV','local')
  
  if ENV == 'local':
    from detox_web.settings_local import *
  elif ENV == 'test':
    from detox_web.settings_test import *
  elif ENV == 'prod':
    from detox_web.settings_prod import *
    
  ...
  
  ```
  
这个结构有个缺点，在执行远程命令或crontab脚本时需要显示设置环境变量ENV，因为在执行非交互式操作时，不会读取`.bashrc`中的配置

## app结构
### 初始、小型的项目app
对于初始、小型的项目只需要一个app，其结构建议是这样的（另外建议html的代码单独放到一个app中，其结构下面有说明）

```
app_name\
 	apps.py
 	admin.py
 	consts.py
 	models.py
 	db_managers.py
 	managers.py
 	urls.py
 	views.py
 	class_viws.py
 	
```

* `consts.py` ： 保存一些app内的全局变量
* `views.py` ：存放函数view
* `class_viws.py` ：存放class view 
* `db_managers.py` ： 存放每个model的纯数据库操作函数，定义model后应在`db_managers.py`中加入该model的基本db操作函数，在view中不应有`xx.objects.xx`这样的操作。比如（[demo](https://github.com/gojuukaze/DeerU/blob/master/app/db_manager/content_manager.py)）：  

  ```python
  
  ########### User ############
  
  def create_user(name, age):
      return User.objects.create(name=name, age=age)
      
  def get_user_by_id(id):
      try:
          return User.objects.get(id=id)
      except:
          return None   
          
  def get_user_by_id_with_lock(id):
      try:
          return User.objects.select_for_update().get(id=id)
      except:
          return None
          
  def get_all_user():
      return User.objects.all()
          
  def filter_user_by_age(age):
      return User.objects.filter(age=age)
      
  def filter_user_by_age_and_name(age, name):
      return User.objects.filter(age=age, name=name)
   
  def delete_user_by_id(id):
      return User.objects.delete(id=id)
   
  # 在函数中进行进一步处理，这个根据喜好
  def filter_user_order_by_age():
      return User.objects.filter().order_by('age')
      
  	```
  	
  	* 函数命名为`动词`+`model名`+`by 条件`+`with 其他操作`；动词一般就是objects后面跟的方法  
  	
  	* 对于`get`操作建议返回查询值或者`None`，而不是抛错。

  	* 对于`filter`操作建议直接返回`QuerySet`，方便外部对结果进行进一步操作，比如:count、odder。
  	* 对于`filter`的结果很多时候你需要进一步处理，你可以直接用filter函数的返回结果进行处理，或者像`filter_user_order_by_age()`一样直接把它封装成一个函数，这个根据个人偏好选择。
  	
  	  ```python
  	  users = get_all_user().order_by('age')
  	  # or
  	  users = filter_user_order_by_age()
  	  ```
  	* 一般来说这里面的函数只是单纯的db操作，不应有太多的逻辑。

* `managers.py` ：存放model相关的逻辑处理函数，功能类函数，以及一些该app专用的函数，比如：

  ```python
  ######### user ###########
  
  def login_user(name, passwd):
      user = get_user_by_name(name=name)
      if not user:
          return False
      return login_user(user, passwd)
  
  def add_user(name, age):
      user = get_user_by_name(name)
      if user:
          return None, "存在相同name"
      return create_user(name, age), ""
      
  def get_online_user():
      online_user=[]
      users=get_all_user()
      for u in users:
          if is_ online(u):
              online_user.append(u)
      return online_user
  
  ```
  	
### 中型项目app结构

随着项目不断迭代，功能增加后，单个`views.py`文件可能无法放下所有代码，这时你需要对app进行进一步的分层（[demo](https://github.com/gojuukaze/DeerU/tree/master/app)）

```bash
# demo中对models也进行了拆分，之后会讲到

app_name\
 	apps.py
 	admin.py
 	consts.py
 	models.py
 	db_managers\
 		user_managers.py
 		xx_managers.py
 	managers\
 		user_managers.py
 		xx_managers.py
 	urls\
 		__init__.py
 		# 根据版本划分
 		v1_urls.py
 		v2_urls.py
 		
 		# 根据功能，model划分
 		user_urls.py
 	views\
 		# 根据版本划分
 		v1_views.py
 		v2_views.py
 		v1_class_viws.py
 		
 		# 根据功能，model划分
 		user_views.py
 		user_class_views.py

```
可以看到该结构主要对views、urls、managers、db_managers建立了单独的文件夹。  
结合实际情况，根据model，功能，版本把单个文件py文件拆分为多个。

* 对于urls目录，注意它的`__init__.py`很重要，它里面include了其他的子urls文件，如（[demo](https://github.com/gojuukaze/DeerU/blob/master/app/urls/__init__.py)）：

  ```python
  urlpatterns = [
      path('v1/', include('app_name.urls.v1_urls')),
      path('v2/', include('app_name.urls.v2_urls')),
    
      path('user/', include('app_name.urls.user_urls')),

  ]
  ```
  在项目的入口`project_name/urls.py`中这样include app的url：
  
  ```python
  urlpatterns = [
      path('app_name/', include('app_name.urls')),,

  ]
  ```
  这样做可以防止每次新增url文件都修改项目的`urls.py`


### 中大型项目app目录结构

项目再次扩展，单个`models.py`文件无法放下所有model了，这时需要对`models.py`文件进行拆分，新的结构如下（[demo](https://github.com/gojuukaze/DeerU/tree/master/app)）:

```bash
app_name\
 	apps.py
 	admin.py
 	consts.py
 	models.py
 	app_models\
 		user_models.py
 		xx_models.py
 	db_managers\
 	managers\
 	urls\
 	views\

```

根据实际情况把`models.py`分成多个子model，建议在这次拆分model时充分考虑所有的情况，重点思考这样拆分是否可以做的model之间的解耦。

* 大家可能注意到，`models.py`文件并没有删除，因为这个django的默认model位置，在`models.py`中这样把其他model引入（[demo](https://github.com/gojuukaze/DeerU/blob/master/app/models.py)）：

  ```python
  from app_name.app_models.user_model import *
  from app_name.app_models.xxx_model import *

  ```
  在子model文件中把每个model加到`__all__`中（[demo](https://github.com/gojuukaze/DeerU/blob/master/app/app_models/content_model.py)）：
  
  ```python
  # user_models.py
  
  __all__ = ['User', 'UserGroup',]
  
  class User(models. Model):
      pass
     
  class UserGroup(models. Model):
      pass
  ```
  
  需要用到model时你应该从子model文件import，而不是从models.py（[demo](https://github.com/gojuukaze/DeerU/blob/master/app/db_manager/content_manager.py#L1)）：
  
  ```python
  from app_name.app_models.user_models import User
  
  user = User.objects.get(id=1)
  ```
  
### 大型项目app结构
发展到大型项目时，单个app已经无法放下所有文件了，应该根据实际情况把一些model、功能单独拿出来作为一个新的app。如果在中大型项目时拆分足够好，那么把代码放到一个新的app会很轻松，因此中大型项目的app结构拆分十分重要。


## html_app结构

建议吧html代码单独放到一个app中，结构如下

```bash
html_app\
 	apps.py
 	templates\
 		# 根据app划分
 		app1_templates\
 			app1_html.html
 			
 		# 根据model、功能划分
 		user_templates\
 			get_user_html.html
 			create_user_html.html
 	static\
 		app1_static\
 			css\
 			js\
 		user_static\
 			css\
 			js\
 		
 		# or
 		
 		css\
 			app1_css\
 			user_css\
 		js\
 			app1_js\
 			user_js\

```

根据实际情况对`templates`，`static`目录的结构进行划分，这里没有什么标准建议 ，具体问题具体分析。


## view命名
view函数、类命名应该以view结尾，方便与其他函数区分，如：

```python

def create_user_view(request):
    pass

def get_user_view(request):
    pass
    
class UserListView(ListView):
    pass
```

## model导入

导入model时建议直接导入model，而不是导入model所在的py文件，如：

```python
# 不建议这样做
# from app_name import models

from app_name.models import User, UserGroup
```
这样如果对model进行删除、迁移可以很快的发现错误。

## html中引入静态文件
在html中应使用软连接引入静态文件，如（[demo](https://github.com/gojuukaze/DeerU/blob/master/base_theme/templates/base_theme/head.html)）：

```html
{% load static %}
<link href="{% static '/app_name/css/base_theme.css' %}" type="text/css"/>
<script defer src="{% static '/base_theme/js/fontawesome.js' %}"></script>
```
为了保证引入静态文件成功，你应该正确的配置`settings.py`文件，具体可以参考官方文档。

## 正确导入settings

需要使用settings中的配置，你应该这样import：

```python
from django.conf import settings

settings.xxx
```

## 是否提交migrations文件

提交migrations可以方便的管理数据库的表，为新表增加默认值。但如果开发时，把一些临时的、不完善的migrations文件提交上去有可能对数据库造成不可逆的修改。  
因此一些项目中选择不提交migrations文件，上线时在服务器上生成migrations文件。  
这个问题同样需要根据实际情况决定，但如果你选择提交，那要注意提交的migrations文件一定是经过测试的。
