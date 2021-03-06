RESTful APIs
============

You can write RESTful APIs for iPhone, Android applications etc. very easily.

::

  import xitrum.Action
  import xitrum.annotation.GET

  @GET("articles")
  class ArticlesIndex extends Action {
    def execute() {...}
  }

  @GET("articles/:id")
  class ArticlesShow extends Action {
    def execute() {...}
  }

The same for POST, PUT, PATCH, DELETE, and OPTIONS.
Xitrum automatically handles HEAD as GET with empty response body.

For HTTP clients that do not support PUT and DELETE (like normal browsers), to
simulate PUT and DELETE, send a POST with ``_method=put`` or ``_method=delete`` in the
request body.

On web application startup, Xitrum will scan all those annotations, build the
routing table and print it out for you so that you know what APIs your
application has, like this:

::

  [INFO] Routes:
  GET /articles     quickstart.action.ArticlesIndex
  GET /articles/:id quickstart.action.ArticlesShow

Routes are automatically collected in the spirit of JAX-RS
and Rails Engines. You don't have to declare all routes in a single place.
Think of this feature as distributed routes. You can plug an app into another app.
If you have a blog engine, you can package it as a JAR file, then you can put
that JAR file into another app and that app automatically has blog feature!
Routing is also two-way: you can recreate URLs (reverse routing) in a typesafe way.
You can document routes using `Swagger Doc <http://swagger.wordnik.com/>`_.

Route cache
-----------

For better startup speed, routes are cached to file ``routes.cache``.
While developing, routes in .class files in the ``target`` directory are not
cached. If you change library dependencies that contain routes, you may need to
delete ``routes.cache``. This file should not be committed to your project
source code repository.

Route order with first and last
---------------------------------

When you want to route like this:

::

  /articles/:id --> ArticlesShow
  /articles/new --> ArticlesNew

You must make sure the second route be checked first. ``First`` is for this purpose:

::

  import xitrum.annotation.{GET, First}

  @GET("articles/:id")
  class ArticlesShow extends Action {
    def execute() {...}
  }

  @First  // This route has higher priority than "ArticlesShow" above
  @GET("articles/new")
  class ArticlesNew extends Action {
    def execute() {...}
  }

``Last`` is similar.

Multiple paths for one action
-----------------------------

::

  @GET("image", "image/:format")
  class Image extends Action {
    def execute() {
      val format = paramo("format").getOrElse("png")
      // ...
    }
  }

Dot in route
------------

::

  @GET("articles/:id", "articles/:id.:format")
  class ArticlesShow extends Action {
    def execute() {
      val id     = param[Int]("id")
      val format = paramo("format").getOrElse("html")
      // ...
    }
  }

Regex in route
--------------

Regex can be used in routes to specify requirements:

::

  GET("articles/:id<[0-9]+>")

Catch the rest of path
----------------------

``/`` character is special thus not allowed in param names. If you want to allow
it, the param must be the last and you must write like this:

::

  GET("service/:id/proxy/:*")

The path below will match:

::

  /service/123/proxy/http://foo.com/bar

To extract the ``:*`` part:

::

  val url = param("*")  // Will be "http://foo.com/bar"

Link to an action
-----------------

Xitrum tries to be typesafe. Don't write URL manually. Do like this:

::

  <a href={url[ArticlesShow]("id" -> myArticle.id)}>{myArticle.title}</a>

Redirect to another action
--------------------------

Read to know `what redirection is <http://en.wikipedia.org/wiki/URL_redirection>`_.

::

  import xitrum.Action
  import xitrum.annotation.{GET, POST}

  @GET("login")
  class LoginInput extends Action {
    def execute() {...}
  }

  @POST("login")
  class DoLogin extends Action {
    def execute() {
      ...
      // After login success
      redirectTo[AdminIndex]()
    }
  }

  GET("admin")
  class AdminIndex extends Action {
    def execute() {
      ...
      // Check if the user has not logged in, redirect him to the login page
      redirectTo[LoginInput]()
    }
  }

You can also redirect to the current action with ``redirecToThis()``.

Forward to another action
-------------------------

Use ``forwardTo[AnotherAction]()``. While ``redirectTo`` above causes the browser to
make another request, ``forwardTo`` does not.

Determine is the request is Ajax request
----------------------------------------

Use ``isAjax``.

::

  // In an action
  val msg = "A message"
  if (isAjax)
    jsRender("alert(" + jsEscape(msg) + ")")
  else
    respondText(msg)

Anti-CSRF
---------

For non-GET requests, Xitrum protects your web application from
`Cross-site request forgery <http://en.wikipedia.org/wiki/CSRF>`_ by default.

When you include ``antiCsrfMeta`` in your layout:

::

  import xitrum.Action
  import xitrum.view.DocType

  trait AppAction extends Action {
    override def layout = DocType.html5(
      <html>
        <head>
          {antiCsrfMeta}
          {xitrumCss}
          {jsDefaults}
          <title>Welcome to Xitrum</title>
        </head>
        <body>
          {renderedView}
          {jsForView}
        </body>
      </html>
    )
  }

The ``<head>`` part will include something like this:

::

  <!DOCTYPE html>
  <html>
    <head>
      ...
      <meta name="csrf-token" content="5402330e-9916-40d8-a3f4-16b271d583be" />
      ...
    </head>
    ...
  </html>

The token will be automatically included in all non-GET Ajax requests as
``X-CSRF-Token`` header sent by jQuery if you include
`xitrum.js <https://github.com/xitrum-framework/xitrum/blob/master/src/main/scala/xitrum/js.scala>`_
in your view template. xitrum.js is included in ``jsDefaults``. If you don't
use ``jsDefaults``, you can include xitrum.js in your template like this:

::

  <script type="text/javascript" src={url[xitrum.js]}></script>

antiCsrfInput and antiCsrfToken
-------------------------------

Xitrum takes CSRF token from ``X-CSRF-Token`` request header. If the header does
not exists, Xitrum takes the token from ``csrf-token`` request body param
(note: not param in the URL).

If you manually write forms, and you don't use the meta tag and xitrum.js as
described in the previous section, you need to use ``antiCsrfInput`` or
``antiCsrfToken``:

::

  form(method="post" action={url[AdminAddGroup]})
    != antiCsrfInput

::

  form(method="post" action={url[AdminAddGroup]})
    input(type="hidden" name="csrf-token" value={antiCsrfToken})

SkipCsrfCheck
-------------

When you create APIs for machines, e.g. smartphones, you may want to skip this
automatic CSRF check. Add the trait xitrum.SkipCsrfCheck to you action:

::

  import xitrum.{Action, SkipCsrfCheck}
  import xitrum.annotation.POST

  trait Api extends Action with SkipCsrfCheck

  @POST("api/positions")
  class LogPositionAPI extends Api {
    def execute() {...}
  }

  @POST("api/todos")
  class CreateTodoAPI extends Api {
    def execute() {...}
  }

Manipulate collected routes
---------------------------

Xitrum automatically collect routes on startup.
If you want to manipulate the routes, you can use
`xitrum.Config.routes <http://xitrum-framework.github.io/api/3.17/index.html#xitrum.routing.RouteCollection>`_.

Example:

::

  import xitrum.{Config, Server}

  object Boot {
    def main(args: Array[String]) {
      // You can modify routes before starting the server
      val routes = Config.routes

      // Remove routes to an action by its class
      routes.removeByClass[MyClass]()

      if (demoVersion) {
        // Remove routes to actions by a prefix
        routes.removeByPrefix("premium/features")

        // This also works
        routes.removeByPrefix("/premium/features")
      }

      ...

      Server.start()
    }
  }

Getting entire request content
------------------------------

Usually, when the request content type is not ``application/x-www-form-urlencoded``,
you may need to get the entire request content (and parse it manually etc.).

To get it as a string:

::

  val body = requestContentString

To get it as JSON:

::

  val myJValue = requestContentJValue  // => JSON4S (http://json4s.org) JValue
  val myMap = xitrum.util.SeriDeseri.fromJValue[Map[String, Int]](myJValue)

If you want to full control, use `request.getContent <http://netty.io/4.0/api/io/netty/handler/codec/http/FullHttpRequest.html>`_.
It returns a `ByteBuf <http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html>`_.

Documenting API with Swagger
----------------------------

You can document your API with `Swagger <https://developers.helloreverb.com/swagger/>`_
out of the box. Add ``@Swagger`` annotation on actions that need to be documented.
Xitrum will generate `/xitrum/swagger.json <https://github.com/wordnik/swagger-core/wiki/API-Declaration>`_.
This file can be used with `Swagger UI <https://github.com/wordnik/swagger-ui>`_
to generate interactive API documentation.

Xitrum includes Swagger UI. Access it at the path ``/xitrum/swagger-ui`` of your program,
e.g. http://localhost:8000/xitrum/swagger-ui.

.. image:: ../img/swagger.png

Let's see `an example <https://github.com/xitrum-framework/xitrum-placeholder>`_:

::

  import xitrum.{Action, SkipCsrfCheck}
  import xitrum.annotation.{GET, Swagger}

  @Swagger(
    Swagger.Tags("APIs to create images"),
    Swagger.Description("Dimensions should not be bigger than 2000 x 2000"),
    Swagger.OptStringQuery("text", "Text to render on the image, default: Placeholder"),
    Swagger.Produces("image/png"),
    Swagger.Response(200, "PNG image"),
    Swagger.Response(400, "Width or height is invalid or too big")
  )
  trait ImageApi extends Action with SkipCsrfCheck {
    lazy val text = paramo("text").getOrElse("Placeholder")
  }

  @GET("image/:width/:height")
  @Swagger(  // <-- Inherits other info from ImageApi
    Swagger.Summary("Generate rectangle image"),
    Swagger.IntPath("width"),
    Swagger.IntPath("height")
  )
  class RectImageApi extends Api {
    def execute {
      val width  = param[Int]("width")
      val height = param[Int]("height")
      // ...
    }
  }

  @GET("image/:width")
  @Swagger(  // <-- Inherits other info from ImageApi
    Swagger.Summary("Generate square image"),
    Swagger.IntPath("width")
  )
  class SquareImageApi extends Api {
    def execute {
      val width  = param[Int]("width")
      // ...
    }
  }

`JSON for Swagger <https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md>`_
will be generated when you access ``/xitrum/swagger``.

Swagger UI uses the JSON above to generate interactive API doc.

Params other than Swagger.IntPath and Swagger.OptStringQuery above: BytePath, IntQuery, OptStringForm etc.
They are in the form:

* ``<Value type><Param type>`` (required parameter)
* ``Opt<Value type><Param type>`` (optional parameter)

Value type: Byte, Int, Int32, Int64, Long, Number, Float, Double, String, Boolean, Date, DateTime

Param type: Path, Query, Body, Header, Form

Read more about `value type <https://github.com/wordnik/swagger-core/wiki/Datatypes>`_
and `param type <https://github.com/wordnik/swagger-core/wiki/Parameters>`_.
