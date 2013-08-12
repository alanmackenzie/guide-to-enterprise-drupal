guide-to-enterprise-drupal
==========================

Patterns and Anti-Patterns for Enterprise Drupal Development.



# Anti-Patterns

## Tactical Anti-Patterns 

#### Forgetting to filter the output of ```variable_get()```.

Even if you are capturing the value of this persistant variable in ```$conf[]``` or setting it via ```variable_set()``` in an update hook it is best to code defensively in case a system settings form is added is added later on.

__Wrong Example:__
```php
print '<a href="' . variable_get('marketing_footer_link') . '" rel="external">';
```

_Right Example:_
```php
print '<a href="' . check_url(variable_get('marketing_footer_link')) . '" rel="external">';
```




