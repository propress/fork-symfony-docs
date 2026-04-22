## How to Inject Instances into the Container

In some applications, you may need to inject a class instance as service,
instead of configuring the container to create a new instance.

For instance, the `kernel` service in Symfony is injected into the container
from within the `Kernel` class::

    // ...
    use Symfony\Component\HttpKernel\KernelInterface;
    use Symfony\Component\HttpKernel\TerminableInterface;

    abstract class Kernel implements KernelInterface, TerminableInterface
    {
        // ...

        protected function initializeContainer(): void
        {
            // ...
            $this->container->set('kernel', $this);

            // ...
        }
    }

Services that are set at runtime are called *synthetic services*. This service
has to be configured so the container knows the service exists during compilation
(otherwise, services depending on `kernel` will get a "service does not exist" error).

In order to do so, mark the service as synthetic in your service definition
configuration:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            # synthetic services don't specify a class
            app.synthetic_service:
                synthetic: true

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        return App::config([
            'services' => [
                'app.synthetic_service' => [
                    // synthetic services don't specify a class
                    'synthetic' => true,
                ],
            ],
        ]);

Now, you can inject the instance in the container using
:method:`Container::set() <Symfony\\Component\\DependencyInjection\\Container::set>`::

    // instantiate the synthetic service
    $theService = ...;
    $container->set('app.synthetic_service', $theService);
