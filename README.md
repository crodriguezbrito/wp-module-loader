<a href="https://newfold.com/" target="_blank">
    <img src="https://newfold.com/content/experience-fragments/newfold/site-header/master/_jcr_content/root/header/logo.coreimg.svg/1621395071423/newfold-digital.svg" alt="Newfold Logo" title="Newfold Digital" align="right" 
height="42" />
</a>

# WordPress Module Loader
[![Version Number](https://img.shields.io/github/v/release/newfold-labs/wp-module-loader?color=21a0ed&labelColor=333333)](https://github.com/newfold/wp-module-loader/releases)
[![License](https://img.shields.io/github/license/newfold-labs/wp-module-loader?labelColor=333333&color=666666)](https://raw.githubusercontent.com/newfold-labs/wp-module-loader/master/LICENSE)

This loader instantiates Newfold WordPress Modules inside our WordPress Plugins.

* <a href="#newfold-wordpress-modules">What are modules?</a>
* <a href="#creating--registering-a-module">Creating/registering modules</a>
* <a href="#installing-from-our-satis">Installing from our Satis</a>
* <a href="#local-development">Local development notes</a>
* <a href="#understanding-the-module-lifecycle">Understanding the module lifecycle</a>
* <a href="#module-loader-api">Module Loader API</a>

## Newfold WordPress Modules

Newfold WordPress Modules are PHP packages intended to be installed in WordPress Plugins via composer from our Satis 
registry.

Modules essentially function as WordPress Plugins we can reuse in Newfold products and control programmatically.

Modules can be **required/forced, optional, hidden** and **can be toggled** by code and (sometimes) by users.

## Creating & Registering a Module

Modules will eventually be created from templates, but for now here are some key things to know.

* Modules should contain a `bootstrap.php` file that should get autoloaded by Composer. Functionality should load 
  from `/includes`.
* Modules are loaded on the `after_setup_theme` hook with a priority of `100`.
* Module registration should tap the `plugins_loaded` hook.
* If a plugin registers a dependency injection container, then that container will be accessible to the 
  registered modules. If the plugin doesn't register a container, an empty container will be passed to the module.

### Module Registration

Below is an example of how to register a module within the `bootstrap.php` file:

```php
<?php

use function NewfoldLabs\WP\ModuleLoader\register;

if ( function_exists( 'add_action' ) ) {

	add_action(
		'plugins_loaded',
		function () {

			register(
				[
					'name'     => 'sso',
					'label'    => __( 'SSO', 'endurance' ),
					'callback' => function () {
						require __DIR__ . '/includes/sso.php';
					},
					'isActive' => true,
					'isHidden' => true,
				]
			);

		}
	);

}

```

### Installing from our Satis

Our modules are sourced from our 3rd-party package repository (Satis).

#### 1. Make sure to register our repository in the `composer.json`

Via command line: `composer config repositories.newfold composer https://newfold-labs.github.io/satis/`

OR

```json
{
  "repositories": [
    {
      "type": "composer",
      "url": "https://newfold-labs.github.io/satis/",
      "only": [
        "newfold-labs/*"
      ]
    }
  ]
}
```

#### 2. `composer require [satis-package-identifier]`

#### 3. `composer install`

### Local Development

When working on modules locally:

1. Clone the module repository somewhere on the filesystem, i.e. `/wp-content/modules/wp-module-awesome`.
2. Modify the sourcing-plugin's `composer.json` to reference that directory as a `path` in the repositories array (above the satis reference).
```json
{
  "repositories": [
    {
      "type": "path",
      "url": "../../modules/wp-module-awesome",
      "options": {
        "symlink": true
      }
    },
    {
      "type": "composer",
      "url": "https://newfold-labs.github.io/satis/",
      "only": [
        "newfold-labs/*"
      ]
    }
  ]
}
```
3. In the `require` declaration, change the version to `@dev`
4. `composer install`

## Understanding the module lifecycle

### How It Works

0. During plugin release, a `composer install` is run, creating autoloader files and pulling in composer 
   dependencies -- which include Newfold modules.
1. A request is made to WordPress, firing Core hooks.
2. The plugin containing modules is loaded during `do_action('plugins_loaded')`. WordPress loads plugins alphabetically.
3. In the plugin, the composer autoloader is required and executes. This isn't attached to an action hook, but is effectively running during `plugins_loaded`.
4. Each module defines a `bootstrap.php` that is explicitly set to autoload, so when the main plugin's autoloader fires, each module's bootstrap.php is loaded -- again outside the hook cascade, but these files are effectively run during `plugins_loaded`.
5. In the `boostrap.php` for each module, the module is registered with the module loader module using 
   `NewfoldLabs\WP\ModuleLoader\Module\make()`. Most modules should be registered in `do_action('plugins_loaded')` 
   and before the `do_action('init')` hook.
6. In `newfold-labs/wp-module-loader`, the loader runs on `do_action('after_theme_setup')` with a priority of 
   `100`.
7. Any code in a module that is instantiated via `bootstrap.php` can now access the WordPress Action Hook system, starting with `init`.

#### Hooks available to modules:
* `init` (first available)
* `wp_loaded`
* `admin_menu`
* `admin_init`
* etc

#### Hooks not available to modules
* `plugins_loaded`
* `set_current_user`
* `setup_theme`
* `after_theme_setup`

## Module Loader API

The following functions are namespaced with `NewfoldLabs\WP\ModuleLoader`.

### register( $attributes )
 
> Register a new module.
>
> Required attributes:
>  - name (string) - The internal module name; should be lowercase with dashes.
>  - label (string) - The user-facing module name
>  - callback (callable) - The callback that kicks off the module's functionality.
>  - isActive (bool) - Whether the module defaults to active.
>  - isHidden (bool) - Whether the module should be hidden from users in the UI.


### unregister( $name )
 
> Unregister a module. The `$name` is the internal module name.

### activate( $name )

> Activate a module by name.

### deactivate( $name )

> Deactivate a module by name.

### isActive( $name )

> Check if a module is active by name.

### container( $container )

> Register a container that should be shared with all modules. 
> 
> Currently, the container must be an instance of [`WP_Forge\Container\Container`](https://github.com/wp-forge/container).
> 
> A container should be registered within the WordPress plugin and should be done like this:
> 
> ```php
> use WP_Forge\Container\Container;
> use function NewfoldLabs\WP\ModuleLoader\container as setContainer;
> 
> $container = new Container();
> 
> // Configure container here as needed
> 
> setContainer( $container );
> ```
