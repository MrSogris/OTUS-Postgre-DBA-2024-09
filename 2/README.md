# Задание 2
## Установка и настройка PostgteSQL в контейнере Docker
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Развертывание контейнера с СУБД
Создаем новый контейнер, указываем порт 5432 как внешний и также мапим директорию хост-машины в контейнер:
'''
docker run -d -p 5432:5432 -v D:\psql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres:15.8
'''
> [!CAUTION]
> Здесь я столкнулся с проблемой, если мапить директорию как в тексте ДЗ, то в ней синхронизируется только папка data, но не ее содержимое. Причину этого я к сожалению так и не нашел. Любые другие папки работали замечательно, возможно проблема в том что хост-машина - Win. В итоге нашел вариант мапить сразу директорию data.

Далее, поднимаем контейнер с клиентом
'''
docker run -it --rm jbergknoff/postgresql-client postgresql://postgres:password@192.168.88.126:5432
'''
**192.168.88.126** - адрес моей рабочей машины.
Т.к. запустили в интерактивном режиме, сразу проваливаемся в клиент psql и делаем тестовый набор данных:
'''
postgres=# CREATE TABLE testTable (code char(5) CONSTRAINT testTablePk PRIMARY KEY, name varchar(255) NOT NULL);
postgres=# INSERT INTO testTable VALUES('code1', 'name1');
postgres=# SELECT * FROM testTable;
 code  | name
-------+-------
 code1 | name1
(1 row)
'''
Далее на другом ноутбуке запускам DBeaver и проверяем соединение:
![Screenshot of a comment on a GitHub issue showing an image, added in the Markdown, of an Octocat smiling and raising a tentacle.](https://myoctocat.com/assets/images/base-octocat.svg)
> [!IMPORTANT]
> Таким образом, имеем вполне рабочий контейнер с СУБД, к нему есть доступы извне.

Завершаем работу контейнера с клиентом, удаляем контейнер с сервером. В директории *D:\psql\data* наблюдаем кучку папок и файлов оставшихся от работы СУБД. Теперь создаем новый контейнер той же командой что и ранее:
'''
docker run -d -p 5432:5432 -v D:\psql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres:15.8
docker run -it --rm jbergknoff/postgresql-client postgresql://postgres:password@192.168.88.126:5432
'''
В клиенте убеждаемся что данные успешно примонтировались и наша таблица на месте:
'''
postgres=# SELECT * FROM testTable;
 code  | name
-------+-------
 code1 | name1
(1 row)
'''
> [!IMPORTANT]
> Убедились, что данные СУБД лежат в указанной директории и ее переносом можно по сути "воскрешать" сервера.
