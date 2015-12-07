+++
date = "2015-05-03"
title = "Собственные типы полей в SQLAlchemy"

aliases = [
    "custom-field-types-in-sqlalchemy/"
]

+++

Часто возникает необходимость хранить в БД статус модели. При этом необходимо оптимально хранить статус в базе и удобно работать с ним в коде. Я покажу как это сделать с использованием собственных типов полей в SQLAlchemy.

Допустим у нас есть простая модель юзера:
```
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(80))

    def __init__(self, name):
        self.name = name

    def __repr__(self):
       return '<User: {}>'.format(self.name)
```
Правильный способ решения задачи это хранить в базе числовое значение. Если просто запомнить какое число отвечает какому статусу, то в коде мгновенно начнут появляться ошибки. Использование констант решает это проблему (`STATUS_ACTIVE = 0`, `user.status = user.STATUS_ACTIVE`), но такое решение все равно не достаточно красиво и заставляет создавать отдельные словари для хранения текстового отображения статусов. В Python 3.4 появились удобные типы [Emun/IntEnum](https://docs.python.org/3/library/enum.html), которые мы будем использовать для описания статусов. Создадим атрибут модели:
```
STATUS = IntEnum('UserStatus', {
    'active': 0,
    'deleted': 1,
})
```
Для наглядности покажу как работать с Enum:
```
>>> STATUS.active
<UserStatus.active: 0>
>>> STATUS.active.value
0
>>> STATUS.active.name
'active'
>>> STATUS(1)
<UserStatus.deleted: 1>
>>> STATUS(1).name
'deleted'
```
В SQLAlchemy есть специальный тип `TypeDecorator`, на основе которого можно создать собственный тип поля:
```
class EnumType(TypeDecorator):
    impl = SmallInteger

    def __init__(self, enum, *args, **kwargs):
        self._enum = enum
        TypeDecorator.__init__(self, *args, **kwargs)

    def process_bind_param(self, enum, dialect):
        return enum.value

    def process_result_value(self, value, dialect):
        return self._enum(value)
```
Создадим у юзера поля status, созданного нами, типа `EnumType`:
```
class User(Base):
    __tablename__ = 'users'

    STATUS = IntEnum('UserStatus', {
        'active': 0,
        'deleted': 1,
    })

    id = Column(Integer, primary_key=True)
    name = Column(String(80))
    status = Column(EnumType(STATUS), default=STATUS.active)

    ...
```
Работать это будет таким образом: реальное поле в базе будет типа `SmallInteger`.  Когда мы присваиваем статусу юзера значение `STATUS.active`, метод `process_bind_param()` будет писать в базу `STATUS.active.value`, а при чтении из базы метод `process_result_value()` буде возвращать `STATUS.active` (мне так было удобнее, можно возвращать и `STATUS.active.name`). Это, собственно, все.

P.S. Буду рад вопросам, идеям и замечаниям в комментариях и подписчикам в моем [мини-блоге](https://twitter.com/_alxpy).