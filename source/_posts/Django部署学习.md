---
title: Django部署学习
date: 2024-05-10 16:30:38
tags: 
    - 技术
    - python
---
## Django项目创建

### **1.1 创建和配置Django项目**

虚拟环境创建（可选）

首先，安装Django和MySQL的Python库：

```bash
pip install django
pip install mysqlclient
## 跨域库
pip install django-cors-headers
```

注意：mac安装mysqlclient失败解决方案

https://github.com/PyMySQL/mysqlclient

```bash
brew install mysql-client pkg-config
export PKG_CONFIG_PATH="$(brew --prefix)/opt/mysql-client/lib/pkgconfig"
pip install mysqlclient
```

然后，创建Django项目：

```bash
django-admin startproject myproject
cd myproject
```

创建一个Django应用：

```bash
python manage.py startapp myapp
```

### **1.2 配置MySQL数据库**

编辑**`myproject/settings.py`**，配置MySQL数据库连接：

```python

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'your_database_name',
        'USER': 'your_database_user',
        'PASSWORD': 'your_database_password',
        'HOST': 'localhost',  # 或者你的数据库服务器地址
        'PORT': '3306',       # MySQL默认端口
    }
}

INSTALLED_APPS = [
    # ...
    'myapp',
    'corsheaders',  # 允许跨域
    # ...
]

MIDDLEWARE = [
    # ...
    'corsheaders.middleware.CorsMiddleware',
    # ...
]

CORS_ALLOWED_ORIGINS = [
    'http://localhost:8080',  # Vue开发服务器地址
]
```

### **1.3 创建数据库和初始迁移**

```bash
python manage.py migrate
python manage.py createsuperuser
```

### **1.4 创建简单的API视图**

编辑**`myapp/views.py`**，添加一个简单的API视图：

```python
python复制代码
from django.http import JsonResponse

def hello_world(request):
    return JsonResponse({'message': 'Hello, World!'})

```

编辑**`myapp/urls.py`**，配置URL：

```python
from django.urls import path
from .views import hello_world

urlpatterns = [
    path('api/hello/', hello_world, name='hello_world'),
]
```

在**`myproject/urls.py`**中包含应用URL配置：

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),
]
```

### 1.5 运行

启动Django服务

```bash
python manage.py runserver
```

## 2. 用户管理模块

在 Django 中，有多种用户管理和权限管理框架可以与其搭配使用，其中一些提供了基于角色的访问控制（RBAC）功能。这些框架可以帮助你更轻松地实现用户管理、权限管理和角色管理。以下是一些常见的框架：

### **2.1 Django Rest Framework (DRF) + Django Guardian**

**Django Rest Framework (DRF)** 是一个功能强大的 Django 应用，用于构建 Web APIs。 **Django Guardian** 提供了对象级权限管理，可以与 DRF 集成实现细粒度的权限控制。

- **Django Rest Framework:**
    - 官方网站: [Django Rest Framework](https://www.django-rest-framework.org/)
    - 功能: 提供一组用于构建 RESTful APIs 的工具和类视图
- **Django Guardian:**
    - 官方网站: [Django Guardian](https://django-guardian.readthedocs.io/)
    - 功能: 提供对象级权限管理，可以对单个对象进行权限控制

### **2.2 Django-Role-Permissions**

**Django-Role-Permissions** 是一个基于角色的权限管理框架，简单易用，适合需要实现 RBAC 功能的应用。

- 官方网站: [Django-Role-Permissions](https://django-role-permissions.readthedocs.io/)
- 功能: 提供基于角色的权限管理，支持角色的定义和分配，权限的定义和检查

### **2.3. Django-Rules**

**Django-Rules** 是一个用于设置权限和规则的轻量级框架，支持基于角色的访问控制。

- 官方网站: [Django-Rules](https://github.com/dfunckt/django-rules)
- 功能: 提供简洁的 API 来定义规则和权限，支持基于角色和对象的权限管理

### **2.4. Django-User-Accounts (Account)**

**Django-User-Accounts (Account)** 提供了用户注册、登录、密码管理等功能，虽然不直接提供 RBAC 功能，但可以与其他权限管理框架结合使用。

- 官方网站: [Django-User-Accounts](https://github.com/pinax/django-user-accounts)
- 功能: 提供用户注册、登录、密码重置等基本用户管理功能

### **2.5. Django-Allauth**

**Django-Allauth** 是一个处理用户认证的强大框架，支持多种认证方式（包括社交登录）。尽管它不直接提供 RBAC 功能，但可以与其他权限管理框架结合使用。

- 官方网站: [Django-Allauth](https://django-allauth.readthedocs.io/)
- 功能: 支持用户注册、登录、社交认证等功能

### **2.6. Django-Oscar**

**Django-Oscar** 是一个开源的电子商务框架，内置用户管理和权限管理功能，包括 RBAC。

- 官方网站: [Django-Oscar](https://django-oscar.readthedocs.io/)
- 功能: 提供完整的电子商务解决方案，包括用户管理、权限管理、订单管理等

### **2.7. Django-Organizations**

**Django-Organizations** 是一个组织和团队管理框架，适合需要实现基于组织的权限管理的应用。

- 官方网站: [Django-Organizations](https://github.com/bennylope/django-organizations)
- 功能: 提供组织和团队管理，支持多种角色和权限管理

### **示例：结合 Django-Role-Permissions 和 Django**

以下是一个简单的示例，演示如何在 Django 中使用 Django-Role-Permissions 实现 RBAC 功能：

1. 安装 **`django-role-permissions`**:
    
    ```bash
    bash复制代码
    pip install django-role-permissions
    
    ```
    
2. 配置 **`settings.py`**:
    
    ```python
    python复制代码
    INSTALLED_APPS = [
        ...
        'rolepermissions',
    ]
    
    ROLEPERMISSIONS_MODULE = 'myapp.roles'
    
    ```
    
3. 定义角色和权限 (**`myapp/roles.py`**):
    
    ```python
    python复制代码
    from rolepermissions.roles import AbstractUserRole
    
    class Admin(AbstractUserRole):
        available_permissions = {
            'view_dashboard': True,
            'edit_users': True,
        }
    
    class User(AbstractUserRole):
        available_permissions = {
            'view_dashboard': True,
        }
    
    ```
    
4. 在视图中使用权限检查:
    
    ```python
    python复制代码
    from rolepermissions.checkers import has_permission
    
    def my_view(request):
        if has_permission(request.user, 'view_dashboard'):
            # 用户有权限
            ...
        else:
            # 用户没有权限
            ...
    
    ```
    

通过这些框架和方法，你可以在 Django 中实现灵活且强大的用户管理和基于角色的访问控制功能。