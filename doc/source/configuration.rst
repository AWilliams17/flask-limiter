.. _RFC2616: https://tools.ietf.org/html/rfc2616#section-14.37
.. _ratelimit-conf:

Configuration
=============

The following flask configuration values are honored by
:class:`Limiter`. If the corresponding configuration value is passed in through
the :class:`Limiter` constructor, those will take precedence.

General Configuration
---------------------

========================================= ================================================
``RATELIMIT_ENABLED``                     Overall kill switch for rate limits. Defaults to ``True``
``RATELIMIT_STRATEGY``                    The rate limiting strategy to use.  :ref:`ratelimit-strategy`
                                          for details.
``RATELIMIT_KEY_PREFIX``                  Prefix that is prepended to each stored rate limit key. This can be useful when using a
                                          shared storage for multiple applications or rate limit domains.
``RATELIMIT_APPLICATION``                 A comma (or some other delimiter) separated string
                                          that will be used to apply limits to the application as a whole (i.e. shared
                                          by all routes).

========================================= ================================================

Default Rate limits
-------------------

========================================= ================================================
``RATELIMIT_GLOBAL``                      .. deprecated:: 0.9.4

                                          Use ``RATELIMIT_DEFAULT`` instead.
``RATELIMIT_DEFAULT``                     A comma (or some other delimiter) separated string
                                          that will be used to apply a default limit on all
                                          routes. If not provided, the default limits can be
                                          passed to the :class:`Limiter` constructor
                                          as well (the values passed to the constructor take precedence
                                          over those in the config). :ref:`ratelimit-string` for details.
``RATELIMIT_DEFAULTS_PER_METHOD``         Whether default limits are applied per method, per route or as a
                                          combination of all method per route.
``RATELIMIT_DEFAULTS_EXEMPT_WHEN``        A function that should return a truthy value if the default rate limit(s)
                                          should be skipped for the current request. This callback is called in the
                                          :ref:`flask request context <flask:request-context>` `before_request` phase.
``RATELIMIT_DEFAULTS_DEDUCT_WHEN``        A function that should return a truthy value if a deduction should be made
                                          from the default rate limit(s) for the current request. This callback is called
                                          in the :ref:`flask request context <flask:request-context>` `after_request` phase.

========================================= ================================================

Backend Storage
---------------

========================================= ================================================
``RATELIMIT_STORAGE_URL``                 A storage location conforming to the scheme in :ref:`storage-scheme`.
                                          A basic in-memory storage can be used by specifying ``memory://`` though this
                                          should probably never be used in production. Some supported backends include:

                                           - Memcached: ``memcached://host:port``
                                           - Redis: ``redis://host:port``
                                           - GAE Memcached: ``gaememcached://host:port``

                                          For specific examples and requirements of supported backends please refer to :ref:`storage-scheme`.
``RATELIMIT_STORAGE_OPTIONS``             A dictionary to set extra options to be passed to the
                                          storage implementation upon initialization. (Useful if you're
                                          subclassing :class:`limits.storage.Storage` to create a
                                          custom Storage backend.)
========================================= ================================================

.. _ratelimit-headers:

Headers
-------

If the configuration is enabled, information about the rate limit with respect to the
route being requested will be added to the response headers. Since multiple rate limits
can be active for a given route - the rate limit with the lowest time granularity will be
used in the scenario when the request does not breach any rate limits.

============================== ================================================
``X-RateLimit-Limit``          The total number of requests allowed for the
                               active window
``X-RateLimit-Remaining``      The number of requests remaining in the active
                               window.
``X-RateLimit-Reset``          UTC seconds since epoch when the window will be
                               reset.
``Retry-After``                Seconds to retry after or the http date when the
                               Rate Limit will be reset. The way the value is presented
                               depends on the configuration value set in `RATELIMIT_HEADER_RETRY_AFTER_VALUE`
                               and defaults to `delta-seconds`.
============================== ================================================

.. warning:: Enabling the headers has an additional cost with certain storage / strategy combinations.

    * Memcached + Fixed Window: an extra key per rate limit is stored to calculate
      ``X-RateLimit-Reset``
    * Redis + Moving Window: an extra call to redis is involved during every request
      to calculate ``X-RateLimit-Remaining`` and ``X-RateLimit-Reset``

The header names can be customised if required by either using the configuration
values below or by setting the ``header_mapping`` property of the :class:`Limiter` as follows::

    from flask_limiter import Limiter, HEADERS
    limiter = Limiter()
    limiter.header_mapping = {
        HEADERS.LIMIT : "X-My-Limit",
        HEADERS.RESET : "X-My-Reset",
        HEADERS.REMAINING: "X-My-Remaining"
    }
    # or by only partially specifying the overrides
    limiter.header_mapping[HEADERS.LIMIT] = 'X-My-Limit'


========================================= ================================================
``RATELIMIT_HEADERS_ENABLED``             Enables returning :ref:`ratelimit-headers`. Defaults to ``False``
``RATELIMIT_HEADER_LIMIT``                Header for the current rate limit. Defaults to ``X-RateLimit-Limit``
``RATELIMIT_HEADER_RESET``                Header for the reset time of the current rate limit. Defaults to ``X-RateLimit-Reset``
``RATELIMIT_HEADER_REMAINING``            Header for the number of requests remaining in the current rate limit. Defaults to ``X-RateLimit-Remaining``
``RATELIMIT_HEADER_RETRY_AFTER``          Header for when the client should retry the request. Defaults to ``Retry-After``
``RATELIMIT_HEADER_RETRY_AFTER_VALUE``    Allows configuration of how the value of the `Retry-After` header is rendered. One of `http-date` or `delta-seconds`. (`RFC2616`_).
========================================= ================================================


Error Handling
--------------

========================================= ================================================
``RATELIMIT_SWALLOW_ERRORS``              Whether to allow failures while attempting to perform a rate limit
                                          such as errors with downstream storage. Setting this value to ``True``
                                          will effectively disable rate limiting for requests where an error has
                                          occurred.
``RATELIMIT_IN_MEMORY_FALLBACK_ENABLED``  ``True``/``False``. If enabled an in memory rate limiter will be used
                                          as a fallback when the configured storage is down. Note that, when used in
                                          combination with ``RATELIMIT_IN_MEMORY_FALLBACK`` the original rate limits
                                          will not be inherited and the values provided in
``RATELIMIT_IN_MEMORY_FALLBACK``          A comma (or some other delimiter) separated string
                                          that will be used when the configured storage is down.
========================================= ================================================

.. _ratelimit-string:

Rate limit string notation
--------------------------

Rate limits are specified as strings following the format:

    [count] [per|/] [n (optional)] [second|minute|hour|day|month|year]

You can combine multiple rate limits by separating them with a delimiter of your
choice.

--------
Examples
--------

* 10 per hour
* 10/hour
* 10/hour;100/day;2000 per year
* 100/day, 500/7days

.. warning:: If rate limit strings that are provided to the :meth:`Limiter.limit`
   decorator are malformed and can't be parsed the decorated route will fall back
   to the default rate limit(s) and an ``ERROR`` log message will be emitted. Refer
   to :ref:`logging` for more details on capturing this information. Malformed
   default rate limit strings will however raise an exception as they are evaluated
   early enough to not cause disruption to a running application.

