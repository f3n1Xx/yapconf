=====
Usage
=====

In order to use Yapconf in a project, you will first need to create your specification object.
There are lots of options for this object, so we'll just start with the basics. Check out the
`Item Arguments`_ section for all the options available to you. For now, let's just assume we
have the following specification defined

.. code-block:: python

    from yapconf import YapconfSpec

    my_spec = YapconfSpec({
        'db_name': {'type': 'str'},
        'db_port': {'type': 'int'},
        'db_host': {'type': 'str'},
        'verbose': {'type': 'bool', 'default': True},
        'filename': {'type': 'str'},
    })

Now that you have a specification for your configuration, you should add some sources for where
these config values can be found from. You can find a list of all available sources in the
:ref:`sources` section.

.. code-block:: python

    # Let's say you loaded this dict from the command-line (more on that later)
    cli_args = {'filename': '/path/to/config', 'db_name': 'db_from_cli'}

    # Also assume you have /some/config.yml that has the following:
    #   db_name: db_from_config_file
    #   db_port: 1234
    config_file = '/some/config.yml' # JSON is also supported!

    # Finally, let's assume you have the following set in your environment
    # DB_NAME="db_from_environment"
    # FILENAME="/some/default/config.yml"
    # DB_HOST="localhost"

    # You can, just call load_config directly, but it is helpful to add these as sources
    # to your specification:
    my_spec.add_source('cli_args', 'dict', data=cli_args)
    my_spec.add_source('environment', 'environment')
    my_spec.add_source('config.yaml', 'yaml', filename=config_file)


Then you can load your configuration by calling ``load_config``. When using this method, it is
significant the order in which you pass yoru arguments as it sets precedence for load order. Let's
see this in practice.

.. code-block:: python

    # You can load your config:
    config = my_spec.load_config('cli_args', 'config.yaml', 'environment')


    # You now have a config object which can be accessed via attributes or keys:
    config.db_name    # > db_from_cli
    config['db_port'] # > 1234
    config.db_host    # > localhost
    config['verbose'] # > True
    config.filename   # > /path/to/config

    # If you loaded in a different order, you'll get a different result
    config = my_spec.load_config('environment', 'config.yaml', 'cli_args')
    config.db_name    # > db_from_environment


This config object is powered by python-box_ which is a handy utility
for handling your config object. It behaves just like a dictionary and you can treat it as such!

Loading config without adding sources
-------------------------------------

If all you want to do is load your configuration, you can do that without sources. The point of
the sources is to allow yapconf to eventually support watching those configs. See the `yapconf
watcher issue`_ for more details.

.. code-block:: python

    # You can load your config without the add_source calls:

    config = my_spec.load_config(cli_args, '/path/to/config.yaml', 'ENVIRONMENT')


Nested Items
------------
In a lot of cases, it makes sense to nest your configuration, for example, if we wanted to take
all of our database configuration and put it into a single dictionary, that would make a lot of
sense. You would specify this to yapconf as follows:

.. code-block:: python

    nested_spec = YapconfSpec({
        'db': {
            'type': 'dict',
            'items': {
                'name': { 'type': 'str' },
                'port': { 'type': 'int' }
            }
        }
    })

    config = nested_spec.load_config({'db': {'name': 'db_name', 'port': 1234}})

    config.db.name # returns 'name'
    config.db.port # returns 1234
    config.db      # returns the db dictionary


List Items
----------
List items are a special class of nested items which is only allowed to have a single item listed.
It can be specified as follows:

.. code-block:: python

    list_spec = YapconfSpec({
        'names': {
            'type': 'list',
            'items': {
                'name': {'type': 'str'}
            }
        }
    })

    config = list_spec.load_config({'names': ['a', 'b', 'c']})

    config.names # returns ['a', 'b', 'c']


Environment Loading
-------------------

If no ``env_name`` is specified for each item, then by default, Yapconf will automatically format
the item's name to be all upper-case and snake case. So the name ``foo_bar`` will become
``FOO_BAR`` and ``fooBar`` will become ``FOO_BAR``. If you do not want to apply this formatting,
set ``format_env`` to ``False``. Loading ``list`` items and ``dict`` items from the environment is
not supported and as such ``env_name`` s that are set for these items will be ignored.

Often times, you will want to prefix environment variables with your application name or something
else. You can set an environment prefix on the ``YapconfSpec`` item via the ``env_prefix``:

.. code-block:: python

    import os

    env_spec = Specification({'foo': {'type': 'str'}}, 'MY_APP_')

    os.environ['FOO'] = 'not_namespaced'
    os.environ['MY_APP_FOO'] = 'namespaced_value'

    config = env_spec.load_config('ENVIRONMENT')

    config.foo # returns 'namespaced_value'


.. note:: When using an ``env_name`` with ``env_prefix`` the ``env_prefix`` will still be applied
    to the name you provided. If you want to avoid this behavior, set the ``apply_env_prefix`` to
    ``False``.

As of version 0.1.2, you can specify additional environment names via: ``alt_env_names``. The
``apply_env_prefix`` flag will also apply to each of these. If your environment names collide with
other names, then an error will get raised when the specification is created.

CLI Support
-----------
Yapconf has some great support for adding your configuration items as command-line arguments by
utilizing argparse_. Let's assume the ``my_spec`` object from the original example

.. code-block:: python

    import argparse

    my_spec = YapconfSpec({
        'db_name': {'type': 'str'},
        'db_port': {'type': 'int'},
        'db_host': {'type': 'str'},
        'verbose': {'type': 'bool', 'default': True},
        'filename': {'type': 'str'},
    })

    parser = argparser.ArgumentParser()
    my_spec.add_arguments(parser)

    args = [
        '--db-name', 'db_name',
        '--db-port', '1234',
        '--db-host', 'localhost',
        '--no-verbose',
        '--filename', '/path/to/file'
    ]

    cli_values = vars(parser.parse_args(args))

    config = my_spec.load_config(cli_values)

    config.db_name # 'db_name'
    config.db_port # 1234
    config.db_host # 'localhost'
    config.verbose # False
    config.filename # '/path/to/file'

Yapconf makes adding CLI arguments very easy! If you don't want to expose something over the
command line you can set the ``cli_expose`` flag to ``False``.

Boolean Items and the CLI
^^^^^^^^^^^^^^^^^^^^^^^^^
Boolean items will add special flags to the command-line based on their defaults. If you have a
default set to ``True`` then a ``--no-{item_name}`` flag will get added. If the default is
``False`` then a ``--{{item_name}}`` will get added as an argument. If no default is specified,
then both will be added as mutually exclusive arguments.

Nested Items and the CLI
^^^^^^^^^^^^^^^^^^^^^^^^
Yapconf even supports ``list`` and ``dict`` type items from the command-line:

.. code-block:: python

    import argparse

    spec = YapconfSpec({
        'names': {
            'type': 'list',
            'items': {
                'name': {'type': 'str'}
            }
        },
        'db': {
            'type': 'dict',
            'items': {
                'host': {'type': 'str'},
                'port': {'type': 'int'}
            },
        }
    })

    parser = argparse.ArgumentParser()

    cli_args = [
        '--name', 'foo',
        '--name', 'bar',
        '--db-host', 'localhost',
        '--db-port', '1234',
        '--name', 'baz'
    ]

    cli_values = vars(parser.parse_args(args))

    config = my_spec.load_config(cli_values)

    config.names # ['foo', 'bar', 'baz']
    config.db.host # 'localhost'
    config.db.port # 1234

Limitations
^^^^^^^^^^^
There are a few limitations to how far down the rabbit-hole Yapconf is willing to go. Yapconf does
not support ``list`` type items with either ``dict`` or ``list`` children. The reason is that it
would be very cumbersome to start specifying which items belong to which dictionaries and in which
index in the list.


CLI/Environment Name Formatting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
A quick note on formatting and ``yapconf``. Yapconf tries to create sensible ways to convert your
config items into "normal" environment variables and command-line arguments. In order to do this,
we have to make some assumptions about what "normal" environment variables and command-line
arguments are.

By default, environment variables are assumed to be all upper-case, snake-case names. The item
name ``foO_BaR`` would become ``FOO_BAR`` in the environment.

By default, command-line argument are assumed to be kebab-case. The item name ``foo_bar`` would
become ``--foo-bar``

If you do not like this formatting, then you can turn it off by setting the ``format_env`` and
``format_cli`` flags.

Watching
--------
Yapconf supports watching your configuration. There are two main ways that yapconf can help you
with configuration changes. You can receive them at the global level (i.e. anytime the config
appears to change in the environment), or on an item-by-item basis.

.. note:: You can only watch sources. So if you want to use the watching functionality, you _must_
    use ``add_source`` before these calls.

The simplest way to know your configuration changed is to just use the global level:

.. code-block:: python

    def my_handler(old_config, new_config):
        print("TODO: Something with the new/old config")
        print(old_config)
        print(new_config)

    my_spec.add_source('label', 'json', '/path/to/file.json')
    thread = my_spec.spawn_watcher('label', target=my_handler)
    print(thread.isAlive())

The ``spawn_watcher`` command returns a thread. Now, any time ``/path/to/file.json`` changes the
``my_handler`` event will get called. One thing of note is that if ``/path/to/file.json`` is
deleted, then the thread will die with an exception.

If you want, you can also specify an ``eternal`` flag to the ``spawn_watcher`` call:

.. code-block:: python

    thread = my_spec.spawn_watcher('label', eternal=True)

With this flag, if the watcher dies (for example, the ``/path/to/file.json`` is deleted) then a
new watcher will be spawned in its place.

Often times, there are only certain items in a specification that you would like to watch. Parsing
the configuration can be a pain just to figure out what changed. To solve this problem, yapconf
allows you to specify a ``watch_target`` on individual items.

.. code-block:: python

    def my_handler(old_foo, new_foo):
        print("Foo value changed")
        print(old_foo)
        print(new_foo)

    spec = YapconfSpec({'foo': {'watch_target': my_handler}})


Config Documentation
--------------------
So you have this great app that can be configured easily. Now you need to pass it off to your
operations team. They want to know all the knobs they can tweak and adjust for individual
deployments. Yapconf has you covered. Simply run the ``generate_documentation`` command for your
specification, and behold beautiful documentation for your application!

.. code-block:: python

    my_spec.generate_documentation(output_file_name='config_docs.md')

The configuration documentation takes into account all of your sources. So it's best if you
can add all of your sources before the call to ``generate_documentation``

.. code-block:: python

    my_spec.add_source('Source 1 Label', 'etcd', etcd_client)
    my_spec.add_source('Source 2 Label', 'yaml', '/path/to/config.yaml')
    my_spec.add_source('environment', 'environment')

    my_spec.generate_documentation(output_file_name='config_docs.md')

This will give you some basic information about how your application can be configured! If you
want to see an example of the documentation that can be generated by yapconf you should check
out the `example configuration documentation`_ in our repo.

Config Migration
----------------
Throughout the lifetime of an application it is common to want to move configuration around,
changing both the names of configuration items and the default values for each. Yapconf also makes
this migration a breeze! Each item has a ``previous_defaults`` and ``previous_names`` values that
can be specified. These values help you migrate previous versions of config files to newer
versions. Let's see a basic example where we might want to update a config file with a new default:

.. code-block:: python

    # Assume we have a JSON config file ('/path/to/config.json') like the following:
    # {"db_name": "test_db_name", "db_host": "1.2.3.4"}

    spec = YapconfSpec({
        'db_name': {'type': 'str', 'default': 'new_default', 'previous_defaults': ['test_db_name']},
        'db_host': {'type': 'str', 'previous_defaults': ['localhost']}
    })

    # We can migrate that file quite easily with the spec object:
    spec.migrate_config_file('/path/to/config.json')

    # Will result in /path/to/config.json being overwritten:
    # {"db_name": "new_default", "db_host": "1.2.3.4"}

You can specify different output config files also:

.. code-block:: python

    spec.migrate_config_file('/path/to/config.json',
                             output_file_name='/new/path/to/config.json')

There are many values you can pass to ``migrate_config_file``, by default it looks like this:

.. code-block:: python

    spec.migrate_config_file('/path/to/config',
                             always_update=False,    # Always update values (even if you set them to None)
                             current_file_type=None, # Used for transitioning between json and yaml config files
                             output_file_name=None,  # Will default to current file name
                             output_file_type=None,  # Used for transitioning between json and yaml config files
                             create=True,            # Create the file if it doesn't exist
                             update_defaults=True    # Update the defaults
                             )


YAML Support
------------
Yapconf knows how to output and read both ``json`` and ``yaml`` files. However, to keep the
dependencies to a minimum it does not come with ``yaml``. You will have to manually install
either ``pyyaml`` or ``ruamel.yaml`` if you want to use ``yaml``.

Item Arguments
--------------

For each item in a specification, you can set any of these keys:

+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| Name              | Default          | Description                                                                                                    |
+===================+==================+================================================================================================================+
| name              | N/A              | The name of the config item                                                                                    |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| item_type         | ``'str'``        | The python type of the item ``('str', 'int', 'long', 'float', 'bool', 'complex', 'dict', 'list' )``            |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| default           | ``None``         | The default value for this item                                                                                |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| env_name          | ``name.upper()`` | The name to search in the environment                                                                          |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| description       | ``None``         | Description of the item                                                                                        |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| long_description  | ``None``         | Long description of the item, will support Markdown in the future                                              |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| required          | ``True``         | Specifies if the item is required to exist                                                                     |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| cli_short_name    | ``None``         | One-character command-line shortcut                                                                            |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| cli_name          | ``None``         | An alternate name to use on the command-line                                                                   |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| cli_choices       | ``None``         | List of possible values for the item from the command-line                                                     |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| previous_names    | ``None``         | List of previous names an item had                                                                             |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| previous_defaults | ``None``         | List of previous defaults an item had                                                                          |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| items             | ``None``         | Nested item definition for use by ``list`` or ``dict`` type items                                              |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| cli_expose        | ``True``         | Specifies if this item should be added to arguments on the command-line (nested ``list`` are always ``False``) |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| separator         | ``.``            | The separator to use for ``dict`` type items (useful for ``previous_names``)                                   |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| bootstrap         | ``False``        | A flag that indicates this item needs to be loaded before others can be loaded                                 |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| format_env        | ``True``         | A flag to determine if environment variables will be all upper-case SNAKE_CASE.                                |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| format_cli        | ``True``         | A flag to determine if we should format the command-line arguments to be kebab-case.                           |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| apply_env_prefix  | ``True``         | Apply the env_prefix even if the environment name was set manually. Ignored if ``format_env`` is ``False``     |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| choices           | ``None``         | A list of valid choices for the item. Cannot be set for ``dict`` items.                                        |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| alt_env_names     | ``[]``           | A list of alternate environment names.                                                                         |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| validator         | ``None``         | A custom validator function. Must take exactly one value and return True/False.                                |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| fallback          | ``None``         | A fully qualified backup name to fallback to if no value could be found                                        |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+
| watch_target      | ``None``         | A function to call if the config item changes (you must call ``spawn_watch`` for this to take effect.          |
+-------------------+------------------+----------------------------------------------------------------------------------------------------------------+


.. _python-box: https://github.com/cdgriffith/Box
.. _argparse: https://docs.python.org/3/library/argparse.html
.. _yapconf watcher issue: https://github.com/loganasherjones/yapconf/issues/36
.. _example configuration documentation: https://github.com/loganasherjones/yapconf/blob/master/example/doc.md
