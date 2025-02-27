# Plugin Settings & Configuration

## Introduction

There are two ways to configure plugins - with backend settings forms and with configuration files. Using database settings with backend pages provide a better user experience, but they carry more overhead for the initial development. File-based configuration is suitable for configuration that is rarely modified.

## Database settings

You can create models for storing settings in the database by implementing the `SettingsModel` behavior in a model class. This model can be used directly for creating the backend settings form. You don't need to create a database table and a controller for creating the backend settings forms based on the settings model.

The settings model classes should extend the Model class and implement the `System.Behaviors.SettingsModel` behavior. The settings models, like any other models, should be defined in the **models** subdirectory of the plugin directory. The model from the next example should be defined in the `plugins/acme/demo/models/Settings.php` script.

```php
<?php namespace Acme\Demo\Models;

use Model;

class Settings extends Model
{
    public $implement = ['System.Behaviors.SettingsModel'];

    // A unique code
    public $settingsCode = 'acme_demo_settings';

    // Reference to field configuration
    public $settingsFields = 'fields.yaml';

    // Optional - sets the TTL for the settings cache
    public $settingsCacheTtl = 3600;
}
```

The `$settingsCode` property is required for settings models. It defines the unique settings key which is used for saving the settings to the database.

The `$settingsFields` property is required if are going to build a backend settings form based on the model. The property specifies a name of the YAML file containing the form fields definition. The form fields are described in the [backend forms](../backend/forms) article. The YAML file should be placed to the directory with the name matching the model class name in lowercase. For the model from the previous example the directory structure would look like this:

```treeview
plugins/
`-- acme/
    `-- demo/
        `-- models/
            |-- settings/        # Model files directory
            |   `-- fields.yaml  # Model form fields
            `-- Settings.php     # Model script
```

You may optionally add a `$settingsCacheTtl` property to your settings model if you wish to change the length of time that your settings will remain cached for. By default, settings in your settings model will remain cached for up to 24 minutes. To disable caching, you may set this to `0` or `false`.

Settings models [can be registered](#backend-settings-pages) to appear on the **backend Settings page**, but it is not a requirement - you can set and read settings values like any other model.

### Writing to a settings model

The settings model has the static `set` method that allows to save individual or multiple values. You can also use the standard model features for setting the model properties and saving the model.

```php
use Acme\Demo\Models\Settings;

...

// Set a single value
Settings::set('api_key', 'ABCD');

// Set an array of values
Settings::set(['api_key' => 'ABCD']);

// Set object values
$settings = Settings::instance();
$settings->api_key = 'ABCD';
$settings->save();
```

### Reading from a settings model

The settings model has the static `get` method that enables you to load individual properties. Also, when you instantiate a model with the `instance` method, it loads the properties from the database and you can access them directly.

```php
// Outputs: ABCD
echo Settings::instance()->api_key;

// Get a single value
echo Settings::get('api_key');

// Get a value and return a default value if it doesn't exist
echo Settings::get('is_activated', true);
```

### Initializing default values

In order to provide default values for settings just implement the `initSettingsData()` method; which will be called when instantiating the Settings model instance. These default values will be used when there are no values in the database for the settings. See below for an example implementation:

```php
class Settings extends Model
{
    public function initSettingsData()
    {
        $this->files_per_post = 5;
        $this->file_extensions = 'jpg, jpeg, png, gif, webp';
    }
}
```

## Backend settings pages

The backend contains a dedicated area for housing settings and configuration, it can be accessed by clicking the <strong>Settings</strong> link in the main menu. The Settings page contains a list of links to the configuration pages registered by the system and other plugins.

### Settings link registration

The backend settings navigation links can be extended by overriding the `registerSettings` method inside the [Plugin registration class](registration#registration-file). When you create a configuration link you have two options - create a link to a specific backend page, or create a link to a settings model. The next example shows how to create a link to a backend page.

```php
public function registerSettings(): array
{
    return [
        'location' => [
            'label'       => 'Locations',
            'description' => 'Manage available user countries and states.',
            'category'    => 'Users',
            'icon'        => 'icon-globe',
            'url'         => Backend::url('acme/user/locations'),
            'order'       => 500,
            'keywords'    => 'geography place placement'
        ]
    ];
}
```

> **NOTE:** Backend settings pages should [set the settings context](#setting-the-page-navigation-context) in order to mark the corresponding settings menu item active in the System page sidebar. Settings context for settings models is detected automatically.

The following example creates a link to a settings model. Settings models is a part of the settings API which is described above in the [Database settings](#database-settings) section.

```php
public function registerSettings(): array
{
    return [
        'settings' => [
            'label'       => 'User Settings',
            'description' => 'Manage user based settings.',
            'category'    => 'Users',
            'icon'        => 'icon-cog',
            'class'       => 'Acme\User\Models\Settings',
            'order'       => 500,
            'keywords'    => 'security location',
            'permissions' => ['acme.users.access_settings']
        ]
    ];
}
```
#### Properties

The optional `category` parameter is used by the backend settings page to organize links. If a category is not provided, the new link will be added to the `Misc` category.  
You can define your own link category or use one of the default provided by [`SettingsManager` constants](../../api/System/Classes/SettingsManager#constants).

The optional `keywords` parameter is used by the settings search feature. If keywords are not provided, the search uses only the settings item label and description.

### Setting the page navigation context

Just like [setting navigation context in the controller](../backend/controllers-ajax#setting-the-navigation-context), Backend settings pages should set the settings navigation context. It's required in order to mark the current settings link in the System page sidebar as active. Use the `System\Classes\SettingsManager` class to set the settings context. Usually it could be done in the controller constructor:

```php
public function __construct()
{
    parent::__construct();

    [...]

    BackendMenu::setContext('Winter.System', 'system', 'settings');
    SettingsManager::setContext('You.Plugin', 'settings');
}
```

The first argument of the `setContext` method is the settings item owner in the following format: **author.plugin**. The second argument is the setting name, the same as you provided when [registering the backend settings page](#settings-link-registration).

## File-based configuration

Plugins can have a configuration file `config.php` in the `config` subdirectory of the plugin directory. The configuration files are PHP scripts that define and return an **array**. Example configuration file `plugins/acme/demo/config/config.php`:

```php
<?php

return [
    'maxItems' => 10,
    'display' => 5
];
```

Use the `Config` class for accessing the configuration values defined in the configuration file. The `Config::get($name, $default = null)` method accepts the plugin and the parameter name in the following format: **Acme.Demo::maxItems**. The second optional parameter defines the default value to return if the configuration parameter doesn't exist. Example:

```php
use Config;

...

$maxItems = Config::get('acme.demo::maxItems', 50);
```

A plugin configuration can be overridden by the application by creating a configuration file `config/author/plugin/config.php`, for example `config/acme/todo/config.php`, or `config/acme/todo/dev/config.php` for an environment specific override (in this case `dev`).

> **NOTE:** In order for the config override to work, the plugin must contain a default config file (i.e. `plugins/author/plugin/config/config.php`. Even if you expect all configuration to come from the project override instead of the default configuration file it is still **highly** recommended that a default configuration file is provided as a form of documentation as to what configuration options are available to be modified on the project level.

Inside the overridden configuration file you can return only values you want to override:

```php
<?php

return [
    'maxItems' => 20
];
```

If you want to use separate configurations across different environments (eg: **dev**, **production**), simply create another file in `config/author/plugin/environment/config.php`. Replace **environment** with the environment name. This will be merged with `config/author/plugin/config.php`.

Example:

**config/author/plugin/production/config.php:**

```php
<?php

return [
    'maxItems' => 25
];
```

This will set `maxItems` to 25 when `APP_ENV` is set to **production**.
