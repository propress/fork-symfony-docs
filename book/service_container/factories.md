# Using a Factory to Create Services

Symfony's Service Container provides multiple features to control the creation
of objects, allowing you to specify arguments passed to the constructor as well
as calling methods and setting parameters.

However, sometimes you need to apply the `factory design pattern`_ to delegate
the object creation to some special object called "the factory". In those cases,
the service container can call a method on your factory to create the object
rather than directly instantiating the class.

## Static Factories

Suppose you have a factory that configures and returns a new `NewsletterManager`
object by calling the static `createNewsletterManager()` method::

    // src/Email/NewsletterManagerStaticFactory.php
    namespace App\Email;

    // ...

    class NewsletterManagerStaticFactory
    {
        public static function createNewsletterManager(): NewsletterManager
        {
            $newsletterManager = new NewsletterManager();

            // ...

            return $newsletterManager;
        }
    }

To make the `NewsletterManager` object available as a service, use the
`factory` option to define which method of which class must be called to
create its object:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            App\Email\NewsletterManager:
                # the first argument is the class and the second argument is the static method
                factory: ['App\Email\NewsletterManagerStaticFactory', 'createNewsletterManager']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManager;
        use App\Email\NewsletterManagerStaticFactory;

        return App::config([
            'services' => [
                // ...
                NewsletterManager::class => [
                    // the first argument is the class and the second argument is the static method
                    'factory' => [NewsletterManagerStaticFactory::class, 'createNewsletterManager'],
                ],
            ],
        ]);

.. tip::

    When configuring your services with PHP, you can use `first-class callable syntax`_
    to define the factory::

        $services->set(NewsletterManager::class)
            ->factory(NewsletterManagerStaticFactory::createNewsletterManager(...));

    This syntax also works with global functions::

        $services->set('date', \DateTime::class)
            ->factory(\date_create(...));

.. note::

    When using a factory to create services, the value chosen for class
    has no effect on the resulting service. The actual class name
    only depends on the object that is returned by the factory. However,
    the configured class name may be used by compiler passes and therefore
    should be set to a sensible value.

## Using the Class as Factory Itself

When the static factory method is on the same class as the created instance,
the class name can be omitted from the factory declaration.
Let's suppose the `NewsletterManager` class has a `create()` method that needs
to be called to create the object and needs a sender::

    // src/Email/NewsletterManager.php
    namespace App\Email;

    // ...

    class NewsletterManager
    {
        private string $sender;

        public static function create(string $sender): self
        {
            $newsletterManager = new self();
            $newsletterManager->sender = $sender;
            // ...

            return $newsletterManager;
        }
    }

You can omit the class on the factory declaration:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            App\Email\NewsletterManager:
                factory: [null, 'create']
                arguments:
                    $sender: 'fabien@symfony.com'

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManager;

        return App::config([
            'services' => [
                NewsletterManager::class => [
                    'factory' => [null, 'create'],
                    'arguments' => [
                        '$sender' => 'fabien@symfony.com',
                    ],
                ],
            ],
        ]);

It is also possible to use the `constructor` option, instead of passing `null`
as the factory class:

.. configuration-block::

    .. code-block:: php-attributes

        // src/Email/NewsletterManager.php
        namespace App\Email;

        use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

        #[Autoconfigure(bind: ['$sender' => 'fabien@symfony.com'], constructor: 'create')]
        class NewsletterManager
        {
            private string $sender;

            public static function create(string $sender): self
            {
                $newsletterManager = new self();
                $newsletterManager->sender = $sender;
                // ...

                return $newsletterManager;
            }
        }

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            App\Email\NewsletterManager:
                constructor: 'create'
                arguments:
                    $sender: 'fabien@symfony.com'

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManager;

        return App::config([
            'services' => [
                NewsletterManager::class => [
                    'constructor' => 'create',
                    'arguments' => [
                        '$sender' => 'fabien@symfony.com',
                    ],
                ],
            ],
        ]);

## Non-Static Factories

If your factory is using a regular method instead of a static one to configure
and create the service, instantiate the factory itself as a service too.
Configuration of the service container then looks like this:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            # first, create a service for the factory
            App\Email\NewsletterManagerFactory: ~

            # second, use the factory service as the first argument of the 'factory'
            # option and the factory method as the second argument
            App\Email\NewsletterManager:
                factory: ['@App\Email\NewsletterManagerFactory', 'createNewsletterManager']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManager;
        use App\Email\NewsletterManagerFactory;

        return App::config([
            'services' => [
                // first, create a service for the factory
                NewsletterManagerFactory::class => null,

                // second, use the factory service as the first argument of the 'factory'
                // option and the factory method as the second argument
                NewsletterManager::class => [
                    'factory' => [service(NewsletterManagerFactory::class), 'createNewsletterManager'],
                ],
            ],
        ]);

.. _factories-invokable:

## Invokable Factories

Suppose you now change your factory method to `__invoke()` so that your
factory service can be used as a callback::

    // src/Email/InvokableNewsletterManagerFactory.php
    namespace App\Email;

    // ...
    class InvokableNewsletterManagerFactory
    {
        public function __invoke(): NewsletterManager
        {
            $newsletterManager = new NewsletterManager();

            // ...

            return $newsletterManager;
        }
    }

Services can be created and configured via invokable factories by omitting the
method name:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            App\Email\NewsletterManager:
                class:   App\Email\NewsletterManager
                factory: '@App\Email\InvokableNewsletterManagerFactory'

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\InvokableNewsletterManagerFactory;
        use App\Email\NewsletterManager;

        return App::config([
            'services' => [
                NewsletterManager::class => [
                    'factory' => service(InvokableNewsletterManagerFactory::class),
                ],
            ],
        ]);

## Using Expressions in Service Factories

Instead of using PHP classes as a factory, you can also use
:doc:`expressions </service_container/expression_language>`. This allows you to
e.g. change the service based on a parameter:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\Email\NewsletterManagerInterface:
                # use the "tracable_newsletter" service when debug is enabled, "newsletter" otherwise.
                # "@=" indicates that this is an expression
                factory: '@=parameter("kernel.debug") ? service("tracable_newsletter") : service("newsletter")'

            App\Email\NewsletterManagerInterface:
                # you can use the arg() function to retrieve an argument from the definition
                factory: '@=arg(0).createNewsletterManager() ?: service("default_newsletter_manager")'
                arguments:
                    - '@App\Email\NewsletterManagerFactory'

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManagerFactory;
        use App\Email\NewsletterManagerInterface;

        return App::config([
            'services' => [
                NewsletterManagerInterface::class => [
                    // use the "tracable_newsletter" service when debug is enabled, "newsletter" otherwise.
                    'factory' => expr("parameter('kernel.debug') ? service('tracable_newsletter') : service('newsletter')"),
                ],
                NewsletterManagerInterface::class => [
                    // you can use the arg() function to retrieve an argument from the definition
                    'factory' => expr("arg(0).createNewsletterManager() ?: service('default_newsletter_manager')"),
                    'arguments' => [
                        service(NewsletterManagerFactory::class),
                    ],
                ],
            ],
        ]);

.. _factories-passing-arguments-factory-method:

## Passing Arguments to the Factory Method

.. tip::

    Arguments to your factory method are :ref:`autowired <services-autowire>` if
    that's enabled for your service.

If you need to pass arguments to the factory method you can use the `arguments`
option. For example, suppose the `createNewsletterManager()` method in the
previous examples takes the `templating` service as an argument:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # ...

            App\Email\NewsletterManager:
                factory:   ['@App\Email\NewsletterManagerFactory', createNewsletterManager]
                arguments: ['@templating']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Email\NewsletterManager;
        use App\Email\NewsletterManagerFactory;

        return App::config([
            'services' => [
                NewsletterManager::class => [
                    'factory' => [service(NewsletterManagerFactory::class), 'createNewsletterManager'],
                    'arguments' => [service('templating')],
                ],
            ],
        ]);

.. _`factory design pattern`: https://en.wikipedia.org/wiki/Factory_(object-oriented_programming)
.. _`first-class callable syntax`: https://www.php.net/manual/en/functions.first_class_callable_syntax.php
