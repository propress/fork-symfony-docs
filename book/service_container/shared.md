# How to Define Non Shared Services

In the service container, all services are shared by default. This means that
each time you retrieve the service, you'll get the *same* instance. This is
usually the behavior you want, but in some cases, you might want to always get a
*new* instance.

In order to always get a new instance, set the `shared` setting to `false`
in your service definition:

.. configuration-block::

    .. code-block:: php-attributes

        // src/SomeNonSharedService.php
        namespace App;

        use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

        #[Autoconfigure(shared: false)]
        class SomeNonSharedService
        {
            // ...
        }

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\SomeNonSharedService:
                shared: false
                # ...

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\SomeNonSharedService;

        return App::config([
            'services' => [
                SomeNonSharedService::class => [
                    'shared' => false,
                ],
            ],
        ]);

Now, whenever you request the `App\SomeNonSharedService` from the container,
you will be passed a new instance.
