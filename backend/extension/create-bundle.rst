.. index::
    single: Bundle; Create a Bundle
    single: Customization; Create a custom Bundle

.. _how-to-create-new-bundle:
.. _dev-cookbook-framework-how-to-create-new-bundle:

Create a Bundle
===============

Create a Bundle Manually
------------------------

First you need to specify name and namespace of your bundle. Symfony framework already has
|best practices for bundle structure and bundle name| and we recommend to follow these practices and use them.

Let us assume that we want to create the AcmeDemoBundle and put it under the namespace ``Acme\Bundle\DemoBundle``
in the ``/src`` directory. We need to create the corresponding directory structure and the bundle file with the following content:

.. code-block:: php
   :caption: src/Acme/Bundle/DemoBundle/AcmeDemoBundle.php

    namespace Acme\Bundle\DemoBundle;

    use Symfony\Component\HttpKernel\Bundle\Bundle;

    class AcmeDemoBundle extends Bundle
    {
    }

Basically, it is a regular Symfony bundle. The only difference is in the way it will be enabled (see chapter `Enable a Bundle`_).

Create a Bundle Service Container Extension
-------------------------------------------

For a load configuration files you need to create Service Container Extension. See |Symfony Configuration Files|

.. code-block:: php
   :caption: src/Acme/Bundle/DemoBundle/DependencyInjection/AcmeDemoExtension.php

    namespace Acme\Bundle\DemoBundle\DependencyInjection;

    use Symfony\Component\Config\FileLocator;
    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Extension\Extension;
    use Symfony\Component\DependencyInjection\Loader;

    class AcmeDemoExtension extends Extension
    {
        /**
         * @inheritDoc
         */
       public function load(array $configs, ContainerBuilder $container)
        {
            $loader = new Loader\YamlFileLoader($container, new FileLocator(__DIR__ . '/../Resources/config'));
            // register services configuration
            $loader->load('services.yml');
            // register other configurations in the same way
        }
    }

Create basic ``Resources/config/services.yml`` for define service parameters. See |Symfony Service Parameters|

.. code-block:: yaml
  :caption: src/Acme/Bundle/DemoBundle/Resources/config/services.yml

   services:


.. Create bundle automatically
.. ---------------------------
..
.. Also new bundle can be generated using `Symfony command generate:bundle`_:
..
.. .. _Symfony command generate:bundle: http://symfony.com/doc/2.4/bundles/SensioGeneratorBundle/commands/generate_bundle.html
..
.. .. code-block:: none
..
..
..     user@host:/var/www/vhosts/platform-application$ php bin/console generate:bundle
..     Bundle namespace: Acme/Bundle/DemoBundle
..     Bundle name [AcmeDemoBundle]:
..     Target directory [/var/www/vhosts/platform-application/src]:
..     Configuration format (yml, xml, php, or annotation): yml
..     Do you want to generate the whole directory structure [no]?
..     Do you confirm generation [yes]?
..
..     Generating the bundle code: OK
..     Checking that the bundle is autoloaded: OK
..
..     Confirm automatic update of your Kernel [yes]? no
..     Enabling the bundle inside the Kernel: FAILED
..     Confirm automatic update of the Routing [yes]? no
..     Importing the bundle routing resource: FAILED
..
.. It is important that you don't need to update Kernel and routing, as OroPlatform provides its own way to do that,
.. which will be described in the `Enable bundle`_ chapter and in following articles.
..
.. .. note::
..
..     Automatic bundle generation is provided by the ``sensio/generator-bundle`` package, which is defined in the
..     ``require-dev`` section of the ``composer.json`` file in the OroPlatform repository. Therefore, in order to use
..     automatic generation, please, make sure that this package has been installed (one of the ways to do so is to execute
..     ``composer update`` at the project's root directory to get all packages from the ``require-dev`` section).
..

Enable a Bundle
---------------

Now you have all the required files to enable the new bundle. To enable the bundle:

#. Create a Resources/config/oro/bundles.yml file with the following content:

   .. code-block:: yaml
      :caption: src/Acme/Bundle/DemoBundle/Resources/config/oro/bundles.yml

       bundles:
           - Acme\Bundle\DemoBundle\AcmeDemoBundle

   This file provides a list of bundles to register. All such files are automatically parsed to load the required bundles.

#. Regenerate the application cache using console command ``cache:clear``:

   .. code-block:: none

       user@host:/var/www/vhosts/platform-application$ php bin/console cache:clear
       Clearing the cache for the dev environment with debug true

   .. note::

       If you are working in production environment, use parameter ``--env=prod`` with the command.

#. Check if your bundle is registered and active with following command:

   .. code-block:: none

      php bin/console debug:container --parameter=kernel.bundles --format=json | grep AcmeDemoBundle

   .. note::

       Replace `grep` argument with your bundle's proper name

#. When your bundle is registered and active, the following output (or a similar one) will be displayed in the console after running the command:

   .. code-block:: none

      "AcmeDemoBundle": "Acme\\Bundle\\DemoBundle\\AcmeDemoBundle",

References
----------

* |Symfony Best Practices for Structuring Bundles|
* |Symfony Framework Events|
* :ref:`Bundle-less Structure <dev-backend-architecture-bundle-less-structure>`


.. include:: /include/include-links-dev.rst
   :start-after: begin
