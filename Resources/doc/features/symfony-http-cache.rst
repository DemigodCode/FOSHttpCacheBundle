Symfony HttpCache
=================

Symfony comes with a built-in reverse proxy written in PHP, known as
``HttpCache``. While it is certainly less efficient
than using Varnish or Nginx, it can still provide considerable performance
gains over an installation that is not cached at all. It can be useful for
running an application on shared hosting for instance
(see the `Symfony HttpCache documentation`_).

You can use features of this library with the Symfony ``HttpCache``. The basic
concept is to use event listeners on the HttpCache class.

.. warning::

    Symfony HttpCache support is currently limited to following features:

    * Purge
    * Refresh
    * Cache Tags
    * User Context

    Generic ``BAN`` operations are not supported.

Extending the correct HttpCache
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Instead of extending ``Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache``, your
``AppCache`` should extend ``FOS\HttpCacheBundle\SymfonyCache\EventDispatchingHttpCache``::

    require_once __DIR__.'/AppKernel.php';

    use FOS\HttpCacheBundle\SymfonyCache\EventDispatchingHttpCache;

    class AppCache extends EventDispatchingHttpCache
    {
    }

.. tip::

    If your class already needs to extend a different class, simply copy the event
    handling code from the ``EventDispatchingHttpCache`` into your ``AppCache`` class.
    The drawback is that you need to manually check whether you need to adjust your
    ``AppCache`` each time you update the FOSHttpCache library.

By default, the event dispatching cache kernel registers all event listeners it
knows about. You can disable listeners, or customize how they are instantiated.

If you do not need all listeners, or need to register some yourself to
customize their behavior, overwrite ``getOptions`` and return the right bitmap
in ``fos_default_subscribers``. Use the constants provided by the
``EventDispatchingHttpCache``::

    public function getOptions()
    {
        return array(
            'fos_default_subscribers' => self::SUBSCRIBER_NONE,
        );
    }

To register event listeners that you want to instantiate yourself in the cache
kernel, overwrite ``getDefaultSubscribers``::

    use FOS\HttpCache\SymfonyCache\UserContextSubscriber;

    // ...

    public function getDefaultSubscribers()
    {
        // get enabled listeners with default settings
        $subscribers = parent::getDefaultSubscribers();

        $subscribers[] = new UserContextListener(array(
            'session_name_prefix' => 'eZSESSID',
        ));

        $subscribers[] = new CustomListener();

        return $subscribers;
    }

You can also register event listeners from outside the kernel, e.g. in your
``app.php`` with the ``addListener`` and ``addSubscriber`` methods.

.. warning::

    Since Symfony 2.8, the class cache (``classes.php``) is compiled even in
    console mode by an optional warmer (``ClassCacheCacheWarmer``). This can
    produce conflicting results with the regular web entry points, because the
    class cache may contain definitions (such as the subscribers above) that
    are loaded before the class cache itself; leading to redeclaration fatal
    errors.

    There are two workarounds:

    * Disable class cache warming in console mode with e.g. a compiler pass::

        $container->getDefinition('kernel.class_cache.cache_warmer')->clearTag('kernel.cache_warmer');

    * Force loading of all classes and interfaced used by the ``HttpCache`` in
      ``app/console`` to make the class cache omit those classes. The simplest
      way to achieve this is to call ``class_exists`` resp. ``interface_exists``
      with each of them.

Event Listeners
~~~~~~~~~~~~~~~

Each cache feature has its own event listener. The listeners are provided by
the FOSHttpCache_ library. You can find the documentation for those listeners
in the :ref:`FOSHttpCache Symfony Cache documentation section <foshttpcache:symfony httpcache configuration>`.

Optimization for Single Server Installations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If your application runs on one single server, you can use the kernel
dispatcher to directly call the ``HttpCache`` rather than sending an actual
web request. This is more efficient, and you don't need to configure the server
IP address.

The :ref:`FOSHttpCache Symfony Proxy Client documentation section <foshttpcache:proxy client setup#kerneldispatcher-for-single-server-installations>`
explains how to adjust your bootstrap - you will need to do this in both
``public/index.php`` and ``bin/console``.

Once your bootstrapping is adjusted, set ``fos_http_cache.proxy_client.symfony.use_kernel_dispatcher: true``.

.. _Symfony HttpCache documentation: http://symfony.com/doc/current/book/http_cache.html#symfony-reverse-proxy
