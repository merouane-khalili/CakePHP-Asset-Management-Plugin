# Asset Plugin

Dealing with javascript files and css files is easy. You just include all your stylesheets and scripts on every page. However, with all these mobile devices accessing your app we must make an effort to reduce loading time. Non-mobile users will also appreciate efforts to optimize frontend performance of your app, as it's in most cases a far more serious bottleneck than the performance of your backend (php, mysql, etc).

The Asset plugin does exactly that for you - it combines and minifies scripts and stylesheets and makes sure, that only the assets needed for a specific are loaded. Here is the full feature set:

## Features

1. Combining js files and css files.

2. Minification of combined files with support for different algorithmns (jsmin, google closure compiler and cssmin). There is an easy way to add your own algorithmns as well.

3. Ability to specify which assets should be loaded for a given Controller::action pair. This ensures you only load the files that you really need on the page.

4. Optional support for converting LESS to CSS with support for prepending a file to the package that contains variables and less mixins.

5. Automatic detection and conversion for image paths in stylesheets.

5. Combining and minification of files is optional, so that you can load the proper files in development mode to keep error messages pointing to the correct files and lines.

6. Support for javascript internationalisation using the __('some test') syntax

8. Automagic including of files that belong to your controller action or layout. For example if you access /posts/edit/2 the plugin will try to load the stylesheet /app/webroot/css/views/posts/edit.less and the script /app/webroot/js/views/posts/edit.js.

 It will also load /app/webroot/css/views/layouts/default.less and /app/webroot/js/views/layouts/default.js or whatever layout has been set in your PostsController.  
 You can also configure these auto-include paths to your liking.
 
9. A shell to prebuild all packaged asset files for all Controller::action pairs on deployment
   -> This is optional, as in most cases the conversion can be done on the fly as it's really fast
   -> This builds packaged files or all languages (think of javascript i18n) and all layouts (think of the auto include paths)

10. Handles external stylesheets and scripts gracefully, by not including them into packaged files.

11. Only rebuilds packaged files if any of the stylesheets or scripts included in them changed (based on file modified time).


## Requirements & Installation


The plugin has been designed to work with CakePHP 1.3.7 stable. You can also make it easily work for CakePHP 1.2.x. Instructions for that can be found at the end of the documentation.

In case you intend to use LESScss, you require Nodejs version 0.2.2 or later.


1. Move the plugin to /app/plugins or wherever your plugins reside.

2. Create the folders /app/webroot/css/aggregate and /app/webroot/js/aggregate, chmod them to 644 and and then add the execute flag to directories recursively. Make sure to chown them to www-data.www-data:

  chmod -R 644 /path/to/project/app/webroot/js/aggregate
  chmod -R +X /path/to/project/app/webroot/js/aggregate
  chown www-data.www-data /path/to/project/app/webroot/js/aggregate

  .. and likewise for /css/aggregate.

Make sure to create these folders for all environments (production, staging, etc).

If you use Git, it's a good idea to add an "empty" file to each folder and just add that file to the repository while the the directories themselves are added to your .gitignore file. This makes sure all environments get the folders, but the contents are not in your git repository.

3. Add Configure::write('Assets.packaging', true) to your core.php file. Set it to false if you don't want packaged and minified files. It's a good idea to keep this to true for production environments and to false for everything else.

4. Create the files /app/config/css_includes.php and /app/config/js_includes.php and add the following 2 lines in your /app/config/bootstrap.php file:

   Configure::load('css_includes');
   Configure::load('js_includes');

5. Now open the css_includes.php file and add all css (or less) files that you want to load for specific controllers/action pairs:

Example:
``` php
    <?php
    $config = array(
    	'CssIncludes' => array(
    		'vars_and_mixins.less' => '*:*', // always loaded
    		'assemblies.less' => 'Posts:*', // loaded for all actions of the Posts Controller
    		'stats.less' => 'Statistics:*, !Statistics:dashboard', // loaded for all actions in the StatisticsController except for the dashboard action
    		'home.less' => 'Home:view', // only loaded for HomeController::view()
    		'admin.less' => '*:admin_*', // loaded for all actions prefixed by "admin_"
    		'//fonts.googleapis.com/css?family=Inconsolata' => '*:*' // an external stylesheet loaded everywhere
    		// ...
    	)
    );
```

Do this likewise for javascript files in your js_includes.php file:

``` php
    <?php
    $config = array(
    	'JsIncludes' => array(
    		'dep/jquery.js' => '*:*',
    		'plugins/flot/jquery.flot.min.js' => 'Statistics:admin_dashboard',
    		'plugins/flot/jquery.flot.selection.js' => 'Statistics:admin_dashboard',
    		// ...
    	)
    );
    ?>
```

6. Create the files /app/views/elements/css_includes.ctp and /app/views/elements/js_includes.ctp and fill them with the following contents:

css_includes.ctp:

``` php
    <?php
    if (!isset($asset)) {
    	return;
    }

    $inclusionRules = Configure::read('CssIncludes');
    $settings = array(
    	'type' => 'css',
    	'packaging' => Configure::read('Assets.packaging'),
    	'css' => array(
    		'mixins_file' => 'vars_and_mixins.less' // if you need support for less
    	)
    );
    $asset->includeFiles($inclusionRules, $settings);
    ?>
```

js_includes.ctp:
``` php
    <?php
    if (!isset($asset)) {
    	return;
    }

    $inclusionRules = Configure::read('JsIncludes');
    $settings = array(
    	'type' => 'js',
    	'packaging' => Configure::read('Assets.packaging')
    );

    // IE sometimes has problems with minifications.
    // Better turn minification off for IE.
    $isIe = isset($_SERVER['HTTP_USER_AGENT']) && strpos($_SERVER['HTTP_USER_AGENT'], 'MSIE') !== false;
    if ($isIe) {
    	$settings['minify'] = false;
    }
    $asset->includeFiles($inclusionRules, $settings);
    ?>
```

7. Make sure to load the css_includes element in the header of your layouts:

``` php
    <?php echo $this->element('css_includes'); ?>
```

8. Make sure to load the js_includes element in the footer of your layout:

``` php
    <?php echo $this->element('js_includes'); ?>
```

That's it!



## Usage & Options


### The Shell

To prebuild all your assets just run the prebuild_assets shell:

./cake prebuild_assets

This will build all packaged and minified files for all combinations of languages, layouts.

You can supply the list of languages you want to build javascript files for via the lang parameter.

./cake prebuild_assets -lang "en,fr,de"

Make sure to run the shell as root or in sudo as www-data to avoid permission problems.


### CSS Stylesheets

Here is a list of all options you can set for css files:

``` php
    $inclusionRules = Configure::read('CssIncludes');
    $settings = array(
    	'type' => 'css', // the type of inclusion to do; can be "css" or "js"
    	'packaging' => Configure::read('Assets.packaging'), // determine if files should be combined or not
    	'css' => array(
    		'path' => CSS, // the path where to look for stylesheets and where your "aggregate" folder is
    		'preConvertExt' => 'less', // the extension of the files prior to conversion to LESS; set this to false disable LESS conversion; default is 'less'
    		'ext' => 'css', // the extension of the result file(s)
    		'delim' => "\n\n", // delimiter to use between the contents of css files in the combined css file
    		'mixins_file' => 'vars_and_mixins.less' // the file to prepend to less conversions so variables and mixins are properly used, default is ''
    		'minification_engine' => array(
    			'method' => 'cssmin', // which algorithmn to use for css minifications, default is cssmin
    			'per_file' => false // if the minification should be run for each included css file or only once on the combined file; default is false
    		)
    	)
    );
    $asset->includeFiles($inclusionRules, $settings);
```


### Javascript

Here is a list of all options you can set for js files:

``` php
    <?php
    $inclusionRules = Configure::read('JsIncludes');
    $settings = array(
    	'type' => 'js',
    	'packaging' => Configure::read('Assets.packaging')
    	'js' => array(
    		'path' => JS, // the path where to look for scripts and where your "aggregate" folder is
    		'ext' => 'js', // the extension of the result file(s)
    		'delim' => ";\n\n", // delimiter to use between the contents of css files in the combined css
    		'js_i18n' => true, // whether to do translate __('some test') occurences in your javascript files
    		'minification_engine' => array(
    			'method' => 'jsmin', // which algorithmn to use for js minifications, default is "jsmin", can also be "google_closure"
    			'per_file' => true // if the minification should be run for each included js file or only once on the combined file; default is true
    		)
    	),
    );
    $asset->includeFiles($inclusionRules, $settings);
    ?>
```

### General Options

@TODO: Move language to js setting in the helper

1. Combining
@todo

2. Minification
@todo

3. Configuring auto include paths
@todo

4. Directory cleaning
@todo

5. Language
@todo

6. pathToNode (in case you use LESS)
@todo



## Adding your own minification algorithm.

@TODO


## Making the plugin work with CakePHP 1.2.x

The only thing you need to change is the call to $this->Html->script() in the includeFiles() method of the AssetHelper.

Change it from

if ($out) {
	if (!isset($this->Html)) {
		App::import('Helper', 'Html');
		$this->Html = new HtmlHelper();
	}

	if ($type == 'js') {
		foreach ($externals as $file) {
			echo $this->Html->script($file);
		}
	}

	if ($type == 'css') {
		foreach ($externals as $file) {
			echo $this->Html->css($file);
		}
	}
}

to

if ($out) {
	if (!isset($this->Html)) {
		App::import('Helper', 'Html');
		$this->Html = new HtmlHelper();
	}
	if (!isset($this->Javascript)) {
		App::import('Helper', 'Javascript');
		$this->Html = new JavascriptHelper();
	}

	if ($type == 'js') {
		foreach ($externals as $file) {
			echo $this->Javascript->link($file);
		}
	}

	if ($type == 'css') {
		foreach ($externals as $file) {
			echo $this->Html->css($file);
		}
	}
}

Now the plugin should work for you as well on Cake 1.2.x.