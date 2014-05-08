Guide to Enterprise Drupal Development
======================================

Patterns and Anti-Patterns for Enterprise Drupal Development.

External contribution is welcome and encouraged.

Overwhelmed? Read the [Introduction to Software Engineering](https://github.com/alanmackenzie/intro-to-software-eng).

# License 

Creative Commons Attribution-ShareAlike 3.0.

# Project Patterns

### Drupal is already productized

>> Functionality is cheap, customisation is expensive.

* Drupal gives you a lot for free, use it.
* Delivering things ASAP lets you check your assertions with real world data. You can always improve them later on.
* Engineers need to think very carefully before embarking on large pieces of customisation to Drupal. Often this is a failure to corretly [manage complexity](http://www.cs.nott.ac.uk/~cah/G51ISS/Documents/NoSilverBullet.html).
* Traditional service/waterfall model does not play to Drupal's advantages. Develop requirements in dialogue with the business, don't let this happen in isolation.
* Let the system inform the requirements, don't try retrofitting flat HTML into Drupal. The aim is working software with a low implementation risk, not pedantry over the number of div elements.

### Gorilla Platforms v Guerrilla Platforms

* Tightly coupled code and content is a bad idea.
* Distributed, common thread instead of single monolithic platform.
* reusable, flexible.
* step down graph.

### Drupal and WEM (Web Experience Management)

* Most enterprises grew their online presence organically, this leads to a fractured user experience due to the variety of products in use and the problems integrating these system.
* Propriety systems exacerbate the issue due to licensing costs, lack of openness leads to engineering forcing negative product decisions due to implementation time. 
* Drupal is incredibly flexible and can potentially do anything. An engineer used to Drupal would not believe how unflexible propriety systems and the companies behind them can be.
* Drupal places a lot of complexity "at the edge" which makes your site massively flexible, it's not just a display layer.
* To borrow Acquia's marketting material: Commerce, Community, Content. Drupal can do it all.

* Micro case studies:

* BBCWW, shift from publishing to highly interactive experience. Paywalls, personalisation, SASS.
* Lush, Ecommerce is the heart of the business but they have tonnes of content.

// @TODO: Talk to someone at Lush who does Drupal...

### Drupal doesn't come with a moat.

* Most enterprise products come with some kind of vendor lock-in. Drupal has a healthy ecosystem of suppliers.
* Letting core and contrib do the heavy lifting and open sourcing as much custom code as possible creates a platform with very healthy long term prospects.

### Migrating to Drupal.

Drupal has an excellent [migration framework](https://drupal.org/project/migrate), you should steer well clear of any traditional [ETL](http://en.wikipedia.org/wiki/Extract,_transform,_load) process.

### Drupal People.

The Drupal community

### Developer workflow.

* Writing good commit messages.
* Correct use of revision control.
* Git flow, control over when certain features go live, ability to hot fix/cherry pick.

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
* Particular problem with hook_init()/hook_boot() and form alters.
* Letting contrib run first is a good habit to get into.

```php
/**
 * Implements hook_install().
 */
function bbcgf_workbench_alters_install() {
  db_query("UPDATE {system} SET weight = 1 WHERE name = 'namespace_module_name'");
}
```

### Write-back caching.

* With exception of the SchemaCache all caches in core use write-back.
* responsible.

### Configuration management workflow.

* Why? Idempotent, multi environment, local develop builds in sync, human error.

* Deploying changes via update hooks.
* Maximum number of update hooks per module.
* "Finished performing updates." Is a lie.
* Common tasks, enable, disable, uninstalling modules, reverting features.
* Using db_merge() to update configuration.
* Do not do anything but a trivial number of INSERTS, import from CSV etc is a bad idea.

```php

@TODO Many more example update hooks.

// Revert the facetapi section of the namespace_search_api feature.
// @note Inspecting a feature module info file will give you the keys for the
// sections in a feature module.
features_revert(array('namespace_search_api' => array('facetapi')));

// Enable the google analytics module.
// @note You must be careful as some modules use a namespace for their .module
// different from the directory they reside.
// @warning This function preforms a complete cache clear.
module_enable(array('google_analytics'));

// Disable the devel, uglifyjs and backup_migrate modules and then uninstall them.
$modules = array(
  'devel',
  'uglifyjs',
  'backup_migrate',
);

module_disable($modules);
drupal_uninstall_modules($modules);

// Enable the search-sorts block.
// @note It's a smart idea to use db_merge (INSERT or UPDATE) as the same code continue
// to run on 'dirty' local development environments.
db_merge('block')
  ->key(
    array(
      'theme' => 'namespace_theme',
      'delta' => 'search-sorts',
      'module' => 'search_api_sorts'
    )
  )
  ->fields(array(
    'title' => '<none>',
    'region' => 'search_enabled_filters',
    'status' => 1,
    'visibility' => 0,
    'pages' => '*search/*',
  ))->execute();

// Disable a number of views that come with the workbench module.
$status = variable_get('views_defaults', array());
$status['workbench_moderation'] = TRUE;
$status['workbench_current_user'] = TRUE;
$status['workbench_edited'] = TRUE;
$status['workbench_recent_content'] = TRUE;
variable_set('views_defaults', $status);
```
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

>> Good programmers read the documentation, great programmers read the code.

Lazy programmers use reverse engineering.

* Nearly everything has been done before.
* Examine the database.
* Devel query log.
* Inspecting alter hooks.
* APM.
* Reflection API.
* Objects that implement ```__toString()```, db_select, parts of migrate. etc.

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
* Lack of coding standards huge red warning.
* UUID example, adds an extra property to every entity :(.

### Getting (kind of) RESTFUL

* Seperate internal paths prefered over extra parameters. Do one thing and do it well.

e.g.

bad: node/12345/my-special-callback

good: /namespace/my-special-callback

This kind of mixing can get extremely complicated very fast and isn't performant.

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
|   |   |   `-- migration
|   |   `-- themes
|   |       |-- contrib
|   |       `-- custom
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

Patched modules should always have any patches applied to them checked into the root of that modules directory.

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

* views_get_view_results().
* Never run ```count()``` on the results set.
* Any query with a pager also has the count query run by default. If you are having performance issues with the count query part of the pager think about using TODO.
* Use ```hook_view_pre_execute()``` and set the count flag.

### Managing and capturing configuration via persistant variables.

* Simplest way is via $conf[].
* Find active persitant variables with drush, not inactive ones though (shell script).
* Can put comments next to code.
* Strongarm is too complex.
* $conf allows for per environment or special conditions.

```php
// Hide all error messages on the production environment.
if ($_ENV["AH_SITE_ENVIRONMENT"] == "prod") {
  $conf['error_level'] = 0;
}

// Cache the homepage for 3 hours and every other page for 24 hours.
if (drupal_is_front_page()) {
  $conf['page_cache_maximum_age'] = 10800;
}
else {
  $conf['page_cache_maximum_age'] = 86400;
}
```
### Using Shutdown functions

* INSERTS etc dont have to be run inline with requests.
* Workbench is a good example.

### Writing good drush commands at scale.

* Q API.
* interactive.
* log.ning correct results.
* returning correct results.
* tee, nohup.

```php
# Prints an error message and sets the exit code to 1.
drush_set_error('Test role has not been set');

# Prints an error message but does not change the exit code making it useless in shell scripts.
drush_log('Test role has not been set', 'error');
```

### Static caching, memory issues and ```drupal_static_reset()```.

* Running ```drupal_static_reset()``` inline with a request because your code has synchronisation issues is usually a code smell.
* This function is required if doing any kind of large processing in drush a drush command such as generating thousands of nodes for a load test.

### Use a project namespace.

* Be consistant.
* fields.
* modules, module grouping, persistant variable names.
* underscores means private function, "this is likely to change".
* short name
* no underscores in namespace, this keeps a clear separation between hooks and namespace.
* use separate theme namespace.

### Using ```hook_views_form_substitutions()``` to cache partially dynamic rendered output.

## Tactical Anti-Patterns

### Not leveraging the menu system.

* menu_get_object(), don't write garbage with arg().
* menu_execute_active_handler() hijacking example.

### Forgetting to filter the output of ```variable_get()```.

Even if you are capturing the value of this persistant variable in ```$conf[]``` or setting it via ```variable_set()``` in an update hook it is best to code defensively in case a system settings form is added is added later on.

__Wrong Example:__
```php
print '<a href="' . variable_get('namespace_marketing_footer_link') . '" rel="external">';
```

_Right Example:_
```php
print '<a href="' . check_url(variable_get('namespace_marketing_footer_link')) . '" rel="external">';
```

### Abusing ```variable_set()```.

* ```cache_bootstrap```
* deadlock.

### Unsetting form elements.

* Using '#access' instead of unset().

### Abusing ```drupal_goto()```.

* Do not call outside of a menu callback.
* Form submit handler issues. Use $form['redirect'].

### Using ```exit()``` instead of ```drupal_exit()```.

```exit()``` does not fire hook_exit. You must use ```drupal_exit()``` instead.

### Using ```static $cache``` instead of ```drupal_static()```.

* Use Q API and process in batches.

### Adding too many displays to a view.

When loading a view all displays are loaded, separately. This means multiple fetches to your cache backend or database.

// TODO: Verify + views core snippet/architecture?

### ```views_get_view()``` and static caching.

* Pass correct parameters.

### Destroying data in the ```global $user``` object.

* Use $account.

### Not cleaning up after module uninstalltion.

* junk left in the variable table.

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

* 'und' or LANGUAGE_NONE is usually a code smell.

### Check your returns / Use the menu system.

* node_load() can return ```FALSE```.
* If node is guarnteed to exist use ```menu_get_object``` instead.

### Using ```time()``` instead of ```REQUEST_TIME```.

```time()``` is not performant as it requires a [syscall](http://en.wikipedia.org/wiki/Syscall). You should only use ```time()``` if you require the precise time in seconds.

### Trusting the block cache.

TODO:

* Hitting memcache when you dont need to.
* node_save() clearing the block cache.
* block in code no need for cache, block in db still needs cache.
* Use block cache instead of views cache.
* Block cache scales better than views cache as it only requires one fetch from the cache bin.
* Views cache underneath the block cache is the ideal scenario.

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

### Designing with entity view modes in mind.

* Code reuse/consistency.
* Do more work config rather than in templates.
* display suite, custom view modes extra layout and formatting options.
* If you ever find yourself hiding fields ask yourself if you should be using a display.
* Don't default to the fields display handler when using views, unless it's not a representation of a single entity or combinations of fields accross multiple entities.
* Displays are for more than just visual elements. You can have a data export display:

```php
function drush_bbcgf_dev_drush_command_bbcgf_json_dump() {

  $query = new EntityFieldQuery();
  $results = $query->entityCondition('entity_type', 'node')
    ->entityCondition('bundle', 'recipe')
    ->propertyCondition('status', NODE_PUBLISHED)
    ->propertyOrderBy('created', 'DESC')
    ->range(0, 2)
    ->execute();

  $nodes = node_load_multiple(array_keys($results['node']));

  foreach ($nodes as $id => $node) {
    // Second parameter is the display mode.
    $nodes[$id] = node_view($node, 'data-export');
  }

  print drupal_json_encode($nodes);
}

```
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

# Testing

## Testing Patterns

### Tagging all test users with a ```tester``` role.

* Testing with the admin account is an anti-pattern.
* Need the ability to delete all content and comments associated with test users on cronjob/button push.
* Much better if test accounts are available to the entire team across all environments. Ad-hoc style of individuals creating them is likely to lead to more bugs.
* TODO: Write a module for the above.


### Using PHPUnit

* D8 ready. Better than simpletest.
* Hard/PhantomJs

### Continuous Testing v Monitoring
* drupal lint.

# Full Stack/Drupal DevOps.

## Full Stack Patterns.

### Continuous Deployment

* Feature toggle pattern, http://en.wikipedia.org/wiki/Feature_toggle
* Complexity rises in a non-linear fashion, releasing early and often releases this 'pressure valve'.
* Update hooks should clear exact caches.
* hard v soft deployments. Naming convention for OPs, stakeholders and build system.
* https://drupal.org/project/registry_rebuild
* gentle cache clear/gentle RR?
* TODO: Service outage notice.

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

### Caching with varnish and akamai

* Short TTL on akamai, X-Edge-Control.
* Longer TTL in varnish.
* Pages can be purge from varnish much easier than akama due to the delay.

## Full Stack Anti-Patterns.

### Abusing query parameters.

* Page caching
* Back-end stampede
* Do redirects in .htaccess to avoid bootstrapping Drupal
* [Combinatorial explosion](http://en.wikipedia.org/wiki/Combinatorial_explosion)
* https://www.varnish-cache.org/vmod/querystring

### Form token myths.

Drupal's Form API uses a token unique to each user per form to prevent [CSRF](http://en.wikipedia.org/wiki/CSRF).

UID 0 is ignored.

Public and private key.

### Using Memcache as the cache backend for ```cache_form``` or ```sessions```.

* Rewrite to apply to sessions as well.

The downside to this is that ```cache_form```, the place where the CSRF busting token is stored, should always use persistant storage. Memcache uses a [LRU](http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used) algorithm to evict unpopular cache objects, making it incompatible with ```cache_form```.

>> "An unrecoverable error occurred. This form was missing from the server cache. Try reloading the page and submitting again."

If you're getting intermittant reports about the above error message it's likely that you're using non-persistant storage for the ```cache_form``` backend.

### Unintelligent increases in PHP's ```memory_limit```.

Increasing PHP's ```memory_limit``` directive beyond 128M should be treated as very suspect.

* Increasing memory size decreases the number of workers available.
* blah blah concurrency.
* https://drupal.org/project/role_memory_limit
* Prefork doesn't free memory: [MaxRequestPerChild](http://httpd.apache.org/docs/2.2/mod/mpm_common.html#maxrequestsperchild)
* cgi req and time limit.
* safest thing to do is have a seperate editorial webserver or cluster.

### Not designing with page caching in mind.

* Dialogue
* bring in engineers early.
* Examples of drupal_message() getting caught in the cache.

### Failing to use a message queue.

* Connecting components.
* Decoupling.
* Handling burts.
* Async.

### Mixing computation and presentation.

* memory_limit and max_execution_time exist for a reason.
* cron is not the answer, timeouts, server resource usage impact on end user, easier to scale separate logical components

# Enterprise Drupal Configuration

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
 *
 * @warning Caching of redirects can break your site. You may want to disable
 * this functionality if you're creating redirects programmatically until you've
 * verified that they're correct.
 */
$conf['redirect_auto_redirect'] = FALSE;
$conf['pathauto_update_action'] = FALSE;
$conf['redirect_page_cache'] = TRUE;

/**
 * Locking API
 *
 * Memcache is considerably more performant than MySQL so it is much less likely
 * to suffer from lock contention that escaletes to deadlock.
 *
 * @warning You need to be careful memcache's LRU algorithm is not evicting lock objects.
 */
$conf['lock_inc'] = 'sites/all/modules/contrib/memcache/memcache-lock-code.inc';

/**
 * Form API
 *
 * Drupal keeps a secure token for instance of every form, this is to prevent CSRF. The tokens
 * are stored in cache_form, the standard advice is to use a database backend for storage.
 *
 * The cache object contains a serialized copy of the form and often related entities so
 * it can grow extremely large.
 *
 * By default Drupal will clear stale cache_form objects on cron run, the objects have a TTL
 * of 6 hours, below we have lowered this to 3 hours.
 *
 * @note For continous purging of stale items use: https://drupal.org/project/safe_cache_form_clear
 *
 * @warning Despite the name it's worth noting that cache_form requires presistant storage, 
 * think carefully about fail-over and LRU before you use a backend like memcache.
 */
// Small core patch required.
// https://drupal.org/node/2091511
$conf['form_cache_expiry'] = 10800;

/**
 * Fast 404.
 *
 * The contrib fast 404 module is more performant than core fast 404 functionality.
 * @see https://drupal.org/project/fast_404
 */
#$conf['404_fast_paths_exclude'] = '/\/(?:styles)\//';
#$conf['404_fast_paths'] = '/\.(?:txt|png|gif|jpe?g|css|js|ico|swf|flv|cgi|bat|pl|dll|exe|asp)$/i';

/**
 * Agrcache.
 *
 * CSS & JS aggegation can cause deadlock in the variables table under high
 * contention, agrcache prevents this from happening by building the aggregates
 * in a lazy fashion rather than inline. Using Agrcache also offers improved
 * performance as it does not rely on calls to file_exists().
 *
 * @warning Extra configuration is needed to make agrcache work with fast_404.
 */
$conf['fast_404_string_whitelisting'] = array(
  '/files/css/css_',
  '/files/js/js_',
);


/**
 * Rate.
 *
 * The rate module uses a DrupalQueue to enqueue votes for later delete if it detects
 * the vote as coming from IP addresses belonging to bots. This queue can grow very
 * large if left with the default delete limit of 25.
 *
 * @note If rate used hook_cron_queue_info() this would be less of a pain point.
 */
$conf['rate_cron_delete_limit'] = 1000;
```

## Local Developer Configuration.

```php
/**
 * Views configuration.
 */
$conf['views_ui_show_sql_query'] = TRUE;
$conf['views_ui_show_sql_query_where'] = 'above';
$conf['views_ui_show_performance_statistics'] = TRUE;
$conf['views_sql_signature'] = TRUE;

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

/**
 * Disable various forms of caching.
 */
$conf['cache'] = FALSE;
$conf['block_cache'] = FALSE;
$conf['page_cache_maximum_age'] = FALSE;
$conf['preprocess_css'] = FALSE;
$conf['preprocess_js'] = FALSE;
$conf['views_skip_cache'] = TRUE;

// Overridden configuration go the file below. It should never be checked
// into version control.
$path = dirname(__FILE__) . DIRECTORY_SEPARATOR . 'local-overrides.config.inc';
if (file_exists($path)) {
  include $path;
}
```

## ```drushrc.php``` Configuration.

```php

/**
 * List of tables whose *data* is skipped by the 'sql-dump' and 'sql-sync'
 * commands when the "--structure-tables-key=common" option is provided.
 * You may add specific tables to the existing array or add a new element.
 *
 * @warning If you add a table that does not exist to this array, e.g. the
 * watchdog table if you have never enabled dblog then drush will silently fail.
 */
$options['structure-tables']['common'] = array(
  'history',
  'sessions',
  'flag_content',
  'watchdog',
  'blocked_ips',
  'flood',
  'migrate_log',
  'cache',
  'cache_block',
  'cache_bootstrap',
  'cache_entity_comment',
  'cache_entity_file',
  'cache_entity_node',
  'cache_entity_taxonomy_term',
  'cache_entity_taxonomy_vocabulary',
  'cache_entity_user',
  'cache_field',
  'cache_filter',
  'cache_form',
  'cache_image',
  'cache_menu',
  'cache_metatag',
  'cache_page',
  'cache_path',
  'cache_token',
  'cache_update',
  'cache_views',
  'cache_views_data',
);

// @TODO Setting default flags.
```
# General Software Engineering Patterns in Drupal

## General

* [Presentation Abstraction Control (PAC)/Hierarchical MVC](http://en.wikipedia.org/wiki/Presentation-abstraction-control)
* [Entity Attribute Value (EVA)](http://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model)
* [Event Condition Action (ECA)](http://en.wikipedia.org/wiki/Event_condition_action)
* [Post/Redirect/Get (PRG)](http://en.wikipedia.org/wiki/Post/Redirect/Get)
* [Front Controller](http://en.wikipedia.org/wiki/Front_controller)

## Plugins

* [Strategy Pattern](http://en.wikipedia.org/wiki/Strategy_pattern)
* [Dependency Injection (DI)](http://en.wikipedia.org/wiki/Dependency_injection)

## Hooks

* [Aspect Oriented Programming (AOP)](http://en.wikipedia.org/wiki/Aspect_oriented_programming)
* [Observer](http://en.wikipedia.org/wiki/Observer_pattern)

## Security

* [Synchronizer Token Pattern](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29_Prevention_Cheat_Sheet#General_Recommendation:_Synchronizer_Token_Pattern)

# Refactoring Drupal Code

* Write tests first where possible.
* Refactoring "inline" with normal work can be dangerous as it balloon tasks and lead to unexpected changes.
* drush views-analyze
* [Drupal Lint](https://drupal.org/sandbox/alanmackenzie/2096735)
* [Coder](https://drupal.org/project/coder)
* [Views Usage Audit](http://drupal.org/sandbox/alanmackenzie/2099953)

# Credits

## Authors

* Alan MacKenzie

## Inspiriation

* Django Beatty
* Sankatha Bamunuge
* Jon Muir
* Mark Sonnabaum
* Hernani Borges de Freitas

## Appendix

* tree -L 3 -d --charset=ascii

## TODO

* $form_state['storage'];
* egrep 'node/[0-9]+|user/[0-9]+|comment/[0-9]+|taxonomy/term/[0-9]+'
* Migration patterns, forceutf8, breaking migrations into chunks.
* Traditional domain modelling anti-pattern, dont tighly couple code to content.
* block_list, get_block_by_regen.
* overridable local developer configuration, not checked in.
* attach js to render_array or do in hook init
* Designing with teasers and entity displays in mind.
* views Pager id conflicting.
* designing with displays in mind.
* Using the most sepcific alter function. Helper funciton good for unit testing, less likely to run in unexpected places.
* Breakdown of different caches in the appendix.
* install profile needs to be kept up to date else 'ghost' modules will cause you problems when uninstalling dependencies.
* Multiple calls to module_enable/module_disable is a bad idea.
* drush rr --no-cache-clear option.
* hook_hook_info() example.
* views stored in code v views stored in the db.
* Schema api foreign keys.
* DrupalFakeCache
* QueueUI.
* File structure outside the docroot, test, vagrant etc.
* Don't put aliases in templates, l() will point node/1234 to the right alias automatically.
* Editorial freedom using variable_get() with the above.
* Removing magic constants example: ->fieldCondition('field_bbcgf_chef', 'target_id', array_keys(array(18 => 'Gordon Ramsay')), '<>')
* More on magic constants: ->fieldCondition('field_bbcgf_publication_date', 'value', strtotime('November 2007'), '<')
* Features internals, config as entities delete/creation on revert, manually editing feature files. The structure of a feature file: .info etc.
* No namespace in descriptions seen by editorial or end users.
* views tags used in template file names.
* Using APC for anything other than opcode caching.
* Using the correct db_* functions. db_query() is only for maximum control.
* Strategies for relying on external services: Varnish middleware, queues & local caching.
* Module layout, remember to split admin.inc etc. Autoloading via hook_menu & hook_hook_info().
* Drupal & WEM.
* EFQ no left join etc.
* Drupal contrib scalability failure: fivestar, node_revision_delete, node_revision_restrict.
* Rate module botscout IP ban.
* Sub-directories per content type subdirectories for file fields, painful at scale.
