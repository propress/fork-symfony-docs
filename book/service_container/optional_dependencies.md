# How to Make Service Arguments/References Optional

Sometimes, one of your services may have an optional dependency, meaning
that the dependency is not required for your service to work properly. You can
configure the container to not throw an error in this case.

## Setting Missing Dependencies to null

You can use the `null` strategy to explicitly set the argument to `null`
if the service does not exist::

    // config/services.php
    namespace Symfony\Component\DependencyInjection\Loader\Configurator;

    use App\Newsletter\NewsletterManager;

    return App::config([
        'services' => [
            NewsletterManager::class => [
                'arguments' => [service('logger')->nullOnInvalid()],
            ],
        ],
    ]);

.. note::

    The "null" strategy is not currently supported by the YAML driver.

## Ignoring Missing Dependencies

The behavior of ignoring missing dependencies is the same as the "null" behavior
except when used within a method call, in which case the method call itself
will be removed.

In the following example the container will inject a service using a method
call if the service exists and remove the method call if it does not:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\Newsletter\NewsletterManager:
                calls:
                    - setLogger: ['@?logger']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Newsletter\NewsletterManager;

        return App::config([
            'services' => [
                NewsletterManager::class => [
                    'calls' => [
                        'setLogger' => [service('logger')->ignoreOnInvalid()],
                    ],
                ],
            ],
        ]);

.. note::

    If the argument to the method call is a collection of arguments and any of
    them is missing, those elements are removed but the method call is still
    made with the remaining elements of the collection.

In YAML, the special `@?` syntax tells the service container that the
dependency is optional. The `NewsletterManager` must also be rewritten by
adding a `setLogger()` method::

        public function setLogger(LoggerInterface $logger): void
        {
            // ...
        }
