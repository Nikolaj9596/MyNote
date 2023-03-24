## Pathlib

Чтобы создать путь с помощью Pathlib, нужно импортировать класс Path и передать ему строку. 
Эта строка указывает на путь в файловой системе, который не обязательно должен существовать.

```python
from pathlib import Path

path = Path("/Users/ahmed.besbes/projects/posts")

path

# PosixPath('/Users/ahmed.besbes/projects/posts')

print(cwd)

# /Users/ahmed.besbes/projects/posts
```

Теперь у вас есть доступ к объекту Path. Как выполнять простые операции?

### Объединение путей

Pathlib использует оператор / для объединения путей. 
Сначала эта функция может показаться забавной, но на самом деле она облегчает чтение кода.

```python
from pathlib import Path
 
in_file = Path.cwd() / "raw_data" / "input.xlsx"
out_file = Path.cwd() / "processed_data" / "output.xlsx"
```

## Получение текущего рабочего/ домашнего каталога

Под эту задачу уже предусмотрены методы:

```python
from path import Pathlib

cwd = Path.cwd()
home = Path.home()
```
Вы можете либо использовать open с контекстным менеджером,
как это делается в случае обычного пути,
либо использовать read_text или read_bytes.

```python
>>> path = pathlib.Path.home() / file.txt'
>>> path.read_text()
```

## Простое создание файлов и каталогов

Вот как объект Path создает папку:
```python

from pathlib import Path

random_folder = Path("random_folder")

random_folder.exists()
# False

random_folder.mkdir(exist_ok=True)

random_folder.exists()
# True
```

Тот же принцип используется и при создании файлов:
```python
from pathlib import Path

random_file = Path("random_file.txt")

random_file.exists()
# False

random_file.touch()

random_file.exists()
# True
```
## Перемещение по иерархии файловой системы с помощью обращения к родителям

Каждый объект Path имеет свойство parent, которое возвращает объект Path родительской папки.
Это облегчает работу с большими иерархиями папок.
Поскольку Path являются объектами, для доступа к нужному родителю можно использовать цепочку методов:

```python
from pathlib import Path

path = Path("/Users/ahmed.besbes/Downloads/")

path
# PosixPath('/Users/ahmed.besbes/Downloads')

path.parent
# PosixPath('/Users/ahmed.besbes')

path.parent.parent
# PosixPath('/Users')

path.parent.parent.parent
# PosixPath('/')
```

Чтобы избежать построения цепочки свойств parent для доступа к n-му предыдущему родителю,
можно вызвать свойство parents, которое возвращает список всех родителей,
предшествующих текущей папке:
```python
import pathlib

path = Path("/Users/ahmed.besbes/Downloads/")

list(path.parents)
# [PosixPath('/Users/ahmed.besbes'), PosixPath('/Users'), PosixPath('/')]
```

## Выполнение итераций в каталогах и подгонка под шаблон

Допустим, нужно подсчитать количество файлов Python в конкретной папке. Вот как это можно сделать:
```python
from pathlib import Path

# Действительно, довольно большая папка!
path = Path("/Users/ahmed.besbes/anaconda3/")

python_files = path.rglob("**/*.py")

next(python_files)
# PosixPath('/Users/ahmed.besbes/anaconda3/bin/rst2xetex.py')

next(python_files)
# PosixPath('/Users/ahmed.besbes/anaconda3/bin/rst2latex.py')

next(python_files)
# PosixPath('/Users/ahmed.besbes/anaconda3/bin/rst2odt_prepstyles.py')

...

len(list(python_files))
# 67481
```
## Каждый объект Path имеет множество полезных атрибутов

Каждый объект Path имеет множество полезных методов и атрибутов,
которые совершают операции, ранее выполнявшиеся другими библиотеками,
а не os (например, glob и shutil).

* .exists(): проверка наличия пути в файловой системе.
* .is_dir(): проверка соответствия пути каталогу.
* .is_file(): проверка соответствия пути файлу.
* .is_absolute(): проверка абсолютности пути.
* .chmod(): изменение режима файла и разрешений.
* .is_mount() : проверка того, является ли путь точкой монтирования.
* .suffix : получение расширение файла.

Это еще не все методы. Вы можете ознакомиться с полным их перечнем [здесь](https://docs.python.org/3/library/pathlib.html).

