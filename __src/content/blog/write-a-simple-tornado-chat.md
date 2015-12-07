+++
date = "2015-01-30"
title = "Пишем простой чат на Tornado"

aliases = [
    "write-a-simple-tornado-chat/"
]

+++

В этой статье я попытаюсь рассказать о том, как написать очень простой чат на Tornado с использованием протокола WebSocket.

> Tornado — расширяемый, неблокирующий веб-сервер и фреймворк, написанный на Python.

И так, начнем. Первым делом Вам необходимо установить Tornado (собственно, больше ничего устанавливать и не нужно):
```
$ mkdir chat
$ cd chat/
$ virtualenv -p /usr/bin/python3 venv
$ . venv/bin/activate
$ pip install tornado
```

Всё, все предварительные настройки готовы и мы можем начать писать код.

Нам понадобятся седующие модули:

```
import os.path
import tornado.ioloop
import tornado.web
import tornado.websocket
import tornado.auth
import tornado.gen
```

Что делает каждый из них, будет понятно дальше. 

## Конфигурация приложения
Создадим класс, описывающий наше приложение:
```
class Application(tornado.web.Application):
    def __init__(self):
        handlers = [
            (r'/', MainHandler),
            (r'/chat', ChatHandler),
            (r'/ws', WebSocketHandler),
            (r'/login', LoginHandler),
            (r'/logout', LogoutHandler),
        ]
        settings = dict(
            cookie_secret="your_cookie_secret",
            template_path=os.path.join(os.path.dirname(__file__), 'templates'),
            static_path=os.path.join(os.path.dirname(__file__), 'static'),
            twitter_consumer_key='your_twitter_consumer_key',
            twitter_consumer_secret='your_twitter_consumer_secret',
            login_url='/',
            xsrf_cookies=True,
            debug=True,
        )
        tornado.web.Application.__init__(self, handlers, **settings)
```

В терминологии Django `handlers` (обработчик) это аналог `URLconf`, он указывает по какому адресу будет вызываться `handler` (`view` в Django). 

`login_url` - адрес на который будет перенаправлен неизвестный пользователь, если попытается зайти на страницу, для которой нужна авторизация. 

В Tornado есть встроенная поддержка аутентификации и авторизации через Google, Facebook, Twitter и FriendFeed. Мы будем использовать Twitter. Для этого нам необходимо зарегистрировать свое приложение [тут](https://apps.twitter.com/app/new). При создании приложения, необходимо указать `Callback URL` - адрес на который будет перенапрвален пользователь после авторизации (в моем случае это 127.0.0.1:5000/login). После регистрации укажите в коде `twitter_consumer_key` и `twitter_consumer_secret`,   эти данные будут подставленны в запрос на сервер твиттера при авторизации. 

`template_path` - путь к шаблонам, `static_path` - путь к статическим файлам (js, css). В этой статье я не буду рассказывать о шаблонизаторе Tornado.

`cookie_secret` - ключ для шифровки cookies, без него cookies можно было бы легко подделать. `xsrf_cookies=True` - включаем защиту от [межсайтовой подделки запроса](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%B6%D1%81%D0%B0%D0%B9%D1%82%D0%BE%D0%B2%D0%B0%D1%8F_%D0%BF%D0%BE%D0%B4%D0%B4%D0%B5%D0%BB%D0%BA%D0%B0_%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B0).

Во время разработки Вы постоянно правите код и, что-бы после каждой правки не перезапускать Ваше приложение вручную, можно указать Tornado делать это автоматически - `debug=True`.

## Создание обработчиков
Далее мы создадим базовый обработчик (handler):
```
class BaseHandler(tornado.web.RequestHandler):
    def get_current_user(self):
        user = self.get_secure_cookie('username')
        if user:
            user = user.decode('utf-8')
        return user
```
Что бы создать обработчик, необходимо наследоваться от `tornado.web.RequestHandler`. Для определения текущего пользователя (current_user) служит `get_current_user()`.

Обработчик главной страницы:
```
class MainHandler(BaseHandler):
    def get(self):
        if self.current_user:
            self.redirect('/chat')
        self.render('index.html')
```
Наследуемся от базового обработчика. Легко понять, что при GET-запросе (а каждый обработчик может обрабатывать одновременно GET/HEAD/POST запросы), в зависивости от того, вошел ли пользователь в систему или нет, ему либо будет показана страница входа/регистрации (index.html), либо он будет перенаправлен на страницу чата.

При этом просто так зайти на страницу чата нельзя:
```
class ChatHandler(BaseHandler):
    @tornado.web.authenticated
    def get(self):
        self.render('chat.html', user=self.current_user)
```
`@tornado.web.authenticated` "пускает" на страницу только "вошедших" пользователей, а остальных перенаправляет на `login_url`.

Далее создадим обработчик для входа:
```
class LoginHandler(tornado.web.RequestHandler, tornado.auth.TwitterMixin):
    @tornado.gen.coroutine
    def get(self):
        if self.get_argument("oauth_token", None):
            user = yield self.get_authenticated_user()
            self.set_secure_cookie('username', str(user['username']))
            self.redirect("/chat")
        else:
            yield self.authorize_redirect()
```
Дополнительно наследуемся от `tornado.auth.TwitterMixin`, который служит для OAuth-аутентификации через Твиттер. При первом вызове обработчика он перенаправит нас на страницу Твиттера, а после авторизации Твиттер снова "перекинет"  нас на этот обработчик. Если запрос Твиттера содержал `oauth_token`, то мы создаем пользователя, устанавливаем cookie с его именем и делаем "редирект" на страницу чата. В противном случае, пользователя перекинет на страницу с формой/кнопкой входа. 

`@tornado.gen.coroutine` - декоратор для асинхронных генераторов. Про декораторы и генераторы позже будет в отдельных статьх.

В нашем случае, для выхода нужно всего лишь очистить cookies:
```
class LogoutHandler(tornado.web.RequestHandler):
    def get(self):
        self.clear_all_cookies()
        self.redirect("/")
```

## Работа с WebSocket
> WebSocket — протокол полнодуплексной связи поверх TCP-соединения, предназначенный для обмена сообщениями между браузером и веб-сервером в режиме реального времени.

Если очень просто, то мы создаем канал между клиентом и сервером, держим его постоянно открытым и обмениваемся сообшениями. Создать канал очень просто:

```
var ws = new WebSocket("ws://127.0.0.1:5000/ws");
```
Теперь для отправки сообщений мы вызываем `ws.send(msg)`, где msg - переменная с текстом сообщения. `ws.onmessage = function (evt) { ... }` - функция, которая обрабатывает входящие собщения.

На стороне сервера с WebSocket работает обработчик наследованный от BaseHandler и `tornado.websocket.WebSocketHandler`:
```
class WebSocketHandler(BaseHandler, tornado.websocket.WebSocketHandler):
    connections = set()

    def open(self):
        WebSocketHandler.connections.add(self)

    def on_close(self):
        WebSocketHandler.connections.remove(self)

    def on_message(self, msg):
        self.send_messages(msg)

    def send_messages(self, msg):
        for conn in self.connections:
            conn.write_message({'name': self.current_user, 'msg': msg})
```
`open` и `on_close` пополняют список соединений (connections) новым каналом и удалют канал из списка при его закрытии. `on_message` - реагирует на входящее сообщение, `write_message()` отправляет сообщение указанному соединению (conn). Если вдруг можно отослать сообщение сразу по всем каналом, подскажите, пожалуйста, в комментариях как это сделать.

## Запуск приложения
Тут все просто: создаем объект нашего приложения и говорим какой порт ему слушать (если в окружении нет параметра 'PORT', то 5000й):
```
def main():
    port = int(os.environ.get("PORT", 5000))
    app = Application()
    app.listen(port)
    tornado.ioloop.IOLoop.instance().start()

if __name__ == '__main__':
    main()
```
Создаем экземпляр основной петли ввода/вывода `tornado.ioloop.IOLoop` и запускаем его.

## Ссылки

Полный код - https://github.com/alxpy/simple-tornado-chat/tree/0.2 

Документация по Tornado - http://www.tornadoweb.org/en/stable/

## Послесловие
Получилось сумбурно, но надеюсь в целом понятно. Если есть дополнительные вопросы, задавайте их в комментариях и я постараюсь ответить.

P.S. Буду рад вопросам, идеям и замечаниям в комментариях и подписчикам в моем [мини-блоге](https://twitter.com/_alxpy).