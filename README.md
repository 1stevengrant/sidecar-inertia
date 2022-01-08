# Sidecar SSR for InertiaJS

> 🚨 This is currently very much in beta!
 
## Overview 

This package provides a Sidecar function to run [Inertia server-side rendering](https://inertiajs.com/server-side-rendering) on AWS Lambda.

Sidecar packages, deploys, and executes AWS Lambda functions from your Laravel application.

It works with any Laravel 7 or 8 application, hosted anywhere including your local machine, Vapor, Heroku, a shared virtual server, or any other kind of hosting environment.

- [Sidecar docs](https://hammerstone.dev/sidecar/docs/main/overview)
- [Sidecar GitHub](https://github.com/hammerstonedev/sidecar)

You can see a fully working Jetstream + Inertia + Sidecar demo app at hammerstone/TODO

## Enabling SSR

Following the [official Inertia docs](https://inertiajs.com/server-side-rendering#enabling-ssr) on enabling SSR is a good place to start, but there are a few things you can skip:
 
- You do not need to `npm install @inertiajs/server`
- You do not need to `npm install webpack-node-externals`
- Come back here when you get to the "Building your application" section

### Updating Inertia

Make sure that `inertia/laravel-inertia` is at least version `0.5.1`.

### Installation

To require this package, run the following: 

```shell
composer require hammerstone/sidecar-inertia
```

This will install Sidecar as well.

### Use the Sidecar Gateway 

Update your `AppServiceProvider` to use the `SidecarGateway` as the default Inertia SSR Gateway

```php
namespace App\Providers;

use Hammerstone\Sidecar\Inertia\SidecarGateway;
use Illuminate\Support\ServiceProvider;
use Inertia\Ssr\Gateway;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Use Sidecar to run Inertia SSR.
        $this->app->instance(Gateway::class, new SidecarGateway);
    }
}
```

### Update Configuration

Update your `config/inertia.php` to include the Sidecar settings

```php
<?php

return [
    'ssr' => [
        'enabled' => true,

        'sidecar' => [
            // The Sidecar function that handles the SSR.
            'handler' => \Hammerstone\Sidecar\Inertia\SSR::class,
            
            // Log some stats on how long each Lambda request takes.
            'timings' => false,
            
            // Throw exceptions, should they occur.
            'debug' => env('APP_DEBUG', false),
        ],
    ],

    // ...
];
```

### Configure Sidecar

If you haven't already, you'll need to configure Sidecar.

Publish the `sidecar.php` configuration file by running 

```shell
php artisan sidecar:install
```

To configure your Sidecar AWS credentials interactively, you can run 

```shell
php artisan sidecar:configure
```

The [official Sidecar docs](https://hammerstone.dev/sidecar/docs/main/configuration) go into much further detail.

Now update your `config/sidecar.php` to include the function shipped with this package.

```php
<?php

return [
    'functions' => [
        \Hammerstone\Sidecar\Inertia\SSR::class
    ],
    
    // ...
];
```

### Updating Your JavaScript

> This only covers Vue3, please follow the Inertia docs for Vue2 or React, and please open any issues.

You'll need to update your `webpack.ssr.mix.js` file. This _should_ work for most cases, but please open any issues for errors you run into. (This is based on the Inertia docs, with slight modifications.)

```js
const path = require('path')
const mix = require('laravel-mix')

mix
    .js('resources/js/ssr.js', 'public/js')
    .options({
        manifest: false
    })
    .vue({
        version: 3,
        useVueStyleLoader: true,
        options: {
            optimizeSSR: true
        }
    })
    .alias({
        '@': path.resolve('resources/js')
    })
    .webpackConfig({
        target: 'node',
        externals: {
            node: true,
        },
        resolve: {
            alias: {
                // Uncomment if you're using Ziggy.
                // ziggy: path.resolve('vendor/tightenco/ziggy/src/js'),
            },
        },
    })
```

And update your `resources/js/ssr.js` to look something like this. The specifics may vary based on your application. If you're using [Ziggy](https://github.com/tighten/ziggy), you'll want to uncomment the Ziggy stuff. (This is based on the Inertia docs, with slight modifications.)

```js
import {createSSRApp, h} from 'vue'
import {renderToString} from '@vue/server-renderer'
import {createInertiaApp} from '@inertiajs/inertia-vue3'
// import route from 'ziggy';

// This is the Lambda handler that will respond to the Sidecar invocation.
exports.handler = async function (event) {
    return await createInertiaApp({
        page: event,
        render: renderToString,
        resolve: (name) => require(`./Pages/${name}`),
        setup({app, props, plugin}) {
            // const Ziggy = {
            //     ...event.props.ziggy,
            //     location: new URL(event.props.ziggy.url)
            // }

            return createSSRApp({
                render: () => h(app, props),
            }).use(plugin).mixin({
                methods: {
                    // Pull the ziggy config off of the event.
                    // route: (name, params, absolute, config = Ziggy) => route(name, params, absolute, config),
                },
            })
        },
    });
}
```

### Ziggy (Optional)

If you are using Ziggy, you'll need to pass the Ziggy information along to your Lambda. You can do that by adding the following to your
`HandleInertiaRequests` middleware. 

```php
class HandleInertiaRequests extends Middleware
{
    public function share(Request $request)
    {
        return array_merge(parent::share($request), [
            'ziggy' => (new Ziggy)->toArray()
        ]);
    }
}
```

