Поступила установка, что у одного **User** может быть только один объект **Point**. Эту логику можно осуществить через использование **ForeignKey** и использования **constraints** в БД.

## Исходный код

```python

class User(models.Model): ...


class Point(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT, verbose_name='User', related_name='point')
    is_active = models.BooleanField(default=True, verbose_name='Is Active')
    class Meta:
        verbose_name = 'Point'
        verbose_name_plural = 'Points'
        

class Sale(models.Model):
    point = models.ForeignKey(Point, on_delete=models.PROTECT, verbose_name='Point', related_name='sale')
    ...
```


Сразу отвечу на вопрос, почему не **OneToOne**: если пользователь деактивирует **Point**, и в дальнейшем, спустя время, захочет создать **Point** с уже новыми данными? При этом у нас уже есть объекты **Sale** которые содержат ранее деактивированный **Point**, которые никак нельзя удалять из БД. Вот в этот момент при использовании **OneToOne** вы начнете сожалеть о выбранном решении.

====Решение проблемы====
%%(Python)

class Point(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT, verbose_name='User', related_name='point')
    is_active = models.BooleanField(default=True, verbose_name='Is Active')
    class Meta:
        verbose_name = 'Point'
        verbose_name_plural = 'Points'
        constraints = (models.UniqueConstraint(fields=('user',), name='user_point'), )
%%

Таким образом, можно ограничить создание дополнительных **Point** для конкретного **User**.

====Если нужно, чтобы было много Point, но активный объект для User был только один?====
Решение:
%%(Python)
class Point(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT, verbose_name='User', related_name='point')
    is_active = models.BooleanField(default=True, verbose_name='Is Active')
    class Meta:
        verbose_name = 'Point'
        verbose_name_plural = 'Points'
        constraints = (models.UniqueConstraint(fields=('user', 'is_active', ), condition=models.Q(is_active=True), name='user_point'), )
%%

====Зачем это все нужно, когда можно просто перед созданием объекта проверять, а есть ли он уже в БД?====
Затем, что при обращении к БД с разницей в доли секунды, скорее всего ваши потоки оба получат **None**, и создадут два **Point** для одного **User**, а при использовании механизма constraints сама БД не позволит создать два **Point** для одного **User**.

