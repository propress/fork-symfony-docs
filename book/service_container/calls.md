# Service Method Calls and Setter Injection

.. tip::

    If you're using autowiring, you can use `#[Required]` to
    :ref:`automatically configure method calls <autowiring-calls>`.

Usually, you'll want to inject your dependencies via the constructor. But sometimes,
especially if a dependency is optional, you may want to use "setter injection". For
example::

    // src/Service/MessageGenerator.php
    namespace App\Service;

    use Psr\Log\LoggerInterface;

    class MessageGenerator
    {
        private LoggerInterface $logger;

        public function setLogger(LoggerInterface $logger): void
        {
            $this->logger = $logger;
        }

        // ...
    }

To configure the container to call the `setLogger` method, use the `calls` key:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\Service\MessageGenerator:
                # ...
                calls:
                    - setLogger: ['@logger']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Service\MessageGenerator;

        return App::config([
            'services' => [
                MessageGenerator::class => [
                    'calls' => [
                        'setLogger' => [service('logger')],
                    ],
                ],
            ],
        ]);

To provide immutable services, some classes implement immutable setters.
Such setters return a new instance of the configured class
instead of mutating the object they were called on::

    // src/Service/MessageGenerator.php
    namespace App\Service;

    use Psr\Log\LoggerInterface;

    class MessageGenerator
    {
        private LoggerInterface $logger;

        public function withLogger(LoggerInterface $logger): self
        {
            $new = clone $this;
            $new->logger = $logger;

            return $new;
        }

        // ...
    }

Because the method returns a separate cloned instance, configuring such a service means using
the return value of the wither method (`$service = $service->withLogger($logger);`).
The configuration to tell the container it should do so would be like:

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\Service\MessageGenerator:
                # ...
                calls:
                    - withLogger: !returns_clone ['@logger']

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Service\MessageGenerator;

        return App::config([
            'services' => [
                MessageGenerator::class => [
                    'calls' => [
                        ['withLogger', [service('logger')], true],
                    ],
                ],
            ],
        ]);

.. tip::

    If autowire is enabled, you can also use attributes; with the previous
    example it would be::

        #[Required]
        public function withLogger(LoggerInterface $logger): static
        {
            $new = clone $this;
            $new->logger = $logger;

            return $new;
        }

    If you don't want a method with a `static` return type and
    a `#[Required]` attribute to behave as a wither, you can
    add a `@return $this` annotation to disable the *returns clone*
    feature.
