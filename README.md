Multi-tenant Database Schema Manager for Laravel
==============

Tenanti allow you to manage multi-tenant data schema and migration manager for your Laravel application.

[![Build Status](https://travis-ci.org/orchestral/tenanti.svg?branch=3.6)](https://travis-ci.org/orchestral/tenanti)
[![Latest Stable Version](https://poser.pugx.org/orchestra/tenanti/v/stable)](https://packagist.org/packages/orchestra/tenanti)
[![Total Downloads](https://poser.pugx.org/orchestra/tenanti/downloads)](https://packagist.org/packages/orchestra/tenanti)
[![Latest Unstable Version](https://poser.pugx.org/orchestra/tenanti/v/unstable)](https://packagist.org/packages/orchestra/tenanti)
[![License](https://poser.pugx.org/orchestra/tenanti/license)](https://packagist.org/packages/orchestra/tenanti)
[![Coverage Status](https://coveralls.io/repos/github/orchestral/tenanti/badge.svg?branch=3.6)](https://coveralls.io/github/orchestral/tenanti?branch=3.6)

## Table of Content

* [Version Compatibility](#version-compatibility)
* [Installation](#installation)
* [Configuration](#configuration)
* [Usage](#usage)
* [Changelog](https://github.com/orchestral/tenanti/releases)

## Version Compatibility

Laravel  | Tenanti
:--------|:---------
 4.2.x   | 2.2.x
 5.0.x   | 3.0.x
 5.1.x   | 3.1.x
 5.2.x   | 3.2.x
 5.3.x   | 3.3.x
 5.4.x   | 3.4.x
 5.5.x   | 3.5.x
 5.6.x   | 3.6.x

## Installation

To install through composer, simply put the following in your `composer.json` file:

```json
{
    "require": {
        "orchestra/tenanti": "~3.0"
    }
}
```

And then run `composer install` to fetch the package.

### Quick Installation

You could also simplify the above code by using the following command:

    composer require "orchestra/tenanti=~3.0"

## Configuration

Next add the following service provider in `config/app.php`.

```php
'providers' => [

    // ...
    Orchestra\Tenanti\TenantiServiceProvider::class,
    Orchestra\Tenanti\CommandServiceProvider::class,
],
```

> The command utility is enabled via `Orchestra\Tenanti\CommandServiceProvider`.

### Aliases

To make development easier, you could add `Orchestra\Support\Facades\Tenanti` alias for easier reference:

```php
'aliases' => [

    'Tenanti' => Orchestra\Support\Facades\Tenanti::class,

],
```

### Publish Configuration

To make it easier to configuration your tenant setup, publish the configuration:

    php artisan vendor:publish

## Usage

### Configuration Tenant Driver for Single Database

Open `config/orchestra/tenanti.php` and customize the drivers.

```php
<?php

return [
    'drivers' => [
        'user' => [
            'model'  => App\User::class,
            'path'   => database_path('tenanti/user'),
            'shared' => true,
        ],
    ],
];
```

You can customize, or add new driver in the configuration. It is important to note that `model` configuration only work with `Eloquent` instance.

#### Setup migration autoload

For each driver, you should also consider adding the migration path into autoload (if it not already defined). To do this you can edit your `composer.json`.

##### composer.json

```json
{
    "autoload": {
        "classmap": [
            "database/tenant/users"
        ]
    }
}
```

### Setup Tenantor Model

Now that we have setup the configuration, let add an observer to our `User` class:

```php
<?php 

namespace App;

use App\Observers\UserObserver;
use Orchestra\Tenanti\Tenantor;
use Illuminate\Notifications\Notifiable;
use Orchestra\Tenanti\Contracts\TenantProvider;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements TenantProvider
{
    use Notifiable;

    /**
     * Convert to tenantor.
     * 
     * @return \Orchestra\Tenanti\Tenantor
     */
    public function asTenantor(): Tenantor
    {
        return Tenantor::fromEloquent('user', $this);
    }

    /**
     * Make a tenantor.
     *
     * @return \Orchestra\Tenanti\Tenantor
     */
    public static function makeTenantor($key, $connection = null): Tenantor
    {
        return Tenantor::make(
            'user', $key, $connection ?: (new static())->getConnectionName()
        );
    }

    /**
     * The "booting" method of the model.
     */
    protected static function boot()
    {
        parent::boot();

        static::observe(new UserObserver);
    }
}
```

and your `App\Observers\UserObserver` class should consist of the following:

```php
<?php 

namespace App\Observers;

use Orchestra\Tenanti\Observer;

class UserObserver extends Observer
{
    public function getDriverName()
    {
        return 'user';
    }
}
```

## Console Support

Tenanti include additional command to help you run bulk migration when a new schema is created, the available command resemble the usage available from `php artisan migrate` namespace.

Command                                      | Description
:--------------------------------------------|:--------------------------------------------
 php artisan tenanti:install {driver}        | Setup migration table on each entry for a given driver.
 php artisan tenanti:make {driver} {name}    | Make a new Schema generator for a given driver.
 php artisan tenanti:migrate {driver}        | Run migration on each entry for a given driver.
 php artisan tenanti:rollback {driver}       | Rollback migration on each entry for a given driver.
 php artisan tenanti:reset {driver}          | Reset migration on each entry for a given driver.
 php artisan tenanti:refresh {driver}        | Refresh migration (reset and migrate) on each entry for a given driver.
 php artisan tenanti:queue {driver} {action} | Execute any of above action using separate queue to minimize impact on current process.
 php artisan tenanti:tinker {driver} {id}    | Run tinker using a given driver and ID.

## Multi Database Connection Setup

Instead of using Tenanti with a single database connection, you could also setup a database connection for each tenant.

### Configuration Tenant Driver for Multiple Database

Open `config/orchestra/tenanti.php` and customize the drivers.

```php
<?php

return [
    'drivers' => [
        'user' => [
            'model'  => App\User::class,
            'path'   => database_path('tenanti/user'),
            'shared' => false,
        ],
    ],
];
```

By introducing a `migration` config, you can now setup the migration table name to be `tenant_migrations` instead of `user_{id}_migrations`.

### Database Connection Resolver

For tenanti to automatically resolve your multiple database connection, we need to setup the resolver. You can do this via:

```php
<?php namespace App\Providers;

use Orchestra\Support\Facades\Tenanti;

class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Tenanti::connection('tenants', function (User $entity, array $config) {
            $config['database'] = "acme_{$entity->getKey()}"; 
            // refer to config under `database.connections.tenants.*`.

            return $config;
        });
    }
}
```

Behind the scene, `$config` will contain the template database configuration fetch from `"database.connections.tenants"` (based on the first parameter `tenants`). We can dynamically modify the connection configuration and return the updated configuration for the tenant.

### Setting Default Database Connection

Alternatively you can also use Tenanti to set the default database connection for your application:

```php

use App\User;
use Orchestra\Support\Facades\Tenanti;

// ...

$user = User::find(5);

Tenanti::driver('user')->asDefaultConnection($user, 'tenants_{id}');
```

> Most of the time, this would be use in a Middleware Class when you resolve the tenant ID based on `Illuminate\Http\Request` object.

目录
版本兼容性
安装工具
配置文件
使用情况
变更日志文件
版本兼容性
Laravel Tenanti
4.2.x	2.2.x
5.0.x	3.0.x
5.1.x	3.1.x
5.2.x	3.2.x
5.3.x	3.3.x
5.4.x	3.4.x
5.5.x	3.5.x
安装
要通过 Composer 安装，只需在 composer.json file: 中放置以下

复制代码
{
 "require": {
 "orchestra/tenanti": "~3.0" }
}
然后运行 composer install 以获取包。

快速安装
你还可以使用下面的命令简化上述代码：

复制代码
composer require"orchestra/tenanti=~3.0"
配置
接下来在 config/app.php 中添加以下服务提供者。

复制代码
'providers'=> [//.. .OrchestraTenantiTenantiServiceProvider::class,OrchestraTenantiCommandServiceProvider::class,],
命令实用程序是通过 OrchestraTenantiCommandServiceProvider

别名
为了简化开发，你可以添加 OrchestraSupportFacadesTenanti 别名以方便参考：

复制代码
'aliases'=> ['Tenanti'=>OrchestraSupportFacadesTenanti::class,],
发布配置
要更轻松地配置你的租户设置，请发布配置：

复制代码
php artisan vendor:publish
用法
用于单个数据库的配置租户驱动程序
打开 config/orchestra/tenanti.php 并自定义驱动程序。

复制代码
<?phpreturn ['drivers'=> ['user'=> ['model'=>AppUser::class,'path'=> database_path('tenanti/user'),'shared'=>true, ], ],];
你可以自定义或者在配置中添加新驱动程序。 需要注意的是 model 配置只适用于 Eloquent 实例。

安装迁移自动加载
对于每个驱动程序，你还应该考虑将迁移路径添加到自动加载的( 如果还没有定义) 中。 要做到这一点，你可以编辑你的composer.json 。

composer.json复制代码
{
 "autoload": {
 "classmap": [
 "database/tenant/users" ]
 }
}
设置模型观察器
设置好配置之后，让我们将一个观察者添加到 User 类( AppProvidersAppServiceProvider 中的preferly ):

复制代码
<?phpnamespaceAppProviders;useAppUser;useAppObserversUserObserver;classAppServiceProviderextendsServiceProvider{publicfunctionboot() {User::observe(newUserObserver); }}
你的AppObserversUserObserver 类应该包含以下内容：

复制代码
<?phpnamespaceAppObservers;useOrchestraTenantiObserver;classUserObserverextendsObserver{publicfunctiongetDriverName() {return'user'; }}
控制台支持
在创建新模式时，Tenanti包含帮助你执行批量迁移的附加命令，可用命令类似于 php artisan migrate 命名空间中可用的用法。

命令说明
php artisan tenanti:install {driver}	为给定驱动程序在每个条目上设置迁移表。
php artisan tenanti:make {driver} {name}	为给定的驱动程序生成新的架构生成器。
php artisan tenanti:migrate {driver}	为给定驱动程序在每个条目上运行迁移。
php artisan tenanti:rollback {driver}	针对给定驱动程序的每个条目回滚迁移。
php artisan tenanti:reset {driver}	针对给定驱动程序重置每个条目上的迁移。
php artisan tenanti:refresh {driver}	为给定驱动程序刷新每个条目上的迁移( 重置并迁移) 。
php artisan tenanti:queue {driver} {action}	使用单独的队列执行上述任何操作，以最小化对当前进程的影响。
php artisan tenanti:tinker {driver} {id}	使用给定的驱动程序和ID运行修补程序。
多数据库连接设置
你还可以为每个租户设置数据库连接，而不是使用单个数据库连接来使用 Tenanti 。

用于多个数据库的配置租户驱动程序
打开 config/orchestra/tenanti.php 并自定义驱动程序。

复制代码
<?phpreturn ['drivers'=> ['user'=> ['model'=>AppUser::class,'path'=> database_path('tenanti/user'),'shared'=>false, ], ],];
通过引入 migration 配置，你现在可以将迁移表 NAME 设置为 tenant_migrations 而不是 user_{id}_migrations 。

数据库连接冲突解决程序
要使tenanti自动解析多数据库连接，我们需要设置冲突解决程序。 你可以通过以下方式执行这里操作：

复制代码
<?phpnamespaceAppProviders;useOrchestraSupportFacadesTenanti;classAppServiceProviderextendsServiceProvider{publicfunctionboot() {Tenanti::connection('tenants', function (User$entity, array$config) {$config['database'] ="acme_{$entity->getKey()}"; // refer to config under `database.connections.tenants.*`.return$config; }); }}
在幕后，$config 将包含来自 "database.connections.tenants" ( 基于第一个参数 tenants )的模板数据库配置提取。 我们可以动态修改连接配置并为租户返回更新的配置。

设置默认数据库连接
或者，也可以使用Tenanti为应用程序设置默认数据库连接：

复制代码
useAppUser;useOrchestraSupportFacadesTenanti;//.. .$user=User::find(5);Tenanti::driver('user')->asDefaultConnection($user, 'tenants_{id}');
大多数时候，当你根据 IlluminateHttpRequest 对象解析租户标识时，这将在一个中间件类中使用。
