.. _topics-addons:

=======
Add-ons
=======

Scrapy's add-on system is a framework which unifies managing and configuring
components that extend Scrapy's core functionality, such as middlewares,
extensions, or pipelines. It provides users with a plug-and-play experience in
Scrapy extension management, and grants extensive configuration control to
developers.


Activating and configuring add-ons
==================================

During :class:`~scrapy.crawler.Crawler` initialization, the list of enabled
add-ons is read from your ``ADDONS`` setting.

The ``ADDONS`` setting is a dict in which every key is an add-on class or its
import path and the value is its priority.

This is an example where two add-ons are enabled in a project's
``settings.py``::

    ADDONS = {
        'path.to.someaddon': 0,
        SomeAddonClass: 1,
    }


Writing your own add-ons
========================

Add-ons are :ref:`components <topics-components>` that include one or both of
the following methods:

.. method:: update_settings(settings)

    This method is called during the initialization of the
    :class:`~scrapy.crawler.Crawler`. Here, you should perform dependency checks
    (e.g. for external Python libraries) and update the
    :class:`~scrapy.settings.Settings` object as wished, e.g. enable components
    for this add-on or set required configuration of other extensions.

    :param settings: The settings object storing Scrapy/component configuration
    :type settings: :class:`~scrapy.settings.Settings`

.. classmethod:: update_pre_crawler_settings(cls, settings)

    Use this class method instead of the :meth:`update_settings` method to
    update :ref:`pre-crawler settings <pre-crawler-settings>` whose value is
    used before the :class:`~scrapy.crawler.Crawler` object is created.

    :param settings: The settings object storing Scrapy/component configuration
    :type settings: :class:`~scrapy.settings.BaseSettings`

The settings set by the add-on should use the ``addon`` priority (see
:ref:`populating-settings` and :func:`scrapy.settings.BaseSettings.set`)::

    class MyAddon:
        def update_settings(self, settings):
            settings.set("DNSCACHE_ENABLED", True, "addon")

This allows users to override these settings in the project or spider
configuration.

When editing the value of a setting instead of overriding it entirely, it is
usually best to leave its priority unchanged. For example, when editing a
:ref:`component priority dictionary <component-priority-dictionaries>`.

If the ``update_settings`` method raises
:exc:`scrapy.exceptions.NotConfigured`, the add-on will be skipped. This makes
it easy to enable an add-on only when some conditions are met.

Fallbacks
---------

Some components provided by add-ons need to fall back to "default"
implementations, e.g. a custom download handler needs to send the request that
it doesn't handle via the default download handler, or a stats collector that
includes some additional processing but otherwise uses the default stats
collector. And it's possible that a project needs to use several custom
components of the same type, e.g. two custom download handlers that support
different kinds of custom requests and still need to use the default download
handler for other requests. To make such use cases easier to configure, we
recommend that such custom components should be written in the following way:

1. The custom component (e.g. ``MyDownloadHandler``) shouldn't inherit from the
   default Scrapy one (e.g.
   ``scrapy.core.downloader.handlers.http.HTTPDownloadHandler``), but instead
   be able to load the class of the fallback component from a special setting
   (e.g. ``MY_FALLBACK_DOWNLOAD_HANDLER``), create an instance of it and use
   it.
2. The add-ons that include these components should read the current value of
   the default setting (e.g. ``DOWNLOAD_HANDLERS``) in their
   ``update_settings()`` methods, save that value into the fallback setting
   (``MY_FALLBACK_DOWNLOAD_HANDLER`` mentioned earlier) and set the default
   setting to the component provided by the add-on (e.g.
   ``MyDownloadHandler``). If the fallback setting is already set by the user,
   they shouldn't change it.
3. This way, if there are several add-ons that want to modify the same setting,
   all of them will fallback to the component from the previous one and then to
   the Scrapy default. The order of that depends on the priority order in the
   ``ADDONS`` setting.


Add-on examples
===============

Set some basic configuration:

.. skip: next
.. code-block:: python

    from myproject.pipelines import MyPipeline


    class MyAddon:
        def update_settings(self, settings):
            settings.set("DNSCACHE_ENABLED", True, "addon")
            settings.remove_from_list("METAREFRESH_IGNORE_TAGS", "noscript")
            settings.setdefault_in_component_priority_dict(
                "ITEM_PIPELINES", MyPipeline, 200
            )

.. _priority-dict-helpers:

.. tip:: When editing a :ref:`component priority dictionary
    <component-priority-dictionaries>` setting, like :setting:`ITEM_PIPELINES`,
    consider using setting methods like
    :meth:`~scrapy.settings.BaseSettings.replace_in_component_priority_dict`,
    :meth:`~scrapy.settings.BaseSettings.set_in_component_priority_dict`
    and
    :meth:`~scrapy.settings.BaseSettings.setdefault_in_component_priority_dict`
    to avoid mistakes.

Check dependencies:

.. code-block:: python

    class MyAddon:
        def update_settings(self, settings):
            try:
                import boto
            except ImportError:
                raise NotConfigured("MyAddon requires the boto library")
            ...

Access the crawler instance:

.. code-block:: python

    class MyAddon:
        def __init__(self, crawler) -> None:
            super().__init__()
            self.crawler = crawler

        @classmethod
        def from_crawler(cls, crawler):
            return cls(crawler)

        def update_settings(self, settings): ...

Use a fallback component:

.. code-block:: python

    from scrapy.core.downloader.handlers.http import HTTPDownloadHandler
    from scrapy.utils.misc import build_from_crawler


    FALLBACK_SETTING = "MY_FALLBACK_DOWNLOAD_HANDLER"


    class MyHandler:
        lazy = False

        def __init__(self, settings, crawler):
            dhcls = load_object(settings.get(FALLBACK_SETTING))
            self._fallback_handler = build_from_crawler(dhcls, crawler)

        def download_request(self, request, spider):
            if request.meta.get("my_params"):
                # handle the request
                ...
            else:
                return self._fallback_handler.download_request(request, spider)


    class MyAddon:
        def update_settings(self, settings):
            if not settings.get(FALLBACK_SETTING):
                settings.set(
                    FALLBACK_SETTING,
                    settings.getwithbase("DOWNLOAD_HANDLERS")["https"],
                    "addon",
                )
            settings["DOWNLOAD_HANDLERS"]["https"] = MyHandler
