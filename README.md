# redis-symfony-deploy
Let’s try to configure Doctrine to work with Redis for caching in few steps.

## Redis?
Redis is an open source (BSD licensed), in-memory data structure store, used as database, cache and message broker.
You probably know much more than I do about redis. Take a look here: http://redis.io/ 

## Installation
For Ubuntu, build it from sources:

```
wget http://download.redis.io/releases/redis-2.8.19.tar.gz
tar xzf redis-2.8.19.tar.gz
cd redis-2.8.19
make
sudo make install
cd utils
sudo ./install_server.sh
# start server
service redis_6379 start
```
For default config, redis is running on 6379 port.

Under OS X probably you need to use ```homebrew```

## Let's begin.
As always there is an awsome bundle already to work with Redis inside our application.

[snc/SncRedisBundle](https://github.com/snc/SncRedisBundle)

This bundle can work with both [Predis](https://github.com/nrk/predis) or [PHPRedis](https://github.com/phpredis/phpredis) libraries to communicate with Redis, so it’s your choice which one to use. I went with Predis.

Let’s add bundle itself and Predis as a needed dependency inside our composer.json and run ```composer install``` after that:

```
"predis/predis": "1.0.x-dev",
"snc/redis-bundle": "2.x-dev"
```
Activate bundle in ```app/AppKernel.php```:

```php
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

Configure Redis client itself and set it to be used for Doctrine in ```app/config/config.yml```:

```yml
snc_redis:
    # configure predis as client
    clients:
        default:
            type: predis
            alias: default
            dsn: redis://localhost
        doctrine:
            type: predis
            alias: doctrine
            dsn: redis://localhost
    # configure doctrine caching
    doctrine:
        metadata_cache:
            client: doctrine
            entity_manager: default
            document_manager: default
        result_cache:
            client: doctrine
            entity_manager: [default]
        query_cache:
            client: doctrine
            entity_manager: default
```
In the same file, we need to enable metadata and query caching for Doctrine:

``` yml
doctrine:
    dbal:
        driver:   %database_driver%
        host:     %database_host%
        port:     %database_port%
        dbname:   %database_name%
        user:     %database_user%
        password: %database_password%
        charset:  UTF8
    orm:
        auto_generate_proxy_classes: %kernel.debug%
        auto_mapping: true
        # enable metadata caching
        metadata_cache_driver: redis
        # enable query caching
        query_cache_driver: redis
```

## Now the fancy things
In the most cases, you don’t need cache results from all of your queries, so let’s see how you can enable result cache for some individual one only. The best place to do it is inside of an entity repository class.

Let’s say we have ```User``` entity with respective repository. In this repository, we have a method to find active users and return a list with 10 items sorted by user’s name. Let’s add caching for the results of this method.

```php
# use needed classes
use Snc\RedisBundle\Doctrine\Cache\RedisCache;
use Predis\Client;

class UserRepository extends EntityRepository
{
  ...
    public function getActiveUsers($limit = 10)
    {
      # init predis client
      $predis = new RedisCache();
      $predis->setRedis(new Client());
      # define cache lifetime period as 1 hour in seconds
      $cache_lifetime = 3600;

      return $this->getEntityManager()
          ->createQuery('SELECT c FROM AppBundle:User u '
              . 'WHERE u.active = 1 ORDER BY u.name DESC')
          ->setMaxResults($limit)
          # pass predis object as driver
          ->setResultCacheDriver($predis)
          # set cache lifetime
          ->setResultCacheLifetime($cache_lifetime)
          ->getResult();
    }  
  ...
}
```

Yeap, that’s all. Now Doctrine will get users data from the database for the first time, cache it in Redis for 1 hour and update it again after expiration.
