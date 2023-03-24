## Как работате обычный dataclass и какие у него недостатки ?

Для примера напишем несколько простых классов

%%(python)
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Reader:
    id: int 
    message_read_time: datetime

@dataclass
class Message:
    id: int 
    text: str
    author: int
    readers: list[Reader]

@dataclass
class Chat:
    id: int
    chat_name: str
    messages: list[Message]
  
%%

Сгенирируем данные и обернем в класс `Chat.`
%%(python)
data = {
	"id": 1,
	"chat_name": "Test chat",
	"messages": [
		{
	           "id": 1,
		   "text": "Test text",
		   "author": 1,
	           "readers": [
			{
			   "id": 1,
			   "message_read_time": "2022-07-17T10:00:00"
			},	
			{
			   "id": 2,
			   "message_read_time": "2022-07-17T13:00:00"
			}
                   ]
 		},
		{
		   "id": 2,
		   "text": "Test text 2",
		   "author": 1,
		   "readers": [
			{
			   "id": 1,
			   "message_read_time": "2022-07-17T14:00:00"
			}	
		   ]
	       }
         ]
}

chat = Chat(**data)

#Посмотрим что получилось
print(chat)
"""
Chat(
   id=1, 
  chat_name='Test chat', 
  messages=[
     {'id': 1,
      'text': 'Test text', 
      'author': 1, 
      'readers': [{'id': 1, 'message_read_time': '2022-07-17T10:00:00'}, {'id': 2, 'message_read_time': '2022-07-17T13:00:00'}]
     }, 
    {'id': 2, 
     'text': 'Test text 2', 
     'author': 1, 
     'readers': [{'id': 1, 'message_read_time': '2022-07-17T14:00:00'}]
     }
   ]
)
"""
%%

Как мы можем видеть вложенные данные (`messages`, `readers`) не были обернуты в кслассы
Добавляем новый декаратор

%%(python)
from dataclasses import dataclass, is_dataclass
from typing import Any


def nested_dataclass(*args, **kwargs):
    """Improved decorator for dataclasses"""
    def wrapper(cls):
        cls = dataclass(cls, **kwargs)
        original_init = cls.__init__
        def __init__(self, *args, **kwargs):
            for name, value in kwargs.items():
                field_type = cls.__annotations__.get(name, None)
                if is_dataclass(field_type) and isinstance(value, dict):
                     new_obj = field_type(**value)
                     kwargs[name] = new_obj
                if isinstance(value, list):
                    list_type = field_type.__args__[0]
                    new_obj = [list_type(**v) for v in value]
                    kwargs[name] = new_obj
            original_init(self, *args, **kwargs)
        cls.__init__ = __init__
        return cls
    return wrapper(args[0]) if args else wrapper
%%

Оборачиваем классы которые имеют вложенные данные в новый декоратор
%%(python)
@nested_dataclass
class Message:
    id: int 
    text: str
    author: int
    readers: list[Reader]

@nested_dataclass
class Chat:
    id: int
    chat_name: str
    messages: list[Message]
%%

Снова обернем наши данные в обновленный класс `Chat.`

%%(python)
chat = Chat(**data)

#Посмотрим что получилось
print(chat)
"""
Chat(id=1, 
     chat_name='Test chat', 
     messages=[
         Message(id=1,
                text='Test text', 
                author=1, 
                readers=[
                    Reader(id=1, message_read_time='2022-07-17T10:00:00'), 
                    Reader(id=2, message_read_time='2022-07-17T13:00:00')
                ]
         ), 
         Message(id=2,
                text='Test text 2', 
                author=1, 
                readers=[Reader(id=1, message_read_time='2022-07-17T14:00:00')]
         )
     ]
  )
"""
%%
Как видем все вложенные данные были обернуты в классы
