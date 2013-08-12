Guide to Enterprise Drupal
==========================

Patterns and Anti-Patterns for Enterprise Drupal Development.

# Project Patterns

### Drupal as a packaged product.

### Go "All In".

### Migrating to Drupal.

Drupal has an excellent [migration framework](https://drupal.org/project/migrate), you should steer well clear of any traditional [ETL](http://en.wikipedia.org/wiki/Extract,_transform,_load) process.

### Drupal People.

### Managing Complexity.

# Back-end Patterns

## Strategic Patterns

### Write-through caching.

### Configuration management workflow.

## Tactical Patterns.

### Project Directory Layout.

### Use Entity Field Query.

### Use Entity Metadata Wrapper.

### Managing and capturing configuration via persistant variables.

### Deploying changes via update hooks.

### Writing good drush commands at scale.

#### Memory issues and ```drupal_static_reset()```.

# Anti-Patterns

## Tactical Anti-Patterns

### Forgetting to filter the output of ```variable_get()```.

Even if you are capturing the value of this persistant variable in ```$conf[]``` or setting it via ```variable_set()``` in an update hook it is best to code defensively in case a system settings form is added is added later on.

__Wrong Example:__
```php
print '<a href="' . variable_get('marketing_footer_link') . '" rel="external">';
```

_Right Example:_
```php
print '<a href="' . check_url(variable_get('marketing_footer_link')) . '" rel="external">';
```

### Abusing ```drupal_goto()```.

### Using ```exit()``` instead of ```drupal_exit()```.

### Using ```static $cache``` instead of ```drupal_static()```.

# Front-end Patterns

## Strategic Patterns

### SMACCS.

## Strategic Anti-Patterns

## Tactical Patterns

## Tactical Anti-Patterns

### Not removing 'dead' CSS supplied by contrib modules.

# Migration Patterns

## Strategic Patterns

### Breaking Migrations into smaller chunks.

# Credits

## Authors

* Alan MacKenzie

## Inspiriation

* Django Beatty
* Sankatha Bamunuge
