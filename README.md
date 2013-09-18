Guide to Enterprise Drupal Development
======================================

Patterns and Anti-Patterns for Enterprise Drupal Development.

External contribution is welcome and encouraged.

# Project Patterns

### Drupal as a packaged product.

### Go "All In".

### Migrating to Drupal.

Drupal has an excellent [migration framework](https://drupal.org/project/migrate), you should steer well clear of any traditional [ETL](http://en.wikipedia.org/wiki/Extract,_transform,_load) process.

### Drupal People.

### Managing Complexity.

### Developer workflow.

* Writing good commit messages.
* Correct use of revision control.
* Feature branching.

### Inline documentation.

* Documentation needs to be living and breathing.
* Revision control.
* Tools.
* self documenting.

### Teach everybody Drupal

* High bandwidth conversations.
* Sometimes a speedy "Drupal solution" is worth much more than a tardy completely custom one.

Teach the non technical parts of you team the basics of Drupal. If they have an understanding of what a content type, node, taxononmy term or view is they can begin to follow (to some degree) technical discussions they may be part of.

Get the same people using the Drupal admin UI as much as possible. Extend it where possible instead of replacing it with a tamed down admin panel.

The Admin views module is an good starting point.

# Back-end Patterns

## Strategic Patterns

### Module weight.

* example.
* system, contrib, customisation.

```php
/**
 * Implements hook_install().
 */
function bbcgf_workbench_alters_install() {
  db_query("UPDATE {system} SET weight = 1 WHERE name = 'project_namespace_module_name'");
}
```

### Write-back caching.

* responsible.

### Configuration management workflow.

* Why? Idempotent, multi environment, local develop builds in sync, human error.

* Deploying changes via update hooks.
* Maximum number of update hooks per module.
* "Finished performing updates." Is a lie.
* Common tasks, enable, disable, uninstalling modules, reverting features.
* Using db_merge() to update configuration.

### Building a full site

* Devel generate.
* use migrated/real data where possible.

### Transactions

* keep it simple, transactions inside transactions.
* db_merge(), for rows with low contention it is great.

```bash
# audit-drupal-transactions.sh

# @note This can throw false positives.

for i in $(egrep -c -R 'db_insert|db_delete|db_update' --exclude-dir=*\.install  * | egrep -v "(0|1)$");
do
  echo ${i%%:*}
  egrep -C 7 'db_insert|db_delete|db_update' ${i%%:*}
done
```

### Multisite v Domain / Content wheel / distibuted site

* Avoid where possible if it cant be handled in code.
* d2d migrate.
* services/feeds.

### Reverse engineering

Good programmers read the documentation, great programmers read the code.

* Nearly everything has been done before.
* Examine the database.
* Devel query log.
* Inspecting alter hooks.
* APM.
* Reflection API.

```php
// Views has a fairly involved architecture, it may be quicker just to look
// for a method that is suitably named for the task we're trying to achieve.
// @see http://php.net/manual/en/book.reflection.php

$views = views_get_all_views();

foreach (views_get_all_views() as $view) {
  foreach ($view->display as $display) {

    $class = new ReflectionClass(get_class($display));
    dpm($class->getMethods(), get_class($display));

    // Do something.
  };
}

```

## Strategic Anti-Patterns

### Failing to plan for UTF-8/Translation.

### Module explosion.

* Embrace composable modules when the pay off is worth it.
* If using prescriptive modules pick one of the options.
* Some times its easier just to implement something directly.

### "Black Boxing"

* Review all contrib code.
* UUID example, adds an extra property to every entity :(.

## Tactical Patterns.

### Project Directory Layout.

```
docroot
|-- includes
|-- modules
|-- profiles
|-- scripts
|-- sites
|   |-- all
|   |   |-- drush
|   |   |-- libraries
|   |   |-- modules
|   |   |   |-- contrib
|   |   |   |-- custom
|   |   |   |-- dev-tools
|   |   |   |-- features
|   |   |   |-- migration
|   |   |   `-- patched
|   |   `-- themes
|   |       |-- contrib
|   |       |-- custom
|   |       `-- patched
|   |-- example.com
|   |   |-- files
|   |   `-- inc
|   `-- default
`-- themes
```

Application code is placed in a sub-directory (docroot in this example) to allow for files that should remain private such as vagrant builds, documentation, release notes and shell scripts.

Never touch anything above the sites directory of your Drupal installation without a very good reason. Drupal can take you very far without the need to hack core.

The drush directory allows for command files to be added to the repository in sites/all/drush rather than placed ad-hoc in ```${HOME}``` directories in different environments.

The modules directory is further divided into separate sub-directories, this makes reviewing the health of any given Drupal project much simpler. A healthy project will have few modules in contrib or patched and the majority of the sites complexity in contrib.

Modules in the patched directory should always have any patches applied to them checked into the root of that modules directory.

When moving modules around the directory structure you will need to use the [registry_rebuild](https://drupal.org/project/registry_rebuild) drush command. Drupal does not expect modules to be moved around underneath it.

Placing all your development tools into a single directory allows your build system to disable them in a single command and for developers to enable them all in a similar fashion. Doing this will avoid the reasonably common mistake of a developer forgetting to remove a call to ```dpm()``` or ```kpr()``` in his or her code and that code reaching production undetected - because production was the only environment with the devel module disabled.

```bash
# disable-development-modules.sh

# @note We don't have to worry about spaces in the module names.

# @note Submodules such as views_ui do not need to be moved into the
# dev-tools directory, you can either use a symlink to the module
# itself or a text file with the matching file name.

for PATH in $(find docroot/sites/all/modules/dev-tools -maxdepth 1)
do
  MODULE=$(basename "${PATH}")
  drush -y pm-disable "${MODULE}"
done
```

### Use Entity Field Query (EFQ).

* ORM on steriods, very powerful abstraction.
* Future proof?

### Use Entity Metadata Wrapper (EMW).

* Reduce bugs, tedious entity loading, language code.
* Concise, much easiser to read.
* Future proof?

### Views, total_rows, pagination and count().

TODO:

Mention [method chaining](http://en.wikipedia.org/wiki/Method_chaining)
Lazy load.
value() v raw().

### Managing and capturing configuration via persistant variables.

* Simplest way is via $conf[].
* Find active persitant variables with drush, not inactive ones though (shell script).
* Can put comments next to code.
* Strongarm is too complex.

### Writing good drush commands at scale.

* Q API.
* 
### Memory issues and ```drupal_static_reset()```.

### Use a project namespace.

* Be consistant.
* fields.
* modules, module grouping, persistant variable names.
* underscores means private function, "this is likely to change".

### Using ```hook_views_form_substitutions()``` to cache partially dynamic rendered output.

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

### Abusing ```variable_set()```.

* ```cache_bootstrap```
* deadlock.

### Unsetting form elements.

* Using '#access' instead of unset().

### Abusing ```drupal_goto()```.

### Using ```exit()``` instead of ```drupal_exit()```.

### Using ```static $cache``` instead of ```drupal_static()```.

### ```views_get_view()``` and static caching.

### Destroying data in the ```global $user``` object.

### Hardcoding values.

Using hardcoded values make code harder to read and refactor. Be kind to your reviewer and use the [constants](https://api.drupal.org/api/drupal/constants/7) that Drupal supplies.

__Wrong Example (1):__

```php
global $user;
$node = new stdClass();
$node->title = t('Example title');
$node->type = "example_bundle";
$node->language = 'und';
$node->uid = $user->uid;
$node->status = 1;
$node->promote = 1;
$node->comment = 0;
```

_Right Example (1):_
```php
global $user;
$node = new stdClass();
$node->title = t('Example title');
$node->type = "example_bundle";
$node->language = LANGUAGE_NONE;
$node->uid = $user->uid;
$node->status = NODE_PUBLISHED;
$node->promote = NODE_PROMOTED;
$node->comment = COMMENT_NODE_OPEN;
```

__Wrong Example (2):__
```php

if ($file->status == 1) {
  return;
}
```

_Right Example (2):_
```php

if ($file->status == FILE_STATUS_PERMANENT) {
  return;
}
```

### Being language oblivious.

### Forgetting to check your returns.

### Using ```time()``` instead of ```REQUEST_TIME```.

### Trusting the block cache.

TODO:

* Hitting memcache when you dont need to.
* node_save() clearing the block cache.

### Deleting modules from revision control before uninstalling them.

### Huge cache objects and ```MAX_PACKET_SIZE```

# Front-end Patterns

## Strategic Patterns

### SMACCS.

* Block element modifier.

### Compass & SASS.

* High res.
* extends.
* file structure.
* Documenting css
* Drupal centric css structure.

### Use Drupal markup.

Make use of the default markup provided by Drupal where possible. It provides a standardised setup that allows for rapid development and avoids the need for large numbers of template files / theme overrides.

This does not mean you cannot cut down the often bloated Drupal markup, fewer nodes in the DOM is of course a good thing, especially when in comes to JavaScript. Instead be intelligent about it, just because you don't need an extra wrapper at this point does not mean it will not be useful for styling, or hooking into with JavaScript, in the future.

The same goes for stripping out the extra classes and ids added by Drupal, remove them for a reason rather then for the sake of it. Additionally, Contrib modules sometimes make assumptions about the markup available.

### JavaScript.

All your JavaScript should be wrapped in an outer context:
```js
(function ($) {
  // All your code here
})(jQuery);
```

Do not use jQuery(document).ready(), instead use Drupal Behaviours. These behaviours will allow your JavaScript to act on any new element added to the page.

In the case that you only want your JavaScript to run the first time make use of .once(). Be sure to namespace the class you are adding.

Use Drupal.t() on the strings in your JavaScript, as you would with PHP, so that they are translatable.

#### Performance

Make your jQuery selectors as specific as possible.

Use context.find('.el') over $('el') where possbile.

* Firing events all the time on window resize etc.

### Theme structure

Break up you template.php file to keep it managable in larger builds:
```php
$path = drupal_get_path('theme', 'bbcw_goodfood');
include_once $path . '/inc/preprocess.inc';
include_once $path . '/inc/theme.inc';
include_once $path . '/inc/menu.inc';
include_once $path . '/inc/form.inc';
```

Structure you templates folder to make it managable.
```
templates
|-- blocks
|-- comments
|-- fields
|-- forms
|-- html
|-- modules
|-- nodes
|-- pages
|-- panels
|-- regions
|-- user
|-- views
```
You make take this further and add sub folders where there are many templates, for example per view folders under views.

When creating custom modules specific to your site you should treat these as if they will be made open source, this means keeping the modules template files in module directory, as with any other contrib module. If your template is closely tied to your specfic theme it is still good practice to provide a generic template with your module.

### CSS testing.

* [https://drupal.org/project/styleguide](Styleguide module)
* Hardy.

## Strategic Anti-Patterns

### JavaScript.

Do not be tempted to rearrange the DOM via JS for responsive layouts. It causes page flicker as the JS loads, especially on mobile devices.

## Tactical Patterns

### Know your render functions.

* do not use render(), use drupal_render() & drupal_render_children().
* render(), renders the first element only (always shows() it first).
* drupal_render(), iterates over every element and renders it.
* drupal_render_children() only renders the child elements.

## Tactical Anti-Patterns

### Not removing 'dead' CSS supplied by contrib modules.

### Being function shy.

* l()
* url()
* theme_list()
* formate_date()


### Push complexity into modules.

## Tactical Anti-patterns.

### Failing to use ```t()``` in your js.

# Full Stack/Drupal DevOps.

## Full Stack Patterns.


### Continuous Deployment

* Feature toggle pattern, http://en.wikipedia.org/wiki/Feature_toggle
* Complexity rises in a non-linear fashion, releasing early and often releases this 'pressure valve'.
* Update hooks should clear exact caches.
* hard v soft deployments. Naming convention for OPs, stakeholders and build system.
* https://drupal.org/project/registry_rebuild
* gentle cache clear/gentle RR?

```sh
#!/bin/sh

if [ $(echo "$deployedtag" | egrep '\-hard$')  ]
then
  echo 'Hard deploy, running registry rebuild.'
  drush @$drush_alias rr
fi
```

### Clearing individual caches.

* Hook drush_cache_clear.

### Cron Jobs.

* HA.
* Locks (cache_bootstrap?).
* unix cron keep it simple.

### Using strace to debug black boxes.

* strace example, remember -s.
* system call over head.

### Using memkeys to debug memcache.

* https://github.com/tumblr/memkeys

### Keeping your logs readable.

* https://drupal.org/project/syslog_advanced


## Full Stack Anti-Patterns.

### Abusing query parameters.

* Page caching
* Back-end stampede
* Do redirects in .htaccess to avoid bootstrapping Drupal

### Using Memcache as the cache backend for ```cache_form```.

Drupal's Form API uses a token unique to each user per form to prevent [CSRF](http://en.wikipedia.org/wiki/CSRF).

The downside to this is that ```cache_form```, the place where the CSRF busting token is stored, should always use persistant storage. Memcache uses a [LRU](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used) algorithm to evict unpopular cache objects, making it incompatible with ```cache_form```.

>> "An unrecoverable error occurred. This form was missing from the server cache. Try reloading the page and submitting again."

If you're getting intermittant reports about the above error message it's likely that you're using non-persistant storage for the ```cache_form``` backend.

### Unintelligent increases in PHP's ```memory_limit```.

Increasing PHP's ```memory_limit``` directive beyond 128M should be treated as very suspect.

* Increasing memory size decreases the number of workers available.
* blah blah concurrency.
* https://drupal.org/project/role_memory_limit
* Prefork doesn't free memory: [MaxRequestPerChild](http://httpd.apache.org/docs/2.2/mod/mpm_common.html#maxrequestsperchild)

### Not designing with page caching in mind.

* Dialogue
* bring in engineers early.

### Failing to use a message queue.

* Connecting components.
* Decoupling.
* Handling burts.
* Async.

### Mixing computation and presentation.

* memory_limit and max_execution_time exist for a reason.
* cron is not the answer, timeouts, server resource usage impact on end user, easier to scale separate logical components

# Example Drupal Configuration

## Production Configuration.

```php

/**
 * Anti-DDoS image preset security token.
 *
 * @note Disabling this does not increase your cache hit ratio, the itok
 * query parameter is added to the images anyway, the option below only
 * controls whether the itok parameter is verified or not. The good news
 * is that the itok parameter will stay the same provided you do not
 * change the private key or hash salt for your Drupal instance.
 *
 * @see image_style_path_token()
 */
// $conf['image_allow_insecure_derivatives'] = TRUE;

 /**
  * Set session lifetime (in seconds), i.e. the time from the user's last visit
  * to the active session may be deleted by the session garbage collector. When
  * a session is deleted, authenticated users are logged out, and the contents
  * of the user's $_SESSION variable is discarded.
  *
  * @note Drupals default value for this setting is 200000 seconds or 2.3 days,
  * this is a sensible default but one which likely hurts your user engagement
  * by forcing users to repeatedly login. Below we have upped this value to 3
  * weeks.
  *
  * @warning Depending on the size and activity of your user base you may need
  * to monitor the size of your sessions table.
  */
ini_set('session.gc_maxlifetime', 1814400);

/**
 * Set session cookie lifetime (in seconds), i.e. the time from the session is
 * created to the cookie expires, i.e. when the browser is expected to discard
 * the cookie. The value 0 means "until the browser is closed".
 *
 * @see The note for session lifetime.
 *
 * @note The default value is only 2000000 seconds or 23 days. Below we have
 * upped this value to 90 days.
 *
 * @note If you want conditional lifetimes use: https://drupal.org/project/remember_me
 */
ini_set('session.cookie_lifetime', 7776000);

/**
 * One-time login link.
 *
 * @note If you're using this for email verification you're likely hurting user
 * engagement with the default value of 1 day. Below we've upped the value to
 * 3 days.
 */
$conf['user_password_reset_timeout'] = 259200;
// TODO Double check the registration validation email uses this.

/**
 * Flood prevention.
 *
 * @note You may wish to relax these during the launch period of any new
 * site and gradually tighten them up as you understand your user base more.
 *
 * @see user_failed_login_user_window
 * @see user_failed_login_ip_window
 */
$conf['user_failed_login_user_limit'] = 7;
$conf['user_failed_login_ip_limit'] = 50;

/**
 * Akamai.
 *
 * @note If a complete list of reverse proxies is not available in your
 * environment (for example, if you use a CDN) you may set the
 * $_SERVER['REMOTE_ADDR'] variable directly in settings.php.
 * Be aware, however, that it is likely that this would allow IP
 * address spoofing unless more advanced precautions are taken.
 *
 * @note We use Akamai's KONA Site Defender to prevent attackers hitting
 * origin directly.
 *
 * @warning This piece of code must come after Acquia's own reverse
 * proxy configuration to work.
 */

if (isset($_SERVER['HTTP_TRUE_CLIENT_IP'])) {
  $_SERVER['REMOTE_ADDR'] = $_SERVER['HTTP_TRUE_CLIENT_IP'];
}

/**
 * Caching
 *
 * @note cache_lifetime is a global lock that is only useful if you're running
 * cron extremely regularly or have it set to a very high value making it useless
 * for most large Drupal sites that do event based purging or are highly dynamic.
 * Setting cache_lifetime to 0 makes debugging caching issues much easier.
 */
$conf['cache_lifetime'] = 0;

/**
 * Cron
 *
 * Rather than have cron run in an ad-hoc manner at the end of a request it is
 * a better idea to have it run at set times through drush and log all of the
 * output. This makes debugging easier and the system more predictable.
 *
 * @see drupal_page_footer()
 */
$conf['cron_safe_threshold'] = 0;

/**
 * Field API
 *
 * Once you delete a field the actual data is not immediately removed, you will
 * see tables name field_deleted_data_n and field_deleted_revision_n. BY default
 * field cron will remove 10 items from these tables per batch run. If you have
 * a large number of entities completely removing a field could take years.
 *
 * @note You can see which fields are currently in the process of being deleted
 * by running a query such as this:
 *   select field_name, type from field_config where deleted = 1;
 *
 * @see field_cron()
 * @see field_purge_batch()
 */
$conf['field_purge_batch_size'] = 10000;

/**
 * Redirect
 *
 * @note The redirect module has a nasty habit of creating redirect loops
 * when allowed to automatically create redirects. Think carefully before
 * enabling this.
 *
 * @note If you're disabling the automatic creation of redirects you probably
 * do not want pathauto to modify the alias of the target entity on update.
 */
$conf['redirect_auto_redirect'] = FALSE;
$conf['pathauto_update_action'] = FALSE;

// TODO: Devel error_handler settings.

/**
 * Locking API
 *
 * TODO: Rationale.
 */
$conf['lock_inc'] = 'sites/all/modules/contrib/memcache/memcache-lock-code.inc';


```

## Local Developer Configuration.

```php
/**
 * Views configuration.
 */
$conf['views_ui_show_sql_query'] = TRUE;
$conf['views_ui_show_sql_query_where'] = 'above';
$conf['views_ui_show_performance_statistics'] = TRUE;

// @note This makes developing plugins much more pleasant, grep the views code base
// for calls to vpr() and see for yourself.
$conf['views_devel_output'] = TRUE;

/**
 * Proxy settings.
 */
$conf['proxy_server'] = 'www-proxy.example.com';
$conf['proxy_port'] = '80';

// drupal_http_request() does not support proxying HTTPS.
define('ACQUIA_DEVELOPMENT_NOSSL', TRUE);

/**
 * The stage_file_proxy module prevents you from having to download the entire files
 * directory from production.
 */
$conf['stage_file_proxy_origin'] = 'http://www.example.com';
$conf['stage_file_proxy_origin_dir'] = 'sites/example.com/files';

/**
 * Devel settings.
 *
 * @note Drupal's default error handler will silence xdebug.
 */
$conf['devel_error_handlers'] = array(0 => 0);

```

## ```drushrc.php``` Configuration.

```php

// @TODO.

```
# Patterns in Drupal

## General

* [Presentation Abstraction Control (PAC)/Hierarchical MVC](http://en.wikipedia.org/wiki/Presentation-abstraction-control)
* [Entity Attribute Value (EVA)](http://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model)
* [Event Condition Action (ECA)](http://en.wikipedia.org/wiki/Event_condition_action)
* [Post/Redirect/Get (PRG)](http://en.wikipedia.org/wiki/Post/Redirect/Get)
* [Front Controller](http://en.wikipedia.org/wiki/Front_controller)

## Plugins

* [Strategy Pattern](http://en.wikipedia.org/wiki/Strategy_pattern)
* [Dependency Injection (DI)](http://en.wikipedia.org/wiki/Dependency_injection)
* 
## Hooks

* [Aspect Oriented Programming (AOP)](http://en.wikipedia.org/wiki/Aspect_oriented_programming)
* [Observer](http://en.wikipedia.org/wiki/Observer_pattern)


# Credits

## Authors

* Alan MacKenzie

## Inspiriation

* Django Beatty
* Sankatha Bamunuge

### Scrap/TODO

* $form_state['storage'];
* tree -L 3 -d --charset=ascii
* egrep 'node/[0-9]+|user/[0-9]+|comment/[0-9]+|taxonomy/term/[0-9]+'
* Migration patterns, forceutf8, breaking migrations into chunks.
* Traditional domain modelling anti-pattern, dont tighly couple code to content.
