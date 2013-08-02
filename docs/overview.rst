.. currentmodule:: tornado.web

概述
========

`FriendFeed <http://friendfeed.com/>`_ 使用了一款使用Python编写的, 相对简单的非阻塞式Web服务器. 其应用程序使用的Web框架看起来有些像
`web.py <http://webpy.org/>`_
或 Google的
`webapp <http://code.google.com/appengine/docs/python/tools/webapp/>`_ , 不过为了能利用非阻塞式Web服务器和工具，这个Web框架还包含了一些额外的工具和优化。

`Tornado <https://github.com/facebook/tornado>`_ 就是这个Web服务器以及我们在FriendFeed中最常用使用的工具的开源版本. 这个框架和现在的主流Web服务器框架(包括大多数Python框架)有着明显的区别：因为它是非阻塞式的，并且速度相当快。因为它是非阻塞式的, 并且运用
`epoll <http://www.kernel.org/doc/man-pages/online/pages/man4/epoll.4.html>`_ 或kqueue, 它可以同时处理数千同时发生的标准连接, 这意味着这个框架对于实时的web服务是个理想的选择. 我们开发这个Web服务器的主要目的就是为了处理FriendFeed的实时特性 —— FriendFeed的每一个活动用户都会维持着一个与FriendFeed服务器的连接. (关于如何扩充服务器，以支持数以千计的客户端连接的更多信息，请参阅
`The C10K problem <http://www.kegel.com/c10k.html>`_ )

下面是经典的 “Hello, world” 示例应用：

::

    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    application = tornado.web.Application([
        (r"/", MainHandler),
    ])

    if __name__ == "__main__":
        application.listen(8888)
        tornado.ioloop.IOLoop.instance().start()

我们尝试清理代码库, 减少了各模块之间的相互依赖性, 所以(从理论上讲)你可以在自己的项目中独立地使用任何模块, 而不需要使用整个包.

请求处理器(handlers)和请求参数(arguments)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado的Web应用程序会将URL或者URL范式映射到`tornado.web.RequestHandler`. 的子类上去. 在其子类中定义了 ``get()`` 或 ``post()`` 方法, 用以处理相应URL的HTTP ``GET`` 或 ``POST`` 请求.

下面的代码将根URL ``/`` 映射到 ``MainHandler``, 将一个URL范式 ``/story/([0-9]+)`` 映射到 ``StoryHandler``. 正则表达式匹配的分组(groups)会作为参数传递给相应的 ``RequestHandler`` 方法：

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("You requested the main page")

    class StoryHandler(tornado.web.RequestHandler):
        def get(self, story_id):
            self.write("You requested the story " + story_id)

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/story/([0-9]+)", StoryHandler),
    ])

你可以使用 ``get_argument()`` 方法来获取查询字符串参数, 以及解析 ``POST`` 的主体部分(bodies)：

::

    class MyFormHandler(tornado.web.RequestHandler):
        def get(self):
            self.write('<html><body><form action="/myform" method="post">'
                       '<input type="text" name="message">'
                       '<input type="submit" value="Submit">'
                       '</form></body></html>')

        def post(self):
            self.set_header("Content-Type", "text/plain")
            self.write("You wrote " + self.get_argument("message"))

上传的文件可以通过 ``self.request.files`` 访问到, 该对象将名称(HTML元素 ``<input type="file">`` 的 name 属性) 映射到一个文件列表. 每一个文件都以字典的形式存在, 其格式为 ``{"filename":..., "content_type":..., "body":...}``.

如果你想要返回一个错误信息给客户端, 例如 “403 Unauthorized”, 只需要抛出一个 ``tornado.web.HTTPError`` 异常：

::

    if not self.user_is_logged_in():
        raise tornado.web.HTTPError(403)

请求处理程序(request handler)可以通过 ``self.request`` 访问到代表当前请求的对象. 该 ``HTTPRequest`` 对象包含了一些有用的属性, 包括：

-  ``arguments`` - 所有的 ``GET`` 和 ``POST`` 参数
-  ``files`` - 所有上传的文件(通过 ``multipart/form-data`` POST 请求)
-  ``path`` - 请求的路径( ``?`` 之前的所有内容)
-  ``headers`` - 请求的开头信息

查看 `tornado.httpserver.HTTPRequest` 的类定义可以了解到所有属性.

重写 RequestHandler 的方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了 ``get()``/``post()``/等 之外, ``RequestHandler`` 中的某些其它方法, 都是设计用于当必要时由子类重新定义的. 对于每个请求, 方法的调用次序如下：

1. 程序为每一个请求创建一个新的 RequestHandler 对象
2. ``initialize()`` 被调用, 参数是 ``Application`` 配置中的关键字参数(keyword arguments).( ``initialize`` 方法是 Tornado 1.1 中新添加的, 旧版本中子类需重写 ``__init__`` 以达到同样的目的). ``initialize`` 方法通常只是把传入的参数存到成员变量中; 它不应该产生任何输出或者调用像 ``send_error`` 之类的方法.
3. ``prepare()``被调用. 由于无论使用哪种 HTTP 方法，``prepare`` 都会被调用到, 因此这个方法通常会被定义在一个由所有处理器子类(handler subclasses)共享的基类中. ``prepare`` 可以产生输出信息; 如果它调用了 ``finish`` (或 ``send_error``, 等方法), 那么整个处理流程就此结束.
4. 某个 HTTP 方法被调用：``get()``, ``post()``, ``put()``, 等. 如果 URL 的正则表达式模式中有分组匹配, 那么相关匹配会作为参数传入方法.
5. 当请求结束, ``on_finish()`` 被调用. 对于同步处理器, 当 ``get()`` (等) 请求返回时被调用; 对于异步处理器, 在调用 ``finish()`` 后被调用.

下面是一个示例, 演示 ``initialize()`` 方法:

::

    class ProfileHandler(RequestHandler):
        def initialize(self, database):
            self.database = database

        def get(self, username):
            ...

    app = Application([
        (r'/user/(.*)', ProfileHandler, dict(database=database)),
        ])

其它设计用来被复写的方法有:

-  ``write_error(self, status_code, exc_info=None, **kwargs)`` -
   outputs HTML for use on error pages.
-  ``get_current_user(self)`` - 查看 `User
   Authentication <#user-authentication>`_ 一节
-  ``get_user_locale(self)`` - 返回 ``locale`` 对象, 以供当前用户使用
-  ``get_login_url(self)`` - 返回登录url, 供``@authenticated`` 装饰器使用 (默认在 ``Application`` 的settings参数中设置)
-  ``get_template_path(self)`` - 返回模板文件的路径
   (默认在 ``Application`` 的settings参数中设置)
-  ``set_default_headers(self)`` - may be used to set additional headers
   on the response (such as a custom ``Server`` header)

Error Handling
~~~~~~~~~~~~~~

There are three ways to return an error from a `RequestHandler`:

1. Manually call `~tornado.web.RequestHandler.set_status` and output the
   response body normally.
2. Call `~RequestHandler.send_error`.  This discards
   any pending unflushed output and calls `~RequestHandler.write_error` to
   generate an error page.
3. Raise an exception.  `tornado.web.HTTPError` can be used to generate
   a specified status code; all other exceptions return a 500 status.
   The exception handler uses `~RequestHandler.send_error` and
   `~RequestHandler.write_error` to generate the error page.

The default error page includes a stack trace in debug mode and a one-line
description of the error (e.g. "500: Internal Server Error") otherwise.
To produce a custom error page, override `RequestHandler.write_error`.
This method may produce output normally via methods such as
`~RequestHandler.write` and `~RequestHandler.render`.  If the error was
caused by an exception, an ``exc_info`` triple will be passed as a keyword
argument (note that this exception is not guaranteed to be the current
exception in ``sys.exc_info``, so ``write_error`` must use e.g.
`traceback.format_exception` instead of `traceback.format_exc`).

In Tornado 2.0 and earlier, custom error pages were implemented by overriding
``RequestHandler.get_error_html``, which returned the error page as a string
instead of calling the normal output methods (and had slightly different
semantics for exceptions).  This method is still supported, but it is
deprecated and applications are encouraged to switch to
`RequestHandler.write_error`.

重定向(Redirection)
~~~~~~~~~~~

Tornado 中重定向请求有两种主要方法: ``self.redirect``, 或者使用 ``RedirectHandler``.

你可以在 ``RequestHandler`` 方法 (例如 ``get``) 中使用 ``self.redirect``, 将用户重定向到别的地方. 另外还有一个可选参数 ``permanent``, 你可以使用它来指定这次重定向是永久的.

该参数会激发一个 ``301 Moved Permanently`` HTTP 状态, 这在某些情况下是有用的, 例如, 你要将页面重定向到一个标准链接时, 这种方式会更有利于搜索引擎优化(SEO友好).(for e.g. redirecting to a canonical URL for a page in an SEO-friendly
manner.)

``permanent`` 的默认值是 ``False``, 这是为了适用于常见的操作, 例如用户在成功发送 POST 请求 以后的重定向.

::

    self.redirect('/some-canonical-page', permanent=True)

``RedirectHandler`` 在你初始化 ``Application`` 后就获得.

例如, 注意在这个网站中我们是怎样重定向到一个较长的下载URL：

::

    application = tornado.wsgi.WSGIApplication([
        (r"/([a-z]*)", ContentHandler),
        (r"/static/tornado-0.2.tar.gz", tornado.web.RedirectHandler,
         dict(url="https://github.com/downloads/facebook/tornado/tornado-0.2.tar.gz")),
    ], **settings)

``RedirectHandler`` 的默认状态码是 ``301 Moved Permanently``, 不过如果你想使用 ``302 Found`` 状态码, 你需要将 ``permanent`` 设置为 ``False``.

::

    application = tornado.wsgi.WSGIApplication([
        (r"/foo", tornado.web.RedirectHandler, {"url":"/bar", "permanent":False}),
    ], **settings)

注意, 在 ``self.redirect`` 和 ``RedirectHandler`` 中，``permanent`` 的默认值是不同的. 这样做是有一定道理的, ``self.redirect`` 通常会被用在自定义方法中, 是由逻辑事件触发(例如环境变更, 用户认证, 或表单提交). 而 ``RedirectHandler`` 模式在每次匹配到请求 URL 时100%被触发.

模板(Templates)
~~~~~~~~~

You can use any template language supported by Python, but Tornado ships
with its own templating language that is a lot faster and more flexible
than many of the most popular templating systems out there. See the
`tornado.template` module documentation for complete documentation.

A Tornado template is just HTML (or any other text-based format) with
Python control sequences and expressions embedded within the markup:

::

    <html>
       <head>
          <title>{{ title }}</title>
       </head>
       <body>
         <ul>
           {% for item in items %}
             <li>{{ escape(item) }}</li>
           {% end %}
         </ul>
       </body>
     </html>

If you saved this template as "template.html" and put it in the same
directory as your Python file, you could render this template with:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            items = ["Item 1", "Item 2", "Item 3"]
            self.render("template.html", title="My title", items=items)

Tornado templates support *control statements* and *expressions*.
Control statements are surronded by ``{%`` and ``%}``, e.g.,
``{% if len(items) > 2 %}``. Expressions are surrounded by ``{{`` and
``}}``, e.g., ``{{ items[0] }}``.

Control statements more or less map exactly to Python statements. We
support ``if``, ``for``, ``while``, and ``try``, all of which are
terminated with ``{% end %}``. We also support *template inheritance*
using the ``extends`` and ``block`` statements, which are described in
detail in the documentation for the `tornado.template`.

Expressions can be any Python expression, including function calls.
Template code is executed in a namespace that includes the following
objects and functions (Note that this list applies to templates rendered
using ``RequestHandler.render`` and ``render_string``. If you're using
the ``template`` module directly outside of a ``RequestHandler`` many of
these entries are not present).

-  ``escape``: alias for ``tornado.escape.xhtml_escape``
-  ``xhtml_escape``: alias for ``tornado.escape.xhtml_escape``
-  ``url_escape``: alias for ``tornado.escape.url_escape``
-  ``json_encode``: alias for ``tornado.escape.json_encode``
-  ``squeeze``: alias for ``tornado.escape.squeeze``
-  ``linkify``: alias for ``tornado.escape.linkify``
-  ``datetime``: the Python ``datetime`` module
-  ``handler``: the current ``RequestHandler`` object
-  ``request``: alias for ``handler.request``
-  ``current_user``: alias for ``handler.current_user``
-  ``locale``: alias for ``handler.locale``
-  ``_``: alias for ``handler.locale.translate``
-  ``static_url``: alias for ``handler.static_url``
-  ``xsrf_form_html``: alias for ``handler.xsrf_form_html``
-  ``reverse_url``: alias for ``Application.reverse_url``
-  All entries from the ``ui_methods`` and ``ui_modules``
   ``Application`` settings
-  Any keyword arguments passed to ``render`` or ``render_string``

When you are building a real application, you are going to want to use
all of the features of Tornado templates, especially template
inheritance. Read all about those features in the `tornado.template`
section (some features, including ``UIModules`` are implemented in the
``web`` module)

Under the hood, Tornado templates are translated directly to Python. The
expressions you include in your template are copied verbatim into a
Python function representing your template. We don't try to prevent
anything in the template language; we created it explicitly to provide
the flexibility that other, stricter templating systems prevent.
Consequently, if you write random stuff inside of your template
expressions, you will get random Python errors when you execute the
template.

All template output is escaped by default, using the
``tornado.escape.xhtml_escape`` function. This behavior can be changed
globally by passing ``autoescape=None`` to the ``Application`` or
``TemplateLoader`` constructors, for a template file with the
``{% autoescape None %}`` directive, or for a single expression by
replacing ``{{ ... }}`` with ``{% raw ...%}``. Additionally, in each of
these places the name of an alternative escaping function may be used
instead of ``None``.

Note that while Tornado's automatic escaping is helpful in avoiding
XSS vulnerabilities, it is not sufficient in all cases.  Expressions
that appear in certain locations, such as in Javascript or CSS, may need
additional escaping.  Additionally, either care must be taken to always
use double quotes and ``xhtml_escape`` in HTML attributes that may contain
untrusted content, or a separate escaping function must be used for
attributes (see e.g. http://wonko.com/post/html-escaping)

Cookies and secure cookies
~~~~~~~~~~~~~~~~~~~~~~~~~~

You can set cookies in the user's browser with the ``set_cookie``
method:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_cookie("mycookie"):
                self.set_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

Cookies are easily forged by malicious clients. If you need to set
cookies to, e.g., save the user ID of the currently logged in user, you
need to sign your cookies to prevent forgery. Tornado supports this out
of the box with the ``set_secure_cookie`` and ``get_secure_cookie``
methods. To use these methods, you need to specify a secret key named
``cookie_secret`` when you create your application. You can pass in
application settings as keyword arguments to your application:

::

    application = tornado.web.Application([
        (r"/", MainHandler),
    ], cookie_secret="__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__")

Signed cookies contain the encoded value of the cookie in addition to a
timestamp and an `HMAC <http://en.wikipedia.org/wiki/HMAC>`_ signature.
If the cookie is old or if the signature doesn't match,
``get_secure_cookie`` will return ``None`` just as if the cookie isn't
set. The secure version of the example above:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_secure_cookie("mycookie"):
                self.set_secure_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

User authentication
~~~~~~~~~~~~~~~~~~~

The currently authenticated user is available in every request handler
as ``self.current_user``, and in every template as ``current_user``. By
default, ``current_user`` is ``None``.

To implement user authentication in your application, you need to
override the ``get_current_user()`` method in your request handlers to
determine the current user based on, e.g., the value of a cookie. Here
is an example that lets users log into the application simply by
specifying a nickname, which is then saved in a cookie:

::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            return self.get_secure_cookie("user")

    class MainHandler(BaseHandler):
        def get(self):
            if not self.current_user:
                self.redirect("/login")
                return
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    class LoginHandler(BaseHandler):
        def get(self):
            self.write('<html><body><form action="/login" method="post">'
                       'Name: <input type="text" name="name">'
                       '<input type="submit" value="Sign in">'
                       '</form></body></html>')

        def post(self):
            self.set_secure_cookie("user", self.get_argument("name"))
            self.redirect("/")

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], cookie_secret="__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__")

You can require that the user be logged in using the `Python
decorator <http://www.python.org/dev/peps/pep-0318/>`_
``tornado.web.authenticated``. If a request goes to a method with this
decorator, and the user is not logged in, they will be redirected to
``login_url`` (another application setting). The example above could be
rewritten:

::

    class MainHandler(BaseHandler):
        @tornado.web.authenticated
        def get(self):
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    settings = {
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

If you decorate ``post()`` methods with the ``authenticated`` decorator,
and the user is not logged in, the server will send a ``403`` response.

Tornado comes with built-in support for third-party authentication
schemes like Google OAuth. See the `tornado.auth`
for more details. Check out the `Tornado Blog example application <https://github.com/facebook/tornado/tree/master/demos/blog>`_ for a
complete example that uses authentication (and stores user data in a
MySQL database).

.. _xsrf:

Cross-site request forgery protection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Cross-site request
forgery <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_, or
XSRF, is a common problem for personalized web applications. See the
`Wikipedia
article <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_ for
more information on how XSRF works.

The generally accepted solution to prevent XSRF is to cookie every user
with an unpredictable value and include that value as an additional
argument with every form submission on your site. If the cookie and the
value in the form submission do not match, then the request is likely
forged.

Tornado comes with built-in XSRF protection. To include it in your site,
include the application setting ``xsrf_cookies``:

::

    settings = {
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

If ``xsrf_cookies`` is set, the Tornado web application will set the
``_xsrf`` cookie for all users and reject all ``POST``, ``PUT``, and
``DELETE`` requests that do not contain a correct ``_xsrf`` value. If
you turn this setting on, you need to instrument all forms that submit
via ``POST`` to contain this field. You can do this with the special
function ``xsrf_form_html()``, available in all templates:

::

    <form action="/new_message" method="post">
      {% module xsrf_form_html() %}
      <input type="text" name="message"/>
      <input type="submit" value="Post"/>
    </form>

If you submit AJAX ``POST`` requests, you will also need to instrument
your JavaScript to include the ``_xsrf`` value with each request. This
is the `jQuery <http://jquery.com/>`_ function we use at FriendFeed for
AJAX ``POST`` requests that automatically adds the ``_xsrf`` value to
all requests:

::

    function getCookie(name) {
        var r = document.cookie.match("\\b" + name + "=([^;]*)\\b");
        return r ? r[1] : undefined;
    }

    jQuery.postJSON = function(url, args, callback) {
        args._xsrf = getCookie("_xsrf");
        $.ajax({url: url, data: $.param(args), dataType: "text", type: "POST",
            success: function(response) {
            callback(eval("(" + response + ")"));
        }});
    };

For ``PUT`` and ``DELETE`` requests (as well as ``POST`` requests that
do not use form-encoded arguments), the XSRF token may also be passed
via an HTTP header named ``X-XSRFToken``.  The XSRF cookie is normally
set when ``xsrf_form_html`` is used, but in a pure-Javascript application
that does not use any regular forms you may need to access
``self.xsrf_token`` manually (just reading the property is enough to
set the cookie as a side effect).

If you need to customize XSRF behavior on a per-handler basis, you can
override ``RequestHandler.check_xsrf_cookie()``. For example, if you
have an API whose authentication does not use cookies, you may want to
disable XSRF protection by making ``check_xsrf_cookie()`` do nothing.
However, if you support both cookie and non-cookie-based authentication,
it is important that XSRF protection be used whenever the current
request is authenticated with a cookie.

Static files and aggressive file caching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can serve static files from Tornado by specifying the
``static_path`` setting in your application:

::

    settings = {
        "static_path": os.path.join(os.path.dirname(__file__), "static"),
        "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
        (r"/(apple-touch-icon\.png)", tornado.web.StaticFileHandler,
         dict(path=settings['static_path'])),
    ], **settings)

This setting will automatically make all requests that start with
``/static/`` serve from that static directory, e.g.,
`http://localhost:8888/static/foo.png <http://localhost:8888/static/foo.png>`_
will serve the file ``foo.png`` from the specified static directory. We
also automatically serve ``/robots.txt`` and ``/favicon.ico`` from the
static directory (even though they don't start with the ``/static/``
prefix).

In the above settings, we have explicitly configured Tornado to serve
``apple-touch-icon.png`` “from” the root with the ``StaticFileHandler``,
though it is physically in the static file directory. (The capturing
group in that regular expression is necessary to tell
``StaticFileHandler`` the requested filename; capturing groups are
passed to handlers as method arguments.) You could do the same thing to
serve e.g. ``sitemap.xml`` from the site root. Of course, you can also
avoid faking a root ``apple-touch-icon.png`` by using the appropriate
``<link />`` tag in your HTML.

To improve performance, it is generally a good idea for browsers to
cache static resources aggressively so browsers won't send unnecessary
``If-Modified-Since`` or ``Etag`` requests that might block the
rendering of the page. Tornado supports this out of the box with *static
content versioning*.

To use this feature, use the ``static_url()`` method in your templates
rather than typing the URL of the static file directly in your HTML:

::

    <html>
       <head>
          <title>FriendFeed - {{ _("Home") }}</title>
       </head>
       <body>
         <div><img src="{{ static_url("images/logo.png") }}"/></div>
       </body>
     </html>

The ``static_url()`` function will translate that relative path to a URI
that looks like ``/static/images/logo.png?v=aae54``. The ``v`` argument
is a hash of the content in ``logo.png``, and its presence makes the
Tornado server send cache headers to the user's browser that will make
the browser cache the content indefinitely.

Since the ``v`` argument is based on the content of the file, if you
update a file and restart your server, it will start sending a new ``v``
value, so the user's browser will automatically fetch the new file. If
the file's contents don't change, the browser will continue to use a
locally cached copy without ever checking for updates on the server,
significantly improving rendering performance.

In production, you probably want to serve static files from a more
optimized static file server like `nginx <http://nginx.net/>`_. You can
configure most any web server to support these caching semantics. Here
is the nginx configuration we use at FriendFeed:

::

    location /static/ {
        root /var/friendfeed/static;
        if ($query_string) {
            expires max;
        }
     }

Localization
~~~~~~~~~~~~

The locale of the current user (whether they are logged in or not) is
always available as ``self.locale`` in the request handler and as
``locale`` in templates. The name of the locale (e.g., ``en_US``) is
available as ``locale.name``, and you can translate strings with the
``locale.translate`` method. Templates also have the global function
call ``_()`` available for string translation. The translate function
has two forms:

::

    _("Translate this string")

which translates the string directly based on the current locale, and

::

    _("A person liked this", "%(num)d people liked this",
      len(people)) % {"num": len(people)}

which translates a string that can be singular or plural based on the
value of the third argument. In the example above, a translation of the
first string will be returned if ``len(people)`` is ``1``, or a
translation of the second string will be returned otherwise.

The most common pattern for translations is to use Python named
placeholders for variables (the ``%(num)d`` in the example above) since
placeholders can move around on translation.

Here is a properly localized template:

::

    <html>
       <head>
          <title>FriendFeed - {{ _("Sign in") }}</title>
       </head>
       <body>
         <form action="{{ request.path }}" method="post">
           <div>{{ _("Username") }} <input type="text" name="username"/></div>
           <div>{{ _("Password") }} <input type="password" name="password"/></div>
           <div><input type="submit" value="{{ _("Sign in") }}"/></div>
           {% module xsrf_form_html() %}
         </form>
       </body>
     </html>

By default, we detect the user's locale using the ``Accept-Language``
header sent by the user's browser. We choose ``en_US`` if we can't find
an appropriate ``Accept-Language`` value. If you let user's set their
locale as a preference, you can override this default locale selection
by overriding ``get_user_locale`` in your request handler:

::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            user_id = self.get_secure_cookie("user")
            if not user_id: return None
            return self.backend.get_user_by_id(user_id)

        def get_user_locale(self):
            if "locale" not in self.current_user.prefs:
                # Use the Accept-Language header
                return None
            return self.current_user.prefs["locale"]

If ``get_user_locale`` returns ``None``, we fall back on the
``Accept-Language`` header.

You can load all the translations for your application using the
``tornado.locale.load_translations`` method. It takes in the name of the
directory which should contain CSV files named after the locales whose
translations they contain, e.g., ``es_GT.csv`` or ``fr_CA.csv``. The
method loads all the translations from those CSV files and infers the
list of supported locales based on the presence of each CSV file. You
typically call this method once in the ``main()`` method of your server:

::

    def main():
        tornado.locale.load_translations(
            os.path.join(os.path.dirname(__file__), "translations"))
        start_server()

You can get the list of supported locales in your application with
``tornado.locale.get_supported_locales()``. The user's locale is chosen
to be the closest match based on the supported locales. For example, if
the user's locale is ``es_GT``, and the ``es`` locale is supported,
``self.locale`` will be ``es`` for that request. We fall back on
``en_US`` if no close match can be found.

See the `tornado.locale`
documentation for detailed information on the CSV format and other
localization methods.

.. _ui-modules:

UI modules
~~~~~~~~~~

Tornado supports *UI modules* to make it easy to support standard,
reusable UI widgets across your application. UI modules are like special
functional calls to render components of your page, and they can come
packaged with their own CSS and JavaScript.

For example, if you are implementing a blog, and you want to have blog
entries appear on both the blog home page and on each blog entry page,
you can make an ``Entry`` module to render them on both pages. First,
create a Python module for your UI modules, e.g., ``uimodules.py``:

::

    class Entry(tornado.web.UIModule):
        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", entry=entry, show_comments=show_comments)

Tell Tornado to use ``uimodules.py`` using the ``ui_modules`` setting in
your application:

::

    class HomeHandler(tornado.web.RequestHandler):
        def get(self):
            entries = self.db.query("SELECT * FROM entries ORDER BY date DESC")
            self.render("home.html", entries=entries)

    class EntryHandler(tornado.web.RequestHandler):
        def get(self, entry_id):
            entry = self.db.get("SELECT * FROM entries WHERE id = %s", entry_id)
            if not entry: raise tornado.web.HTTPError(404)
            self.render("entry.html", entry=entry)

    settings = {
        "ui_modules": uimodules,
    }
    application = tornado.web.Application([
        (r"/", HomeHandler),
        (r"/entry/([0-9]+)", EntryHandler),
    ], **settings)

Within ``home.html``, you reference the ``Entry`` module rather than
printing the HTML directly:

::

    {% for entry in entries %}
      {% module Entry(entry) %}
    {% end %}

Within ``entry.html``, you reference the ``Entry`` module with the
``show_comments`` argument to show the expanded form of the entry:

::

    {% module Entry(entry, show_comments=True) %}

Modules can include custom CSS and JavaScript functions by overriding
the ``embedded_css``, ``embedded_javascript``, ``javascript_files``, or
``css_files`` methods:

::

    class Entry(tornado.web.UIModule):
        def embedded_css(self):
            return ".entry { margin-bottom: 1em; }"

        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", show_comments=show_comments)

Module CSS and JavaScript will be included once no matter how many times
a module is used on a page. CSS is always included in the ``<head>`` of
the page, and JavaScript is always included just before the ``</body>``
tag at the end of the page.

When additional Python code is not required, a template file itself may
be used as a module. For example, the preceding example could be
rewritten to put the following in ``module-entry.html``:

::

    {{ set_resources(embedded_css=".entry { margin-bottom: 1em; }") }}
    <!-- more template html... -->

This revised template module would be invoked with

::

    {% module Template("module-entry.html", show_comments=True) %}

The ``set_resources`` function is only available in templates invoked
via ``{% module Template(...) %}``. Unlike the ``{% include ... %}``
directive, template modules have a distinct namespace from their
containing template - they can only see the global template namespace
and their own keyword arguments.

Non-blocking, asynchronous requests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When a request handler is executed, the request is automatically
finished. Since Tornado uses a non-blocking I/O style, you can override
this default behavior if you want a request to remain open after the
main request handler method returns using the
``tornado.web.asynchronous`` decorator.

When you use this decorator, it is your responsibility to call
``self.finish()`` to finish the HTTP request, or the user's browser will
simply hang:

::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            self.write("Hello, world")
            self.finish()

Here is a real example that makes a call to the FriendFeed API using
Tornado's built-in asynchronous HTTP client:

::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            http.fetch("http://friendfeed-api.com/v2/feed/bret",
                       callback=self.on_response)

        def on_response(self, response):
            if response.error: raise tornado.web.HTTPError(500)
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")
            self.finish()

When ``get()`` returns, the request has not finished. When the HTTP
client eventually calls ``on_response()``, the request is still open,
and the response is finally flushed to the client with the call to
``self.finish()``.

For a more advanced asynchronous example, take a look at the `chat
example application
<https://github.com/facebook/tornado/tree/master/demos/chat>`_, which
implements an AJAX chat room using `long polling
<http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_.  Users
of long polling may want to override ``on_connection_close()`` to
clean up after the client closes the connection (but see that method's
docstring for caveats).

Asynchronous HTTP clients
~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado includes two non-blocking HTTP client implementations:
``SimpleAsyncHTTPClient`` and ``CurlAsyncHTTPClient``. The simple client
has no external dependencies because it is implemented directly on top
of Tornado's ``IOLoop``. The Curl client requires that ``libcurl`` and
``pycurl`` be installed (and a recent version of each is highly
recommended to avoid bugs in older version's asynchronous interfaces),
but is more likely to be compatible with sites that exercise little-used
parts of the HTTP specification.

Each of these clients is available in its own module
(``tornado.simple_httpclient`` and ``tornado.curl_httpclient``), as well
as via a configurable alias in ``tornado.httpclient``.
``SimpleAsyncHTTPClient`` is the default, but to use a different
implementation call the ``AsyncHTTPClient.configure`` method at startup:

::

    AsyncHTTPClient.configure('tornado.curl_httpclient.CurlAsyncHTTPClient')

Third party authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado's ``auth`` module implements the authentication and
authorization protocols for a number of the most popular sites on the
web, including Google/Gmail, Facebook, Twitter, and FriendFeed.
The module includes methods to log users in via these sites and, where
applicable, methods to authorize access to the service so you can, e.g.,
download a user's address book or publish a Twitter message on their
behalf.

Here is an example handler that uses Google for authentication, saving
the Google credentials in a cookie for later access:

::

    class GoogleHandler(tornado.web.RequestHandler, tornado.auth.GoogleMixin):
        @tornado.web.asynchronous
        def get(self):
            if self.get_argument("openid.mode", None):
                self.get_authenticated_user(self._on_auth)
                return
            self.authenticate_redirect()

        def _on_auth(self, user):
            if not user:
                self.authenticate_redirect()
                return
            # Save the user with, e.g., set_secure_cookie()

See the `tornado.auth` module documentation for more details.

.. _debug-mode:

Debug mode and automatic reloading
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you pass ``debug=True`` to the ``Application`` constructor, the app
will be run in debug/development mode. In this mode, several features
intended for convenience while developing will be enabled:

* The app will watch for changes to its source files and reload itself
  when anything changes. This reduces the need to manually restart the
  server during development. However, certain failures (such as syntax
  errors at import time) can still take the server down in a way that
  debug mode cannot currently recover from.
* Templates will not be cached, nor will static file hashes (used by the
  ``static_url`` function)
* When an exception in a ``RequestHandler`` is not caught, an error
  page including a stack trace will be generated.

Debug mode is not compatible with ``HTTPServer``'s multi-process mode.
You must not give ``HTTPServer.start`` an argument other than 1 (or
call `tornado.process.fork_processes`) if you are using debug mode.

The automatic reloading feature of debug mode is available as a
standalone module in ``tornado.autoreload``.  The two can be used in
combination to provide extra robustness against syntax errors: set
``debug=True`` within the app to detect changes while it is running,
and start it with ``python -m tornado.autoreload myserver.py`` to catch
any syntax errors or other errors at startup.

Reloading loses any Python interpreter command-line arguments (e.g. ``-u``)
because it re-executes Python using ``sys.executable`` and ``sys.argv``.
Additionally, modifying these variables will cause reloading to behave
incorrectly.

On some platforms (including Windows and Mac OSX prior to 10.6), the
process cannot be updated "in-place", so when a code change is
detected the old server exits and a new one starts.  This has been
known to confuse some IDEs.


Running Tornado in production
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At FriendFeed, we use `nginx <http://nginx.net/>`_ as a load balancer
and static file server. We run multiple instances of the Tornado web
server on multiple frontend machines. We typically run one Tornado
frontend per core on the machine (sometimes more depending on
utilization).

When running behind a load balancer like nginx, it is recommended to
pass ``xheaders=True`` to the ``HTTPServer`` constructor. This will tell
Tornado to use headers like ``X-Real-IP`` to get the user's IP address
instead of attributing all traffic to the balancer's IP address.

This is a barebones nginx config file that is structurally similar to
the one we use at FriendFeed. It assumes nginx and the Tornado servers
are running on the same machine, and the four Tornado servers are
running on ports 8000 - 8003:

::

    user nginx;
    worker_processes 1;

    error_log /var/log/nginx/error.log;
    pid /var/run/nginx.pid;

    events {
        worker_connections 1024;
        use epoll;
    }

    http {
        # Enumerate all the Tornado servers here
        upstream frontends {
            server 127.0.0.1:8000;
            server 127.0.0.1:8001;
            server 127.0.0.1:8002;
            server 127.0.0.1:8003;
        }

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        access_log /var/log/nginx/access.log;

        keepalive_timeout 65;
        proxy_read_timeout 200;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1000;
        gzip_proxied any;
        gzip_types text/plain text/html text/css text/xml
                   application/x-javascript application/xml
                   application/atom+xml text/javascript;

        # Only retry if there was a communication error, not a timeout
        # on the Tornado server (to avoid propagating "queries of death"
        # to all frontends)
        proxy_next_upstream error;

        server {
            listen 80;

            # Allow file uploads
            client_max_body_size 50M;

            location ^~ /static/ {
                root /var/www;
                if ($query_string) {
                    expires max;
                }
            }
            location = /favicon.ico {
                rewrite (.*) /static/favicon.ico;
            }
            location = /robots.txt {
                rewrite (.*) /static/robots.txt;
            }

            location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_redirect false;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://frontends;
            }
        }
    }

WSGI and Google AppEngine
~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado comes with limited support for `WSGI <http://wsgi.org/>`_.
However, since WSGI does not support non-blocking requests, you cannot
use any of the asynchronous/non-blocking features of Tornado in your
application if you choose to use WSGI instead of Tornado's HTTP server.
Some of the features that are not available in WSGI applications:
``@tornado.web.asynchronous``, the ``httpclient`` module, and the
``auth`` module.

You can create a valid WSGI application from your Tornado request
handlers by using ``WSGIApplication`` in the ``wsgi`` module instead of
using ``tornado.web.Application``. Here is an example that uses the
built-in WSGI ``CGIHandler`` to make a valid `Google
AppEngine <http://code.google.com/appengine/>`_ application:

::

    import tornado.web
    import tornado.wsgi
    import wsgiref.handlers

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    if __name__ == "__main__":
        application = tornado.wsgi.WSGIApplication([
            (r"/", MainHandler),
        ])
        wsgiref.handlers.CGIHandler().run(application)

See the `appengine example application
<https://github.com/facebook/tornado/tree/master/demos/appengine>`_ for a
full-featured AppEngine app built on Tornado.
