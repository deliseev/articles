# [Моделирование полиморфизма в Django с помощью Python](https://realpython.com/modeling-polymorphism-django-python/)

**Оглавление**

- Что такое полиморфизм?
- Почему моделирование полиморфизма является сложной задачей?
- Наивная реализация
- Разрозненная модель
- Полуструктурированная модель
- Абстрактная базовая модель
- Конкретная базовая модель
- Обобщенный внешний ключ
- Заключение

Моделирование полиморфизма в реляционных базах данных является сложной задачей. В данной статье мы представляем несколько методов моделирования для представления полиморфных объектов в реляционной базе данных с помощью объектно-реляционного отображения (ORM) Django.

Этот учебник среднего уровня рассчитан на читателей, уже знакомых с фундаментальным дизайном Django.

# Что такое полиморфизм?
Полиморфизм - это способность объекта принимать различные формы. Частыми примерами полиморфных объектов являются потоки событий, различные типы пользователей и товары на сайте электронной коммерции. Полиморфная модель используется в тех случаях, когда для одного объекта требуется различная функциональность или информация.

В приведенных выше примерах все события регистрируются для последующего использования, но они могут содержать различные данные. Все пользователи должны иметь возможность войти в систему, но у них может быть разная структура профиля. На каждом сайте электронной коммерции пользователь хочет положить в корзину различные товары.

# Почему моделирование полиморфизма является сложной задачей?

Существует множество способов моделирования полиморфизма. Некоторые подходы используют стандартные возможности Django ORM, а некоторые - специальные возможности Django ORM. Основные проблемы, с которыми вы столкнетесь при моделировании полиморфных объектов, заключаются в следующем:

Как представить один полиморфный объект: Полиморфные объекты имеют различные атрибуты. Django ORM сопоставляет атрибуты с колонками в базе данных. В таком случае, как Django ORM должен сопоставлять атрибуты со столбцами таблицы? Должны ли разные объекты располагаться в одной таблице? Должно ли у вас быть несколько таблиц?

Как ссылаться на экземпляры полиморфной модели: Чтобы использовать возможности базы данных и Django ORM, необходимо ссылаться на объекты с помощью внешних ключей. От того, как вы решите представить один полиморфный объект, зависит возможность ссылаться на него.

Чтобы по-настоящему понять сложности моделирования полиморфизма, вам предстоит пройти путь от небольшого книжного магазина с его первым интернет-сайтом до крупного интернет-магазина, продающего самые разные товары. Попутно вы испытаете и проанализируете различные подходы к моделированию полиморфизма с помощью Django ORM.

> Примечание: Для выполнения данного руководства рекомендуется использовать бэкэнд PostgreSQL, Django 2.x и Python 3.
> Возможно использование и других бэкендов баз данных. В тех местах, где используются возможности, характерные только для PostgreSQL, будет представлена альтернатива для других баз данных.

# Наивная реализация

У вас есть книжный магазин в хорошем районе города, расположенный рядом с кофейней, и вы хотите начать продавать книги через Интернет.

Вы продаете только один вид товара: книги. В своем интернет-магазине вы хотите показывать подробную информацию о книгах, например, название и цену. Вы хотите, чтобы пользователи просматривали сайт и собирали много книг, поэтому вам также нужна корзина. В конечном итоге вам нужно будет отправлять книги пользователю, поэтому для расчета стоимости доставки необходимо знать вес каждой книги.

Давайте создадим простую модель для вашего нового книжного магазина:

```python
from django.contrib.auth import get_user_model
from django.db import models


class Book(models.Model):
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )
    weight = models.PositiveIntegerField(
        help_text='in grams',
    )

    def __str__(self) -> str:
        return self.name


class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(Book)
```

Для создания новой книги указывается название, цена и вес:

```python
>>> from naive.models import Book
>>> book = Book.objects.create(name='Python Tricks', price=1000, weight=200)
>>> book
<Product: Python Tricks>
```

Чтобы создать корзину, необходимо сначала связать ее с пользователем:

```python
>>> from django.contrib.auth import get_user_model
>>> haki = get_user_model().create_user('haki')

>>> from naive.models import Cart
>>> cart = Cart.objects.create(user=haki)
```

**За**

- **Легко понять и поддерживать:** достаточно для одного вида продукции.

**Против**  

- **Ограничен однородными продуктами:** Поддерживаются только продукты с одинаковым набором атрибутов. Полиморфизм не учитывается и вообще не допускается.

# Разрозненная модель

После успеха вашего книжного интернет-магазина пользователи стали спрашивать, продаете ли вы также электронные книги. Электронные книги - отличный товар для вашего интернет-магазина, и вы хотите начать их продажу прямо сейчас.

Физическая книга отличается от электронной:

- Электронная книга не имеет веса. Это виртуальный продукт.
- Электронная книга не требует пересылки. Пользователи скачивают ее с веб-сайта.

Чтобы существующая модель поддерживала дополнительную информацию для продажи электронных книг, вы добавляете некоторые поля к существующей модели Book:

```python
from django.contrib.auth import get_user_model
from django.db import models


class Book(models.Model):
    TYPE_PHYSICAL = 'physical'
    TYPE_VIRTUAL = 'virtual'
    TYPE_CHOICES = (
        (TYPE_PHYSICAL, 'Physical'),
        (TYPE_VIRTUAL, 'Virtual'),
    )
    type = models.CharField(
        max_length=20,
        choices=TYPE_CHOICES,
    )

    # Common attributes
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )

    # Specific attributes
    weight = models.PositiveIntegerField(
        help_text='in grams',
    )
    download_link = models.URLField(
        null=True, blank=True,
    )

    def __str__(self) -> str:
        return f'[{self.get_type_display()}] {self.name}'


class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(
        Book,
    )
```

Сначала добавляется поле type, указывающее тип книги. Затем добавляется поле URL для хранения ссылки на скачивание электронной книги.

Чтобы добавить физическую книгу в книжный магазин, выполните следующие действия:

```python
>>> from sparse.models import Book
>>> physical_book = Book.objects.create(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     weight=200,
...     download_link=None,
... )
>>> physical_book
<Book: [Physical] Python Tricks>
```

Чтобы добавить новую электронную книгу, необходимо выполнить следующие действия:

```python
>>> virtual_book = Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='The Old Man and the Sea',
...     price=1500,
...     weight=0,
...     download_link='https://books.com/12345',
... )
>>> virtual_book
<Book: [Virtual] The Old Man and the Sea>
```

Теперь ваши пользователи могут добавлять в корзину как книги, так и электронные книги:

```python
>>> from sparse.models import Cart
>>> cart = Cart.objects.create(user=user)
>>> cart.books.add(physical_book, virtual_book)
>>> cart.books.all()
<QuerySet [<Book: [Physical] Python Tricks>, <Book: [Virtual] The Old Man and the Sea>]>
```

Виртуальные книги пользуются большим успехом, и вы решаете нанять сотрудников. Новые сотрудники, видимо, не очень хорошо разбираются в технологиях, и вы начинаете видеть странные вещи в базе данных:

```python
>>> Book.objects.create(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     weight=0,
...     download_link='http://books.com/54321',
... )
```

Эта книга, очевидно, весит 0 фунтов и имеет ссылку на скачивание.

Эта электронная книга, по-видимому, весит 100 г и не имеет ссылки на скачивание:

```python
>>> Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='Python Tricks',
...     price=1000,
...     weight=100,
...     download_link=None,
... )
```

Это не имеет никакого смысла. Возникает проблема целостности данных.

Чтобы решить проблему целостности, в модель добавляются валидации:

```python
from django.core.exceptions import ValidationError


class Book(models.Model):

    # ...

    def clean(self) -> None:
        if self.type == Book.TYPE_VIRTUAL:
            if self.weight != 0:
                raise ValidationError(
                    'A virtual product weight cannot exceed zero.'
                )

            if self.download_link is None:
                raise ValidationError(
                    'A virtual product must have a download link.'
                )

        elif self.type == Book.TYPE_PHYSICAL:
            if self.weight == 0:
                raise ValidationError(
                    'A physical product weight must exceed zero.'
                )

            if self.download_link is not None:
                raise ValidationError(
                    'A physical product cannot have a download link.'
                )

        else:
            assert False, f'Unknown product type "{self.type}"'
```

Вы использовали [встроенный в Django механизм валидации](https://docs.djangoproject.com/en/2.1/ref/models/instances/#django.db.models.Model.clean) для обеспечения правил целостности данных. `clean()` вызывается автоматически только формами Django. Для объектов, которые не создаются Django-формой, необходимо убедиться в том, что объект явно валидирован.

Чтобы сохранить целостность модели Book, необходимо внести небольшое изменение в способ создания книг:

```python
>>> book = Book(
...    type=Book.TYPE_PHYSICAL,
...    name='Python Tricks',
...    price=1000,
...    weight=0,
...    download_link='http://books.com/54321',
... )
>>> book.full_clean()
ValidationError: {'__all__': ['A physical product weight must exceed zero.']}

>>> book = Book(
...    type=Book.TYPE_VIRTUAL,
...    name='Python Tricks',
...    price=1000,
...    weight=100,
...    download_link=None,
... )
>>> book.full_clean()
ValidationError: {'__all__': ['A virtual product weight cannot exceed zero.']}
```

При создании объектов с помощью менеджера по умолчанию (`Book.objects.create(...)`) Django создает объект и сразу же сохраняет его в базе данных.

В вашем случае необходимо проверить объект перед сохранением в базу данных. Сначала вы создаете объект (`Book(...)`), проверяете его (`book.full_clean()`) и только потом сохраняете (`book.save()`).

> Денормализация:
>
> Разреженная модель является продуктом [денормализации](https://en.wikipedia.org/wiki/Denormalization). В процессе денормализации атрибуты из нескольких нормализованных моделей объединяются в одну таблицу для повышения > производительности. Денормализованная таблица обычно содержит много нулевых столбцов.
> 
> Денормализация часто используется в системах поддержки принятия решений, таких как хранилища данных, где производительность чтения является наиболее важной. В отличие от [OLTP-систем](https://en.wikipedia.org/wiki/Online_transaction_processing), хранилища данных обычно не требуют соблюдения правил целостности данных, что делает денормализацию идеальным решением.

**За**

- **Легко понять и поддерживать:** Разреженная модель обычно является первым шагом, который мы делаем, когда определенные типы объектов нуждаются в дополнительной информации. Она очень интуитивна и проста для понимания.

**Против**

- **Невозможность использования ограничений базы данных NOT NULL:** Нулевые значения используются для атрибутов, которые определены не для всех типов объектов.

- **Сложная логика проверки:** Для обеспечения правил целостности данных требуется сложная логика проверки. Сложная логика также требует большего количества тестов.

- **Большое количество Null-полей создает беспорядок:** Представление нескольких типов продуктов в одной модели усложняет ее понимание и сопровождение.

- **Новые типы требуют изменения схемы:** Новые типы продуктов требуют дополнительных полей и валидаций.

**Пример использования**

Разрозненная модель идеально подходит для представления разнородных объектов, имеющих большинство общих атрибутов, и когда новые элементы добавляются нечасто.

# Полуструктурированная модель

Ваш книжный магазин пользуется большим успехом, и вы продаете все больше и больше книг. У вас есть книги разных жанров и издательств, электронные книги разных форматов, книги странных форм и размеров и т.д.

При использовании разреженной модели вы добавляли поля для каждого нового типа товара. Теперь в модели много нулевых полей, и новые разработчики и сотрудники с трудом справляются с этой задачей.

Чтобы устранить беспорядок, вы решили оставить в модели только общие поля (название и цена). Остальные поля хранятся в одном `JSONField`:

```python
from django.contrib.auth import get_user_model
from django.contrib.postgres.fields import JSONField
from django.db import models

class Book(models.Model):
    TYPE_PHYSICAL = 'physical'
    TYPE_VIRTUAL = 'virtual'
    TYPE_CHOICES = (
        (TYPE_PHYSICAL, 'Physical'),
        (TYPE_VIRTUAL, 'Virtual'),
    )
    type = models.CharField(
        max_length=20,
        choices=TYPE_CHOICES,
    )

    # Common attributes
    name = models.CharField(
        max_length=100,
    )
    price = models.PositiveIntegerField(
        help_text='in cents',
    )

    extra = JSONField()

    def __str__(self) -> str:
        return f'[{self.get_type_display()}] {self.name}'


class Cart(models.Model):
    user = models.OneToOneField(
        get_user_model(),
        primary_key=True,
        on_delete=models.CASCADE,
    )
    books = models.ManyToManyField(
        Book,
        related_name='+',
    )
```

> JSONField:
>
> В этом примере в качестве бэкенда базы данных используется PostgreSQL. Django предоставляет встроенное JSON-поле для PostgreSQL в django.contrib.postgres.fields.
>
> Для других баз данных, таких как [SQLite](https://realpython.com/python-sqlite-sqlalchemy/) и MySQL, существуют [пакеты](https://djangopackages.org/grids/g/json-fields/), предоставляющие аналогичную функциональность.

Теперь ваша модель книги не содержит беспорядка. Общие атрибуты моделируются в виде полей. Атрибуты, которые не являются общими для всех типов товаров, хранятся в `extra` JSON-поле:

```python
>>> from semi_structured.models import Book
>>> physical_book = Book(
...     type=Book.TYPE_PHYSICAL,
...     name='Python Tricks',
...     price=1000,
...     extra={'weight': 200},
... )
>>> physical_book.full_clean()
>>> physical_book.save()
<Book: [Physical] Python Tricks>

>>> virtual_book = Book(
...     type=Book.TYPE_VIRTUAL,
...     name='The Old Man and the Sea',
...     price=1500,
...     extra={'download_link': 'http://books.com/12345'},
... )
>>> virtual_book.full_clean()
>>> virtual_book.save()
<Book: [Virtual] The Old Man and the Sea>

>>> from semi_structured.models import Cart
>>> cart = Cart.objects.create(user=user)
>>> cart.books.add(physical_book, virtual_book)
>>> cart.books.all()
<QuerySet [<Book: [Physical] Python Tricks>, <Book: [Virtual] The Old Man and the Sea>]>
```

Устранение беспорядка - это важно, но за это приходится платить. Логика проверки становится намного сложнее:

```python
from django.core.exceptions import ValidationError
from django.core.validators import URLValidator

class Book(models.Model):

    # ...

    def clean(self) -> None:

        if self.type == Book.TYPE_VIRTUAL:

            try:
                weight = int(self.extra['weight'])
            except ValueError:
                raise ValidationError(
                    'Weight must be a number'
                )
            except KeyError:
                pass
            else:
                if weight != 0:
                    raise ValidationError(
                        'A virtual product weight cannot exceed zero.'
                    )

            try:
                download_link = self.extra['download_link']
            except KeyError:
                pass
            else:
                # Will raise a validation error
                URLValidator()(download_link)

        elif self.type == Book.TYPE_PHYSICAL:

            try:
                weight = int(self.extra['weight'])
            except ValueError:
                raise ValidationError(
                    'Weight must be a number'
                 )
            except KeyError:
                pass
            else:
                if weight == 0:
                    raise ValidationError(
                        'A physical product weight must exceed zero.'
                     )

            try:
                download_link = self.extra['download_link']
            except KeyError:
                pass
            else:
                if download_link is not None:
                    raise ValidationError(
                        'A physical product cannot have a download link.'
                    )

        else:
            raise ValidationError(f'Unknown product type "{self.type}"')
```

Преимущество использования правильного поля заключается в том, что оно подтверждает тип. И Django, и Django ORM могут выполнять проверки, чтобы убедиться, что для поля используется правильный тип. При использовании JSONField необходимо проверять как тип, так и значение:

```python
>>> book = Book.objects.create(
...     type=Book.TYPE_VIRTUAL,
...     name='Python Tricks',
...     price=1000,
...     extra={'weight': 100},
... )
>>> book.full_clean()
ValidationError: {'__all__': ['A virtual product weight cannot exceed zero.']}
```

Другой проблемой использования JSON является то, что не все базы данных имеют надлежащую поддержку запросов и индексирования значений в полях JSON.

Например, в PostgreSQL можно запросить все книги, вес которых превышает 100 экземпляров:

```python
>>> Book.objects.filter(extra__weight__gt=100)
<QuerySet [<Book: [Physical] Python Tricks>]>
```

Однако не все производители баз данных поддерживают эту возможность.

Еще одним ограничением, накладываемым при использовании JSON, является невозможность использования ограничений базы данных, таких как not null, unique и foreign keys. Эти ограничения необходимо реализовать в приложении.

Такой полуструктурированный подход напоминает архитектуру [NoSQL](https://en.wikipedia.org/wiki/NoSQL) и имеет много ее преимуществ и недостатков. Поле JSON - это способ обойти строгую схему реляционной базы данных. Этот гибридный подход дает нам возможность объединить множество типов объектов в одной таблице, сохраняя при этом некоторые преимущества реляционной, строго и жестко типизированной базы данных. Для многих распространенных случаев использования NoSQL такой подход может оказаться более подходящим.

**Плюсы**

- **Уменьшение беспорядка:** Общие поля хранятся в модели. Другие поля хранятся в одном JSON-поле.

- **Легче добавлять новые типы:** Новые типы продуктов не требуют изменения схемы.

**Минусы**

- **Сложная и нестандартная логика проверки:** Валидация JSON-поля требует проверки как типов, так и значений. Эту проблему можно решить, используя другие решения для проверки JSON-данных, например, JSON-схему.

- **Невозможность использования ограничений базы данных:** Ограничения базы данных, такие как null null, unique и ограничения внешнего ключа, которые обеспечивают целостность типов и данных на уровне базы данных, не могут быть использованы.

- **Ограничения, связанные с поддержкой JSON базами данных:** не все поставщики баз данных поддерживают запросы и индексирование полей JSON.

- **Схема не навязывается системой базы данных:** Изменения схемы могут потребовать обратной совместимости или специальных миграций. Данные могут "сгнить".

- **Нет глубокой интеграции с системой метаданных базы данных:** Метаданные о полях не хранятся в базе данных. Схема реализуется только на уровне приложения.

**Пример использования**

Полуструктурированная модель идеально подходит для представления разнородных объектов, не имеющих большого количества общих атрибутов, а также при частом добавлении новых элементов.

Классическим примером использования полуструктурированного подхода является хранение событий (например, журналов, аналитики и хранилищ событий). Большинство событий имеют временную метку, тип и метаданные, такие как устройство, пользовательский агент, пользователь и т.д. Данные для каждого типа хранятся в поле JSON. Для аналитики и журналов событий важно иметь возможность добавлять новые типы событий с минимальными усилиями, поэтому такой подход является идеальным.

# Абстрактная базовая модель

# Конкретная базовая модель

# Generic Foreign Key

# Заключение
