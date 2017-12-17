# meetup-python

Partie 1
========

Créer un environement virtuel
-----------------------------

``` bash
$ python3 -m venv hwaiohttp --without-pip
$ source hwaiohttp/bin/activate
(hwaiohttp) $ wget -qO - https://bootstrap.pypa.io/get-pip.py | python -
(hwaiohttp) $ pip install aiohttp jinja2 aiohttp_jinja2 aiohttp-session cryptography
```

Créer le repertoire du projet `hwaiohttp` qui contiendra le package `hwaiohttp`
-------------------------------------------------------------------------------

```
hwaiohttp
└── hwaiohttp
    ├── __init__.py
    ├── __main__.py
    └── views.py
```


Créer ça première vue dans le fichier `views.py` écrire
-------------------------------------------------------

``` python
import aiohttp
from aiohttp import web

async def hello(request):
    name = request.match_info.get('name', "Anonymous")
    text = "Hello, " + name
    return web.Response(text=text)
```


Importer la vue `hello` dans le `__main__.py` et la lier à une URL
------------------------------------------------------------------

``` python
from aiohttp import web

from .views import hello

app = web.Application()
app.router.add_get('/', hello)
app.router.add_get('/{name}', hello)
web.run_app(app)
```

Lancer son application
----------------------

$ python -m hwaiohttp

Se rendre à l'URL http://127.0.0.1:8080


Partie 2
========

Afficher le Hello depuis un template.

Créer un répertoire `templates` contenant un ficher `base.html`
---------------------------------------------------------------

```
hwaiohttp
└── hwaiohttp
    ├── __init__.py
    ├── __main__.py
    ├── templates
    │   └── base.html
    └── views.py
```

Dans le fichier `base.html` écrire ceci:


``` html
<!doctype html>
<html lang="fr">
<head>
    <meta charset="utf-8">
    <title>{% block title %}{% endblock %}</title>
    <script src="/static/chat.js"></script>
</head>
<body>
{% block content %}{% endblock %}
</body>
</html>
```

Créer un fichier `hello.html` qui va hériter de `base.html`
-----------------------------------------------------------


``` html
{% extends "base.html" %}
{% block title %}Hello {{ name }}{% endblock %}
{% block content %}
<h1>Hello {{ name }}</h1>
{% endblock %}
```

Modifier la vue `hello` dans le fichier `view.py`
-------------------------------------------------

``` python
@aiohttp_jinja2.template('hello.html')
async def hello(request):
    name = request.match_info.get('name', 'Anonymous')
    return {'name': name}
```

Et importer le module `aiohttp_jinja2`.

``` python
import aiohttp
from aiohttp import web
import aiohttp_jinja2
``` 


Configurer l'application pour jinja 
-----------------------------------

Dans le `__main__.py`, ajouter l'instruction `aiohttp_jinja2.setup` et 
importer les modules `aiohttp_jinja2`, `jinja2` ainsi que la class `Path` du module `pathlib`


``` python
from pathlib import Path

from aiohttp import web
import aiohttp_jinja2
import jinja2

from .views import hello


app = web.Application()
app.router.add_get('/', hello)
app.router.add_get('/{name}', hello)

aiohttp_jinja2.setup(
    app, loader=jinja2.FileSystemLoader(
        str(Path(__file__).parent / 'templates')))

web.run_app(app)
```

Relancer le server et vérifier le résultat.


Partie 3
========

Créer un répertoire `static`  
----------------------------

Ajouter dedans le fichier `chat.js`

``` js
var socket = null;

function start(wsUrlPath, inputMessageId, messageDisplayerId) {
    inputMessage = document.getElementById(inputMessageId);
    messageDisplayer = document.getElementById(messageDisplayerId);

    inputMessage.onkeypress = function(event) {
        if (event.keyCode === 13) {
            socket.send(inputMessage.value);
            inputMessage.value = '';
            return false;
        }
        else {
            return true;
        }
    }

    try {
        socket = new WebSocket("ws://" + location.host  + wsUrlPath);
    } catch (exception) {
        console.error(exception);
    }

    socket.onerror = function(error) {
        console.error(error);
    };

    socket.onopen = function(event) {
        console.log("Connexion établie.");

        this.onclose = function(event) {
            console.log("Connexion terminé.");
        };

        this.onmessage = function(event) {
            console.log("Message:", event.data);
            messageDisplayer.textContent += event.data + '\n';
        };
    };
};
```

Ajouter la route vers le répertoire static
------------------------------------------

``` python
app.router.add_static('/static/',
                      path=str(Path(__file__).parents[1] / 'static'),
                      name='static')

```


Ajouter le template `chat.html`
-------------------------------

``` html
{% extends "base.html" %}
{% block title %}Chat{% endblock %}
{% block content %}
<h1>Chat</h1>
<div>
    <textarea id="received" rows="20" cols="50" readonly></textarea>
</div>
<div>
    <textarea id="send" rows="4" cols="50"></textarea>
</div>
<script>start("{{ ws_url }}", "send", "received");</script>
{% endblock %}
```


Ajouter la class `ChatHandler` dans view.py
-------------------------------------------

``` python
class ChatHandler:

    @aiohttp_jinja2.template('chat.html')
    async def chat(self, request):
        return {'ws_url': '/ws-chat'}

    async def ws_chat(self, request):
        ws = web.WebSocketResponse()
        await ws.prepare(request)

        async for msg in ws:
            if msg.type == aiohttp.WSMsgType.TEXT:
                await ws.send_str('echo: ' + msg.data)

            elif msg.type == aiohttp.WSMsgType.ERROR:
                print('ws connection closed with exception {}'.
                      format(ws.exception()))

        print('websocket connection closed')
        return ws
```

Importer `ChatHandler` dans le `__main__.py`, instatier le, et 
lier ses méthodes à des urls.
 

``` python
from pathlib import Path

from aiohttp import web
import aiohttp_jinja2
import jinja2

from .views import ChatHandler, hello

chat_handler = ChatHandler()

app = web.Application()
# app.router.add_get('/', hello)
# app.router.add_get('/{name}', hello)
app.router.add_get('/chat', chat_handler.chat)
app.router.add_route('GET', '/ws-chat', chat_handler.ws_chat)
app.router.add_static('/static/',
                      path=str(Path(__file__).parents[1] / 'static'),
                      name='static')

aiohttp_jinja2.setup(
    app, loader=jinja2.FileSystemLoader(
        str(Path(__file__).parent / 'templates')))

web.run_app(app)
```

Redémarrer le serveur et se rendre à l'URL http://127.0.0.1:8080/chat 

Partie 4
========

Ajouter un constructer à la classe `ChatHandler`:

``` python
class ChatHandler:

    def __init__(self):
        self.users = {}
```


Dans la méthode `ChatHandler.chat`, stoquer le nom de l'utilisateur dans la session.

``` python
    @aiohttp_jinja2.template('chat.html')
    async def chat(self, request):
        user_name = request.match_info['user_name']
        session = await aiohttp_session.get_session(request)
        session['user_name'] = user_name
        return {'ws_url': '/ws-chat'} 
```


Puis modifier la méthode `ws_chat` pour stoquer la connection websocket de chaque user.

``` python
    async def ws_chat(self, request):
        ws = web.WebSocketResponse()
        await ws.prepare(request)
        session = await aiohttp_session.get_session(request)
        user_name = session['user_name']
        self.users[user_name] = ws

        async for msg in ws:
            if msg.type == aiohttp.WSMsgType.TEXT:
                msg_with_author = '{}: {}'.format(user_name, msg.data)
                for user in self.users.values():
                    await user.send_str(msg_with_author)

            elif msg.type == aiohttp.WSMsgType.ERROR:
                print('ws connection closed with exception {}'.
                      format(ws.exception()))

        print('websocket connection closed')
        return ws
``` 

Importer le module `aiohttp_session` dans `views.py`

``` python
import aiohttp
from aiohttp import web
import aiohttp_session
import aiohttp_jinja2
```

Modifier l'URL liée à la méthode `chat` pour qu'elle puisse prendre un nom en paramètre.

``` python
app.router.add_get('/chat/{user_name}', chat_handler.chat)
```

importer la classe `Fernet` du module cryptography,
le classe `EncryptedCookieStorage` du module `aiohttp_session.cookie_storage`
ainsi que le module `base64` et `aiohttp_session` dans le `__main__.py`


``` python
import base64
from pathlib import Path

from aiohttp import web
import aiohttp_jinja2
import aiohttp_session
from aiohttp_session.cookie_storage import EncryptedCookieStorage
from cryptography.fernet import Fernet  # py36 import secrets
import jinja2

from .views import ChatHandler, hello
```

Initialiser le midelware de gestion des Cookies

```
# py36 secret_key = secrets.token_bytes(32)
fernet_key = Fernet.generate_key()
secret_key = base64.urlsafe_b64decode(fernet_key)

aiohttp_session.setup(app, EncryptedCookieStorage(secret_key))
```

Redemarrer le serveur et lancer deux navigateurs en navigation privées l'un vers l'URL

http://127.0.0.1:8080/chat/toto

Et l'autre vers l'URL
http://127.0.0.1:8080/chat/tutu

