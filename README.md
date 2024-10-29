
### 1\. 源码分析


注意：以下代码片段为方便理解已进行简化，只保留了与序列化功能相关的代码


序列化的源码中涉及到了元类的概念，我在这里简单说明一下：元类（metaclass）是一个高级概念，用于定义类的创建行为。简单来说，元类是创建类的类，它决定了类的创建方式和行为。


在 Python 中一切皆为对象，包括类。每个类都有一个元类，它定义了如何创建这个类。通常情况下 Python 会使用默认的元类 type 来创建类。但是，当我们需要对类的创建过程进行自定义时，就可以使用元类，举例：



```
class Mytype(type)
	def __new__(cls,name,bases,attrs):   # 类名，继承的父类 ，成员
        # 此处可对要创建的类进行操作
        del attrs["v1"]
        attrs["name"] = "harry"
        
        xx = super().__new__(cls,name,bases,attrs)  # 调用type类创建对象（这个对象就是Bar类）
        retyrn xx 

        
class Bar(object, metaclass=Mytype)  # metaclass指定自定义元类
	v1 = 123
    
    def func(self):
        pass
    
由于元类中删除了v1属性，且增加了name属性，因此此时Bar中无v1属性，且多了name属性

```

另：**父类**如果指定了元类metaclass，则其**子类**都**默认**是用该元类来创建类


补充：实例化Bar类时，相当于是 type对象（），因此会触发type类的\_\_call\_\_方法，其中就调用了Bar的\_\_new\_\_和\_\_init\_\_，因此在实例化类时才会自动触发类的\_\_new\_\_和\_\_init\_\_方法。本质上是因为 对象（） 而调用了type元类的call方法；



#### Serializers组件主要有两个功能：序列化和数据校验


1. 序列化部分：
首先定义一个序列化类



```
class DepartSerializer(serializers.Serializer):
    '''Serializer校验'''
    # 内置校验
    title = serializers.CharField(required=True, max_length=20, min_length=6)
    order = serializers.IntegerField(required=False, max_value=100, min_value=10)
    count = serializers.ChoiceField(choices=[(1, "高级"), (2, "中级")])

```

查看Serializer的父类，可知其是通过SerializerMetaclass元类创建的



```
Serializer(BaseSerializer, metaclass=SerializerMetaclass)

```

SerializerMetaclass元类源码：



```
class SerializerMetaclass(type):
    @classmethod
    def _get_declared_fields(cls, bases, attrs):
        fields = [(field_name, attrs.pop(field_name))  # 通过循环获取field字段对象
                  for field_name, obj in list(attrs.items())
                  if isinstance(obj, Field)]
        fields.sort(key=lambda x: x[1]._creation_counter)

        known = set(attrs)
        def visit(name):
            known.add(name)
            return name

        base_fields = [
            (visit(name), f)
            for base in bases if hasattr(base, '_declared_fields')
            for name, f in base._declared_fields.items() if name not in known
        ]

        return OrderedDict(base_fields + fields)

    def __new__(cls, name, bases, attrs):
        attrs['_declared_fields'] = cls._get_declared_fields(bases, attrs)    # 为类中增加了_declared_fields属性，其中封装了所有的Field字段名及对应的对象
        return super().__new__(cls, name, bases, attrs)

```

通过serializer.data触发序列化流程：



```
    @property
    def data(self):
        ret = super().data   # 寻找其父类BaseSerializer的data方法
        return ReturnDict(ret, serializer=self)

```

BaseSerializer的data方法源码：



```
    @property
    def data(self):
        if hasattr(self, 'initial_data') and not hasattr(self, '_validated_data'):
            msg = (
                'When a serializer is passed a `data` keyword argument you '
                'must call `.is_valid()` before attempting to access the '
                'serialized `.data` representation.\n'
                'You should either call `.is_valid()` first, '
                'or access `.initial_data` instead.'
            )
            raise AssertionError(msg)

        if not hasattr(self, '_data'):
            if self.instance is not None and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.instance)    # 执行to_representation方法获取序列化数据
            elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.validated_data)
            else:
                self._data = self.get_initial()
        return self._data


```

to\_representation方法源码（核心）：



```
    def to_representation(self, instance):
        ret = OrderedDict()
        fields = self._readable_fields  # 筛选出可读的字段对象（其内部对_declared_fields字段进行了深拷贝）

        for field in fields:
            try:
                attribute = field.get_attribute(instance)  # 循环字段对象列表，并执行get_attribute方法获取对应的值
            except SkipField:
                continue
            check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
            if check_for_none is None:
                ret[field.field_name] = None
            else:
                ret[field.field_name] = field.to_representation(attribute)  # 执行to_representation转换格式，并将所有数据封装到ret字典中

        return ret

```

get\_attribute方法源码：



```
def get_attribute(self, instance):
    return get_attribute(instance, self.source_attrs)

```


```
def get_attribute(instance, attrs): # attrs为source字段值  instance为模型对象
    for attr in attrs:
        try:
            if isinstance(instance, Mapping):
                instance = instance[attr]
            else:
                instance = getattr(instance, attr)  # 循环获取模型对象最终的attr的值
        except ObjectDoesNotExist:
            return None
    return instance  # 返回该字段值

```


2\. 数据校验部分
使用is\_valid方法校验数据，获取\_errors数据，\_errors存在则is\_valid返回False。在执行该函数的过程中，触发了run\_validation方法：



```
    def is_valid(self, raise_exception=False):
        if not hasattr(self, '_validated_data'):
            try: # 触发了run_validation方法
                self._validated_data = self.run_validation(self.initial_data) 
            except ValidationError as exc:
                self._validated_data = {}
                self._errors = exc.detail
            else:
                self._errors = {}

        if self._errors and raise_exception:
            raise ValidationError(self.errors)

        return not bool(self._errors)****

```

run\_validation方法，注意该方法是Serializer类下的方法，不是Field类的方法。在to\_internal\_value方法中调用字段内置校验，并执行钩子函数。



```
    def run_validation(self, data=empty):

        (is_empty_value, data) = self.validate_empty_values(data)
        if is_empty_value:
            return data

        value = self.to_internal_value(data)  # 调用字段内置校验，并执行钩子函数
        try:
            self.run_validators(value)
            value = self.validate(value)
            assert value is not None, '.validate() should return the validated data'
        except (ValidationError, DjangoValidationError) as exc:
            raise ValidationError(detail=as_serializer_error(exc))

        return value

```

to\_internal\_value方法，fileds从\_declared\_fields中深拷贝而得到，且只包含了只写的字段对象



```
    def to_internal_value(self, data):
        if not isinstance(data, Mapping):
            message = self.error_messages['invalid'].format(
                datatype=type(data).__name__
            )
            raise ValidationError({
                api_settings.NON_FIELD_ERRORS_KEY: [message]
            }, code='invalid')

        ret = OrderedDict()
        errors = OrderedDict()
        fields = self._writable_fields  # 筛选只写的字段对象

        for field in fields:
            validate_method = getattr(self, 'validate_' + field.field_name, None)
            primitive_value = field.get_value(data)
            try:
                validated_value = field.run_validation(primitive_value) # 执行内置校验
                if validate_method is not None:
                    validated_value = validate_method(validated_value)  # 执行钩子函数进行校验
            except ValidationError as exc:
                errors[field.field_name] = exc.detail
            except DjangoValidationError as exc:
                errors[field.field_name] = get_error_detail(exc)
            except SkipField:
                pass
            else:
                set_value(ret, field.source_attrs, validated_value)
        if errors:
            raise ValidationError(errors)
        return ret

```

run\_validation内置校验：



```
    def run_validation(self, data=empty):
        if data == '' or (self.trim_whitespace and str(data).strip() == ''):
            if not self.allow_blank:
                self.fail('blank')
            return ''
        return super().run_validation(data)

    # 父类的run_validation方法
    def run_validation(self, data=empty):

        (is_empty_value, data) = self.validate_empty_values(data)
        if is_empty_value:
            return data
        value = self.to_internal_value(data)
        self.run_validators(value)  # 调用字段定义的run_validators进行校验
        return value

```

### 2、源码改编：


* 自定义钩子：让某字段既能支持前端传入，又能自定义序列化返回的值；（SerializerMethodField默认是只可读的，用户无法输入，而普通field又无法自定义复杂逻辑返回值）


思路：在调用ser.data开始序列化后的to\_representation方法中判断有无自定义格式的钩子，如果有则替换掉该字段对象的值



```
    def to_representation(self, instance):
        ret = OrderedDict()
        fields = self._readable_fields

        for field in fields:
            if hasattr(self, 'get_%s' % field.field_name):  # 判断是否有"get_xxx"形式的函数，如则执行该方法并将instance传入
                value = getattr(self, 'get_%s' % field.field_name)(instance)
                ret[field.field_name] = value
            else:
                try:
                    attribute = field.get_attribute(instance)
                except SkipField:
                    continue

                check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
                if check_for_none is None:
                    ret[field.field_name] = None
                else:
                    ret[field.field_name] = field.to_representation(attribute)

        return ret

```

如果其他类中也需要使用该重写方法，可将该重新方法封装成类，其他类中继承该类后，即可不用每次都重写to\_representation方法


 本博客参考[wgetCloud机场](https://tabijibiyori.org)。转载请注明出处！
