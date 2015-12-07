+++
date = "2015-01-12"
title = "Готовим Django: uWSGI + Nginx"

aliases = [
    "django-uwsgi-nginx/"
]

+++

На просторах интернета есть масса статей о том, как деплоить Django, используя различные серверы приложений. Статьи хороши, но все же при деплои  этого блога (изначально блог работал на Django), ни одна из них не ответила полностью на все вопросы. Я хочу поделиться тем, как это сделал я. Упор будем делать на простоту.

Итак допустим, что у вас уже есть готовое Django-приложение на вашей локальной машине, файл `requirements.txt` с прописанными в нем зависимостями (используемыми библиотеками), арендованный VPS и что вы умеете работать с консолью.

## Первоначальные настройки

Первым делом, зайдем на сервер и обновим пакеты:
```
ssh user@server
sudo apt-get update
sudo apt-get upgrade
```

Установим Nginx:
```
sudo apt-get install nginx
```

Это очень быстрый и легкий прокси-сервер. Он будет раздавать статику вашего сайта и взаимодействовать с сервером приложений. Сервером приложений будет выступать uWSGI. Именно он будет запускать Django-приложение.

Установим uWSGI:
```
sudo apt-get install python3-pip # менеджер пакетов для Python 3
sudo pip install uwsgi
```

Установим утилиту для создания виртуального окружения:
```
sudo apt-get install python-virtualenv
```

Ну, и не будем же мы заливать приложения на сервер по ftp, в самом деле? Так что установим git:

```
sudo apt-get install git
# минимальная настройка:
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
```

## Настроим БД
Как БД будем использовать PostgreSQL (по рекомендации разработчиков Django):
```
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
```
Сменим пользователя системы:
```
sudo su - postgres
```

Создадим БД для нашего сайта:
```
createdb your_blog_db
```
Создадим пользователя БД:
```
createuser -P your_db_user
```
Входим в консольный интерфейс PostgreSQL:
```
psql
```
и даем нашему пользователю права на работу с нашей базой:
```
GRANT ALL PRIVILEGES ON DATABASE yout_blog_db TO your_db_user;
```
Выйдем из консольного интерфейса PostgreSQL и вернемся к первоначальному пользователю системы: для этого 2 раза нажмем `Ctrl + D`.

## Зальем приложение на сервер

Первым делом необходимо будет сделать некоторые правки нашего приложения. 

Настроить новую БД в файле settings.py вашего проекта:
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'your_blog_db',
        'USER': 'your_db_user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```
Выключим режим отладки:
```
DEBUG = False
```
**Примечание**: это упрощенный пример и лучше разделять настройки проекта для окружений разработки и реальной работы - [раз](https://code.djangoproject.com/wiki/SplitSettings), [два](http://habrahabr.ru/post/104686/).

Теперь, если ваше приложение еще не добавлено в git-репозиторий, самое время сделать это (помним про [gitignore](http://git-scm.com/docs/gitignore)):


```
# корневой каталог вашего приложения
git init
git add * 
git commit -m 'initial'
```

Далее есть минимум 2 способа:

1. Красивый и правильный, но надо немного больше повозиться - тыц.
2. Как делал я. Мне он был более удобен, так как у меня уже был аккаунт в облачном git-хостинге (называйте как хотите).  Расскажу подробнее.

Регистрируетесь на [bitbucket.org](https://bitbucket.org/), создаете новый приватный git-репозитоий и копируете ссылку на него. Это будет ваш удаленный репозиторий. Далее добавляете ссылку на уделенный репозиторий в локальный репозиторий:
```
git remote add origin https://your_user@bitbucket.org/your_user/your_repository.git
```
Отправляем код в удаленный репозиторий:
```
git push
```
Снова заходим по ssh на наш сервер и переходим в каталог, в котором хотим разместить приложение.  И так, копируем наше приложение:
```
git clone https://your_user@bitbucket.org/your_user/your_repository.git
```
Все, теперь ваш код на рабочем сервере. Создадим виртуальное окружение и установим в него необходимые модули:
```
cd /path/to/project  # переходим в корень вашего приложения
virtualenv -p /usr/bin/python3 venv  # создаем виртуальное окружение venv c Python 3 
. venv/bin/activate  # активируем его
pip install -r requirements.txt  # устанавливаем зависимости
pip install psycopg2  # библиотека для работы Django с PostgreSQL
```
Убедитесь, что приложение запускается:
```
python3 manage.py runserver
```
Теперь ваш код почти говтов к работе.

## Конфигурация Nginx и  uWSGI

Скопируйте файл https://github.com/nginx/nginx/blob/master/conf/uwsgi_params в корень вашего проекта. 

Необходимо настроить конфигурации Nginx, uWSGI и настроить связь между ними. Обмениваться данными они будут через сокет-файл.

Создадим конфигурацию Nginx:
```
# /path/to/project/nginx.conf
upstream django {
    server unix:///path/to/project/blog.sock;  # путь к сокет-файлу
}

server {
    # the port your site will be served on
    listen      80;
    # the domain name it will serve for
    server_name ***.***.***.***;  # IP вашего сервера
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;

    # Django media
    location /media  {
        alias /path/to/project/media;  # путь к вашим медиа-файлам
    }

    location /static {
        alias /path/to/project/static; # путь к статическим файлам вашего сайта
    }

    location / {
        uwsgi_pass  django;
        include     /path/to/project/uwsgi_params; # путь к  uwsgi_params
    }
}
```

Создаем ссылку на нашу конфигурацию, что бы Nginx знал о ней. Затем перезапускаем Nginx:
```
ln -s /path/to/project/nginx.conf /etc/nginx/sites-enabled/
service nginx restart
```
Создадим конфигурацию uWSGI:

```
# uwsgi.ini
[uwsgi]

chdir           = /path/to/project  # путь к вашему проекту
module          = your_site.wsgi:application  # your_site - главный модуль вашего приложения
home            = /path/to/project/venv  # путь к вашему виртуальному окружению
master          = True
processes       = 4
socket          = /path/to/projec/blog.sock  # путь к сокет-файлу
# Если у Nginx не хватит прав для работы с сокет-файлом, включим chmod-socket, сначала пробуем 664, потом 666
# chmod-socket    = 664
vacuum          = True
uid             = www-data
gid             = www-data
daemonize = /var/log/uwsgi/app/blog.ini  # пишем логи в этот файл
```
Теперь мы можем запустить сервер приложений:
```
uwsgi --ini uwsgi.ini
```
И так, проверяйте, все должно работать! 

Последним штрихом будет добавление автоматического старта uWSGI при перезагрузке сервера:
```
# Редактируем файл /etc/rc.local
/usr/local/bin/uwsgi --ini /path/to/project/uwsgi.ini
exit 0
```

## Послесловие
Обязательно найдется тот, кто спросит: почему uWSGI, а не Gunicorn? Ответ: это не был окончательный выбор в пользу uWSGI, в дальнейшем я хочу сравнить его с Gunicorn.

Надеюсь данное руководство хоть кому-то будет полезным. Спасибо всем, кто прочитал до конца!
P.S. Буду рад вопросам, идеям и замечаниям в комментариях и подписчикам в моем [мини-блоге](https://twitter.com/_alxpy).