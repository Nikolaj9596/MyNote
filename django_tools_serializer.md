# Django Tools for serializers

### Decorators for convert field values to camel case or snake case

```python
def transform_serializer_input(snake_case_fields: list[str])\
    -> Callable[
        [Type[serializers.Serializer]], Type[serializers.Serializer]
    ]:
    """Decorate DRF Serializer.
    Take list of field names, transform their values into snake_case
    upon deserialization
    """
    def inner_decorator(cls: Type[serializers.Serializer]):
        """Decorate serializer"""
        to_internal_value_old = cls.to_internal_value

        @wraps(cls.to_internal_value)
        def to_internal_value_decorated(self, data):
            """Transform all fields listed in snake_case_fields
            into snake_case
            """
            for field_name in snake_case_fields:
                if data.get(field_name):
                    data[field_name] = snake_case(line=data[field_name])
            return to_internal_value_old(self, data)
        cls.to_internal_value = to_internal_value_decorated
        return cls
    return inner_decorator


def transform_serializer_output(camel_case_fields: list[str])\
    -> Callable[
        [Type[serializers.Serializer]], Type[serializers.Serializer]
    ]:
    """Decorate DRF Serializer.
    Take list of field names, transform their values into camelCase
    upon serialization
    """
    def inner_decorator(cls: Type[serializers.Serializer]):
        """Decorate serializer"""
        to_representation_old = cls.to_representation

        @wraps(cls.to_representation)
        def to_representation_decorated(self, data):
            """Transform all fields listed in camel_case_fields
            into camelCase
            """
            representation = to_representation_old(self, instance=data)
            for field_name in camel_case_fields:
                if representation.get(field_name):
                    representation[field_name] = camel_case(
                        line=representation[field_name]
                    )
            return representation
        cls.to_representation = to_representation_decorated
        return cls
    return inner_decorator


def camel_case(line: str) -> str:
    line = sub(r"(_|-)+", " ", line).title().replace(" ", "")
    return ''.join([line[0].lower(), line[1:]])


def snake_case(line: str) -> str:
    """Transform camelCase string into snake_case"""
    sub_function = lambda match: f"{match.group(1)}_{match.group(2).lower()}"
    return sub('([a-z])([A-Z]+)', sub_function, line)
```
