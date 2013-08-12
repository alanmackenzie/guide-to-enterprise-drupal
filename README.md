Guide to Enterprise Drupal
==========================

Patterns and Anti-Patterns for Enterprise Drupal Development.

# General 
# Back-end Patterns

## Strategic Patterns

### Write-through caching.

### Configuration management workflow.

## Tactical Patterns.

### Use Entity Field Query.

### Use Entity Metadata Wrapper.

### Managing and capturing configuration via persistant variables.

### Deploying changes via update hooks.

### Writing good drush commands at scale.

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

# Front-end Patterns

## Strategic Patterns

### SMACCS.

# Credits

## Authors & Technical Review

* Alan MacKenzie

## Inspiriation

* Django Beatty
* Sankatha Bamunuge
