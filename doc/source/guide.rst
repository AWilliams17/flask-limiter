.. _github issue #41: https://github.com/alisaifee/flask-limiter/issues/41
.. _flask apps and ip spoofing: http://esd.io/blog/flask-apps-heroku-real-ip-spoofing.html

Guide
=====

Installation
------------

::

   pip install Flask-Limiter

Quick start
-----------

.. code-block:: python

   from flask import Flask
   from flask_limiter import Limiter
   from flask_limiter.util import get_remote_address

   app = Flask(__name__)
   limiter = Limiter(
       app,
       key_func=get_remote_address,
       default_limits=["200 per day", "50 per hour"]
   )
   @app.route("/slow")
   @limiter.limit("1 per day")
   def slow():
       return ":("

   @app.route("/medium")
   @limiter.limit("1/second", override_defaults=False)
   def medium():
       return ":|"

   @app.route("/fast")
   def fast():
       return ":)"

   @app.route("/ping")
   @limiter.exempt
   def ping():
       return "PONG"


The above Flask app will have the following rate limiting characteristics:

* Rate limiting by `remote_address` of the request
* A default rate limit of 200 per day, and 50 per hour applied to all routes.
* The ``slow`` route having an explicit rate limit decorator will bypass the default
  rate limit and only allow 1 request per day.
* The ``medium`` route inherits the default limits and adds on a decorated limit
  of 1 request per second.
* The ``ping`` route will be exempt from any default rate limits.

.. note:: The built in flask static files routes are also exempt from rate limits.

Every time a request exceeds the rate limit, the view function will not get called and instead
a `429 <http://tools.ietf.org/html/rfc6585#section-4>`_ http error will be raised.

Using the extension
-------------------
The extension can be initialized with the :class:`flask.Flask` application
in the usual ways.

Using the constructor

   .. code-block:: python

      from flask_limiter import Limiter
      from flask_limiter.util import get_remote_address
      ....

      limiter = Limiter(app, key_func=get_remote_address)

Deferred app initialization using ``init_app``

    .. code-block:: python

        limiter = Limiter(key_func=get_remote_address)
        limiter.init_app(app)



.. _ratelimit-domain:

-----------------
Rate Limit Domain
-----------------
Each :class:`Limiter` instance is initialized with a `key_func` which returns the bucket
in which each request is put into when evaluating whether it is within the rate limit or not.

.. danger:: Earlier versions of Flask-Limiter defaulted the rate limiting domain to the requesting users' ip-address retreived via the :func:`flask_limiter.util.get_ipaddr` function. This behavior is being deprecated (since version `0.9.2`) as it can be susceptible to ip spoofing with certain environment setups (more details at `github issue #41`_ & `flask apps and ip spoofing`_).

It is now recommended to explicitly provide a keying function as part of the :class:`Limiter`
initialization (:ref:`keyfunc-customization`). Two utility methods are still provided:

* :func:`flask_limiter.util.get_ipaddr`: uses the last ip address in the `X-Forwarded-For` header, else falls back to the `remote_address` of the request
* :func:`flask_limiter.util.get_remote_address`: uses the `remote_address` of the request.

Please refer to :ref:`deploy-behind-proxy` for an example.


----------
Decorators
----------
The decorators made available as instance methods of the :class:`Limiter`
instance are

.. _ratelimit-decorator-limit:

:meth:`Limiter.limit`
  There are a few ways of using this decorator depending on your preference and use-case.

  Single decorator
    The limit string can be a single limit or a delimiter separated string

      .. code-block:: python

         @app.route("....")
         @limiter.limit("100/day;10/hour;1/minute")
         def my_route()
           ...

  Multiple decorators
    The limit string can be a single limit or a delimiter separated string
    or a combination of both.

        .. code-block:: python

           @app.route("....")
           @limiter.limit("100/day")
           @limiter.limit("10/hour")
           @limiter.limit("1/minute")
           def my_route():
             ...

  Custom keying function
    By default rate limits are applied based on the key function that the :class:`Limiter` instance
    was initialized with. You can implement your own function to retrieve the key to rate limit by
    when decorating individual routes. Take a look at :ref:`keyfunc-customization` for some examples..

        .. code-block:: python

            def my_key_func():
              ...

            @app.route("...")
            @limiter.limit("100/day", my_key_func)
            def my_route():
              ...

        .. note:: The key function  is called from within a
           :ref:`flask request context <flask:request-context>`.

  Dynamically loaded limit string(s)
    There may be situations where the rate limits need to be retrieved from
    sources external to the code (database, remote api, etc...). This can be
    achieved by providing a callable to the decorator.


        .. code-block:: python

               def rate_limit_from_config():
                   return current_app.config.get("CUSTOM_LIMIT", "10/s")

               @app.route("...")
               @limiter.limit(rate_limit_from_config)
               def my_route():
                   ...

        .. danger:: The provided callable will be called for every request
           on the decorated route. For expensive retrievals, consider
           caching the response.
        .. note:: The callable is called from within a
           :ref:`flask request context <flask:request-context>` during the
           `before_request` phase.

  Exemption conditions
    Each limit can be exempted when given conditions are fulfilled. These
    conditions can be specified by supplying a callable as an
    ```exempt_when``` argument when defining the limit.

        .. code-block:: python

           @app.route("/expensive")
           @limiter.limit("100/day", exempt_when=lambda: current_user.is_admin)
           def expensive_route():
             ...

.. _ratelimit-decorator-shared-limit:

:meth:`Limiter.shared_limit`
    For scenarios where a rate limit should be shared by multiple routes
    (For example when you want to protect routes using the same resource
    with an umbrella rate limit).

    Named shared limit

      .. code-block:: python

        mysql_limit = limiter.shared_limit("100/hour", scope="mysql")

        @app.route("..")
        @mysql_limit
        def r1():
           ...

        @app.route("..")
        @mysql_limit
        def r2():
           ...


    Dynamic shared limit: when a callable is passed as scope, the return value
    of the function will be used as the scope. Note that the callable takes one argument: a string representing
    the request endpoint.

      .. code-block:: python

        def host_scope(endpoint_name):
            return request.host
        host_limit = limiter.shared_limit("100/hour", scope=host_scope)

        @app.route("..")
        @host_limit
        def r1():
           ...

        @app.route("..")
        @host_limit
        def r2():
           ...


    .. note:: Shared rate limits provide the same conveniences as individual rate limits

        * Can be chained with other shared limits or individual limits
        * Accept keying functions
        * Accept callables to determine the rate limit value



.. _ratelimit-decorator-exempt:

:meth:`Limiter.exempt`
  This decorator simply marks a route as being exempt from any rate limits.

.. _ratelimit-decorator-request-filter:

:meth:`Limiter.request_filter`
  This decorator simply marks a function as a filter for requests that are going to be tested for rate limits. If any of the request filters return ``True`` no
  rate limiting will be performed for that request. This mechanism can be used to
  create custom white lists.


        .. code-block:: python

            @limiter.request_filter
            def header_whitelist():
                return request.headers.get("X-Internal", "") == "true"

            @limiter.request_filter
            def ip_whitelist():
                return request.remote_addr == "127.0.0.1"

    In the above example, any request that contains the header ``X-Internal: true``
    or originates from localhost will not be rate limited.
