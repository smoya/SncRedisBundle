# RedisBundle ![project status](http://stillmaintained.com/snc/SncRedisBundle.png) [![build status](https://secure.travis-ci.org/snc/SncRedisBundle.png?branch=master)](https://secure.travis-ci.org/snc/SncRedisBundle) #

## About ##

This bundle integrates [Predis](https://github.com/nrk/predis) and [phpredis](https://github.com/nicolasff/phpredis) into your Symfony2 application.

## Installation ##

### Using composer ###

**Note:** Check out this [gist](https://gist.github.com/2488761) if you want to use `composer` with Symfony2 v2.0.x!

Add the `snc/redis-bundle` package to your `require` section in the `composer.json` file.

``` json
{
    "require": {
        "snc/redis-bundle": "2.1.x-dev"
    }
}
```

### Using the symfony-standard vendor script ###

Append the following lines to your `deps` file:

    [SncRedisBundle]
        git=git://github.com/snc/SncRedisBundle.git
        target=/bundles/Snc/RedisBundle
        version=origin/master

    [predis]
        git=git://github.com/nrk/predis.git
        version=origin/v0.7

then run the `php bin/vendors install` command.

Register the `Snc` and `Predis` namespaces in the autoloader (`app/autoload.php`):

``` php
<?php
$loader->registerNamespaces(array(
    // ...
    'Snc'                            => __DIR__.'/../vendor/bundles',
    'Predis'                         => __DIR__.'/../vendor/predis/lib',
    // ...
));
```

Add the RedisBundle to your application's kernel:

``` php
<?php
public function registerBundles()
{
    $bundles = array(
        // ...
        new Snc\RedisBundle\SncRedisBundle(),
        // ...
    );
    ...
}
```

## Usage ##

Configure the `redis` client(s) in your `config.yml`:

``` yaml
snc_redis:
    clients:
        default:
            type: predis
            alias: default
            dsn: redis://localhost
```

You have to configure at least one client. In the above example your service
container will contain the service `snc_redis.default` which will return a
`Predis` client.

Available types are `predis` and `phpredis`.

A more complex setup which contains a clustered client could look like this:

``` yaml
snc_redis:
    clients:
        default:
            type: predis
            alias: default
            dsn: redis://localhost
            logging: %kernel.debug%
        cache:
            type: predis
            alias: cache
            dsn: redis://secret@localhost/1
            options:
                profile: 2.2
                connection_timeout: 10
                read_write_timeout: 30
        session:
            type: predis
            alias: session
            dsn: redis://localhost/2
        cluster:
            type: predis
            alias: cluster
            dsn:
                - redis://localhost/3?weight=10
                - redis://localhost/4?weight=5
                - redis://localhost/5?weight=1
```

In your controllers you can now access all your configured clients:

``` php
<?php
$redis = $this->container->get('snc_redis.default');
$val = $redis->incr('foo:bar');
$redis_cluster = $this->container->get('snc_redis.cluster');
$val = $redis_cluster->get('ab:cd');
$val = $redis_cluster->get('ef:gh');
$val = $redis_cluster->get('ij:kl');
```

### Sessions ###

Use Redis sessions by adding the following to your config:

``` yaml
snc_redis:
    ...
    session:
        client: session
```

This will use the default prefix `session`.

You may specify another `prefix`:

``` yaml
snc_redis:
    ...
    session:
        client: session
        prefix: foo
```

You can disable the automatic registration of the `session.storage` alias
by setting `use_as_default` to `false`:

``` yaml
snc_redis:
    ...
    session:
        client: session
        prefix: foo
        use_as_default: false
```

By default, a TTL is set using the `framework.session.cookie_lifetime` parameter. But
you can override it using the `ttl` option:

``` yaml
snc_redis:
    ...
    session:
        client: session
        ttl: 1200
```

This will make session data expire after 20 minutes, on the **server side**.
This is hightly recommended if you don't set an expiration date to the session
cookie. Note that using Redis for storing sessions is a good solution to avoid
garbage collection of sessions by PHP.

### Doctrine caching ###

Use Redis caching for Doctrine by adding this to your config:

``` yaml
snc_redis:
    ...
    doctrine:
        metadata_cache:
            client: cache
            entity_manager: default          # the name of your entity_manager connection
            document_manager: default        # the name of your document_manager connection
        result_cache:
            client: cache
            entity_manager: [default, read]  # you may specify multiple entity_managers
        query_cache:
            client: cache
            entity_manager: default
```

### Monolog logging ###

You can store your logs in a redis `LIST` by adding this to your config:

``` yaml
snc_redis:
    clients:
        monolog:
            type: predis
            alias: monolog
            dsn: redis://localhost/1
            logging: false
            options:
                connection_persistent: true
    monolog:
        client: monolog
        key: monolog

monolog:
    handlers:
        main:
            type: service
            id: monolog.handler.redis
            level: debug
```

### SwiftMailer spooling ###

You can spool your mails in a redis `LIST` by adding this to your config:

``` yaml
snc_redis:
    clients:
        default:
            type: predis
            alias: default
            dsn: redis://localhost
            logging: false
    swiftmailer:
        client: default
        key: swiftmailer
```

Additionally you have to set the swiftmailer spool type to `redis`:

``` yaml
swiftmailer:
    ...
    spool:
        type: redis
```


### Complete configuration example ###

``` yaml
snc_redis:
    clients:
        default:
            type: predis
            alias: default
            dsn: redis://localhost
            logging: %kernel.debug%
        cache:
            type: predis
            alias: cache
            dsn: redis://localhost/1
            logging: true
        cluster:
            type: predis
            alias: cluster
            dsn:
                - redis://127.0.0.1/1
                - redis://127.0.0.2/2
                - redis://pw@/var/run/redis/redis-1.sock/10
                - redis://pw@127.0.0.1:63790/10 ]
            options:
                profile: 2.4
                connection_timeout: 10
                connection_persistent: true
                read_write_timeout: 30
                iterable_multibulk: false
                throw_errors: true
                cluster: Snc\RedisBundle\Client\Predis\Connection\PredisCluster
    session:
        client: default
        prefix: foo
        use_as_default: true
    doctrine:
        metadata_cache:
            client: cache
            entity_manager: default
            document_manager: default
        result_cache:
            client: cache
            entity_manager: [default, read]
            document_manager: [default, slave1, slave2]
            namespace: "dcrc:"
        query_cache:
            client: cache
            entity_manager: default
    monolog:
        client: cache
        key: monolog
    swiftmailer:
        client: default
        key: swiftmailer
```
