# OpenConext Monitor bundle

[![Code_Checks](https://github.com/OpenConext/Monitor-bundle/actions/workflows/code_checks.yaml/badge.svg)](https://github.com/OpenConext/Monitor-bundle/actions/workflows/code_checks.yaml)

A Symfony 5/6/7 bundle that adds a /health and /info endpoint to your application.

The endpoints return JSON responses. The `/internal/info` endpoint tries to give as much information about the currently installed 
version of the application as possible. This information is based on the build path of the installation. But also
includes the Symfony environment that is currently active and whether or not the debugger is enabled.

The `/internal/health` endpoint reports on the health of the application. This information could be used for example by a load
balancer. Example output:

```json
{"status":"UP"}
``` 

When a health check failed the HTTP Response status code will be 503. And the JSON Response is formatted like this: 
```json
{"status":"DOWN", "message":"Lorem ipsum dolor sit"}
``` 

:exclamation: Please note that only the first failing health check is reported.

:exclamation: As of version 3.1.0 we started exposing the `health` and `info` routes on `/internal/`. On the next major version we will stop serving the `info` and `health` enpoints on `/`


## Installation

 * Add the package to your Composer file
    ```sh
    composer require openconext/monitor-bundle
    ```

 * If you don't use [Symfony Flex](https://symfony.com/doc/current/setup/flex.html), you must enable the bundle manually in the application:

    ```php
    // config/bundles.php
    // in older Symfony apps, enable the bundle in config/bundles.php
    return [
    // ...
     OpenConext\MonitorBundle\OpenConextMonitorBundle::class => ['all' => true],
    ];
     ```
   
 * Include the routing configuration in `config/routes.yaml` by adding:
    ```yaml
    open_conext_monitor:
        resource:   "@OpenConextMonitorBundle/Resources/config/routing.yml"
        prefix:     /
     ```
 
 * Add security exceptions in `config/packages/security.yaml` (if this is required at all)
    ```yaml
    security:
        firewalls:
            monitor:
                pattern: ^/(info|health)$
                security: false

    ```
 * The /info and /health endpoints should now be available for everybody. Applying custom access restriction is up to
    the implementer of this bundle. 
    
## Adding Health Checks
The Monitor ships with two health checks. These checks are a
 - Database connection check based on Doctrine configuration
 - Session status check
 
### Create the checker
A `HealthCheckInterface` can be implemented to create your own health check. The example below shows an example of what
an implementation of said interface could look like.

```php
use OpenConext\MonitorBundle\HealthCheck\HealthCheckInterface;
use OpenConext\MonitorBundle\HealthCheck\HealthReportInterface;
use OpenConext\MonitorBundle\Value\HealthReport;

class ApiHealthCheck implements HealthCheckInterface
{
    /**
     * @var MyService
     */
    private $testService;

    public function __construct(MyService $service)
    {
        $this->testService = $service;
    }

    public function check(HealthReportInterface $report): HealthReportInterface
    {
        if (!$this->testService->everythingOk()) {
            // Return a HealthReport with a DOWN status when there are indications the application is not functioning as
            // intended. You can provide an optional message that is displayed alongside the DOWN status.
            return HealthReport::buildStatusDown('Not everything is allright.');
        }
        // By default return the report that was passed along as a parameter to the check method
        return $report;
    }
}
``` 
:exclamation: Please note that the check method receives and returns a HealthReport. The Health report is passed along in the chain of
registered health checkers. If everything was OK, just return the report that was passed to the method. 

### Register the checker
To actually include the home made checker simply tag it with 'surfnet.monitor.health_check'

Example service definition in `services.yml`

```yaml
services:
    acme.monitor.my_custom_health_check:
        class: Acme\AppBundle\HealthCheck\MyCustomHealthCheck
        arguments:
            - @test_service
        tags:
            - { name: surfnet.monitor.health_check }
```

## Overriding a default HealthCheck
To run a custom query with the DoctrineConnectionHealthCheck you will need to override it in your own project.

For example in your ACME bunde that is using the monitor bundle:

`services.yml`
```yaml
    # Override the service, service names can be found in `/src/Resources/config/services.yml`
    openconext.monitor.database_health_check:
        # Point to your own implementation of the check
        class: Acme\GreatSuccessBundle\HealthCheck\DoctrineConnectionHealthCheck
        # Do not forget to apply the correct tag
        tags:
        - { name: openconext.monitor.health_check }

```

The rest of the service configuration is up to your own needs. You can inject arguments, factory calls and other service features as need be.

## Release strategy
Please read: https://github.com/OpenConext/Stepup-Deploy/wiki/Release-Management for more information on the release strategy used in Stepup projects.
