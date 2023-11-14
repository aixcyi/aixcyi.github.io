## 问题复现

```python
from datetime import date;

print(date(2023, 11, 3).strftime('%Y年%m月%d日'))
```

以上代码在 Python 3.7 (Windows) 中可以复现以下报错，在其它操作系统或者更高版本就没有，更低版本则没试过。

```text
Traceback (most recent call last):
  File "C:\Program Files\Python37\lib\code.py", line 90, in runcode
    exec(code, self.locals)
  File "<input>", line 1, in <module>
UnicodeEncodeError: 'locale' codec can't encode character '\u5e74' in position 2: encoding error
```

这个问题是写业务时指定了DRF序列化器日期字段格式之后发现的：

```python
from rest_framework import serializers

class MemberInfoSerializer(serializers.ModelSerializer):
    create_at = serializers.DateTimeField(format='%Y/%m/%d')
    gender = serializers.CharField(source='get_gender_display', default=None)
    birth = serializers.DateField(format='%Y年%m月%d日')
```

在 `create_at` 序列化时不会报错，轮到 `birth` 就会报 `UnicodeEncodeError` ，且错误的消息一致。

## 解决方案

#### 0x1 指定DRF全局设置

在 `./[project]/settings.py` 中指定DRF序列化器的 **默认** 日期时间格式。这是最佳方案，统一的格式可以免去冗余代码，也方便前端作统一解析。

```python
# Django 全局设置
DATE_FORMAT = '%Y-%m-%d'
TIME_FORMAT = '%H:%M:%S'
DATETIME_FORMAT = '%Y-%m-%d %H:%M:%S'

# DRF 全局设置
REST_FRAMEWORK = {
    'DATE_FORMAT': DATE_FORMAT,
    'TIME_FORMAT': TIME_FORMAT,
    'DATETIME_FORMAT': DATETIME_FORMAT,
}
```

#### 0x2 手动格式化

如果必须自定义格式，但字段只读，只需要改用类方法进行序列化：

```python
from rest_framework import serializers

class MemberInfoSerializer(serializers.ModelSerializer):
    create_at = serializers.SerializerMethodField()

    def get_create_at(self, instance) -> str:
        return instance.create_at.strftime('%Y{0}%m{1}%d{2}').format(*'年月日')
```

这个方案摘自[Stack Overflow](https://stackoverflow.com/a/16035152)。注意：[SerializerMethodField](https://www.django-rest-framework.org/api-guide/fields/#serializermethodfield) 会将 `read_only` 参数的值覆写为 `True` 。

#### 0x3 自定义字段

如果必须自定义格式，并且字段需要读和写，那么只能[自定义字段](https://www.django-rest-framework.org/api-guide/fields/#a-basic-custom-field)：

```python
from datetime import date, datetime
from rest_framework import serializers

class BirthdayField(serializers.Field):

    def to_representation(self, value: date) -> str:
        return value.strftime('%Y{0}%m{1}%d{2}').format(*'年月日')

    def to_internal_value(self, value: str) -> date:
        return datetime.strptime(value, '%Y年%m月%d日')


class MemberInfoSerializer(serializers.ModelSerializer):
    birth = BirthdayField()
```

#### 0x4 设置语言环境

如果不是用DRF的话，可以用 [`setlocale()`](https://docs.python.org/zh-cn/3.7/library/locale.html#locale.setlocale) 方法：

```python
from contextlib import contextmanager
import datetime
import locale

@contextmanager
def localize(local_name: str, lc_var=locale.LC_ALL):
    orgin_local = locale.getlocale()
    try:
        yield locale.setlocale(lc_var, local_name)
    finally:
        locale.setlocale(lc_var, orgin_local)


with localize('zh'):
    print(datetime.datetime.now().strftime('%Y年%m月%d日'))
```

这个方案搬运自[Stack Overflow](https://stackoverflow.com/a/62790288)，不过我没有应用到实际在业务中，你可能需要参阅[国际化服务](https://docs.python.org/zh-cn/3.7/library/locale.html)这个标准库来了解潜在的bug。
