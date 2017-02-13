# PHP5 Extension Development

Next three articles were originally written by [Sara Golemon](https://github.com/sgolemon) in 2005. Since documentation for internal C API of Zend PHP Engine is very obscure, I've used this articles as some kind of introduction.
Original articles:

* [Extension Writing Part I: Introdcution to PHP and Zend](https://devzone.zend.com/303/extension-writing-part-i-introduction-to-php-and-zend/)
* [Extension Writing Part II: Parameters, Arrays, and ZVALs](https://devzone.zend.com/317/extension-writing-part-ii-parameters-arrays-and-zvals/)
* [Extension Writing Part III: Resources](http://devzone.zend.com/446/extension-writing-part-iii-resources/)

Other place to start:

* [Extending and Embedding PHP](https://www.amazon.com/dp/067232704X), that was written, also, for PHP5.
* [15 Excellent Resources for PHP Extension Development](https://goo.gl/4XPMAO)
* [Online code browser is a great thing](lxr.php.net)
* [Upgrading extensions to PHP7](https://wiki.php.net/phpng-upgrading)
* [Zend API: Hacking the Zend Core](http://php.net/manual/en/internals2.ze1.zendapi.php)
  is a very-very-very old document, but it gives understanding of Zend internals
* Reading codes for other extensions..

# PHP5 Extension Development (by Sara Golemon)


- [Part I: Introduction to PHP and Zend](#extension-writing-part-i-introduction-to-php-and-zend)
  * [Introduction](#introduction)
  * [What’s an Extension?](#whats-an-extension)
  * [Lifecycles](#lifecycles)
  * [Memory Allocation](#memory-allocation)
  * [Setting Up a Build Environment](#setting-up-a-build-environment)
  * [Hello World](#hello-world)
  * [Building Your Extension](#building-your-extension)
  * [INI Settings](#ini-settings)
  * [Global Values](#global-values)
  * [INI Settings as Global Values](#ini-settings-as-global-values)
  * [Sanity Check](#sanity-check)
- [Part II: Parameters, Arrays, and ZVALs](#extension-writing-part-ii-parameters-arrays-and-zvals)
  * [Introduction](#introduction)
  * [Accepting Values](#accepting-values)
  * [The ZVAL](#the-zval)
  * [Creating ZVALs](#creating-zvals)
  * [Arrays](#arrays)
  * [Symbol Tables as Arrays](#symbol-tables-as-arrays)
  * [Reference Counting](#reference-counting)
  * [Copies versus References](#copies-versus-references)
- [Part III: Resources](#extension-writing-part-iii-resources)
  * [Introduction](#introduction)
  * [Resources](#resources)
  * [Initializing Resources](#initializing-resources)
  * [Accepting Resources as Function Parameters](#accepting-resources-as-function-parameters)
  * [Destroying Resources](#destroying-resources)
  * [Destroying a Resource by Force](#destroying-a-resource-by-force)
  * [Persistent Resources](#persistent-resources)
  * [Finding Existing Persistent Resources](#finding-existing-persistent-resources)

## Extension Writing Part I: Introduction to PHP and Zend

### Introduction

If you’re reading this tutorial, you probably have some interest in writing an extension for the PHP language. If not… well perhaps when we’re done you’ll have discovered an interest you didn’t know existed!

This tutorial assumes basic familiarity with both the PHP language and the language the PHP interpreter is written in: C.

Let’s start by identifying why you might want to write a PHP extension.

There is some library or OS specific call which cannot be made from PHP directly because of the degree of abstraction inherent in the language. You want to make PHP itself behave in some unusual way. You’ve already got some PHP code written, but you know it could be faster, smaller, and consume less memory while running. You have a particularly clever bit of code you want to sell, and it’s important that the party you sell it to be able to execute it, but not view the source. These are all perfectly valid reasons, but in order to create an extension, you need to understand what an extension is first.

### What’s an Extension?

If you’ve used PHP, you’ve used extensions. With only a few exceptions, every userspace function in the PHP language is grouped into one extension or another. A great many of these functions are part of the standard extension – over 400 of them in total. The PHP source bundle comes with around 86 extensions, having an average of about 30 functions each. Do the math, that’s about 2500 functions. As if this weren’t enough, the PECL repository offers over 100 additional extensions, and even more can be found elsewhere on the Internet.

“With all these functions living in extensions, what’s left?” I hear you ask. “What are they an extension to? What is the ‘core’ of PHP?”

PHP’s core is made up of two separate pieces. At the lowest levels you find the `Zend Engine` (`ZE`). `ZE` handles parsing a human-readable script into
machine-readable tokens, and then executing those tokens within a process space. `ZE` also handles memory management, variable scope, and dispatching function calls. The other half of this split personality is the PHP core. PHP handles communication with, and bindings to, the SAPI layer (Server Application Programming Interface, also commonly used to refer to the host environment – Apache, IIS, CLI, CGI, etc). It also provides a unified control layer for `safe_mode` and `open_basedir` checks, as well as the streams layer which associates file and network I/O with userspace functions like `fopen()`, `fread()`, and `fwrite()`.

### Lifecycles

When a given SAPI starts up, for example in response to `/usr/local/apache/bin/apachectl` start, PHP begins by initializing its core subsystems. Towards the end of this startup routine, it loads the code for each extension and calls their `Module Initialization routine` (`MINIT`). This gives each extension a chance to initialize internal variables, allocate resources, register resource handlers, and register its functions with `ZE`, so that if a script calls one of those functions, `ZE` knows which code to execute.

Next, PHP waits for the SAPI layer to request a page to be processed. In the case of the CGI or CLI SAPIs, this happens immediately and only once. In the case of Apache, IIS, or other fully-fledged web server SAPIs, it occurs as pages are requested by remote users and repeats any number of times, possibly concurrently. No matter how the request comes in, PHP begins by asking `ZE` to setup an environment for the script to run in, then calls each extension’s `Request Initialization` (`RINIT`) function. `RINIT` gives the extension a chance to set up specific environment variables, allocate request specific resources, or perform other tasks such as auditing. A prime example of the `RINIT` function in action is in the sessions extension where, if the `session.auto_start` option is enabled, `RINIT` will automatically trigger the userspace `session_start()` function and pre-populate the `$_SESSION` variable.

Once the request is initialized, `ZE` takes over by translating the PHP script into tokens, and finally to opcodes which it can step through and execute. Should one of these opcodes require an extension function to be called, `ZE` will bundle up the arguments for that function, and temporarily give over control until it completes.

After a script has finished executing, PHP calls the `Request Shutdown` (`RSHUTDOWN`) function of each extension to perform any last minute cleanup (such as saving session variables to disk). Next, `ZE` performs a cleanup process (known as garbage collection) which effectively performs an `unset()` on every variable used during the previous request.

Once completed, PHP waits for the SAPI to either request another document or signal a shutdown. In the case of the CGI and CLI SAPIs, there is no “next
request”, so the SAPI initiates a shutdown immediately. During shutdown, PHP again cycles through each extension calling their `Module Shutdown`
(`MSHUTDOWN`) functions, and finally shuts down its own core subsystems.

This process may sound daunting at first, but once you dive into a working extension it should all gradually start to make sense.

### Memory Allocation

In order to avoid losing memory to poorly written extensions, `ZE` performs its own internal memory management using an additional flag that indicates persistence. A persistent allocation is a memory allocation that is meant to last for longer than a single page request. A non-persistent allocation, by contrast, is freed at the end of the request in which it was allocated, whether or not the free function is called. Userspace variables, for example, are allocated non-persistently because at the end of a request they’re no longer useful.

While an extension may, in theory, rely on `ZE` to free non-persistent memory automatically at the end of each page request, this is not recommended. Memory
allocations will remain unreclaimed for longer periods of time, resources associated with that memory will be less likely to be shutdown properly, and it’s just poor practice to make a mess without cleaning it up. As you’ll discover later on, it’s actually quite easy to ensure that all allocated data is cleaned up properly

Let’s briefly compare traditional memory allocation functions (which should only be used when working with external libraries) with persistent and non-persistent memory allocation within PHP/`ZE`.

```
+--------------------------------+--------------------------------+---------------------------------+
|Traditional	                 | Non-Persistent                 | Persistent                      |
+--------------------------------+--------------------------------+---------------------------------+
| malloc(count)                  | emalloc(count)                 | pemalloc(count, 1)[*]           |
| calloc(count, num)             | ecalloc(count, num)            | pecalloc(count, num, 1)         |
+--------------------------------+--------------------------------+---------------------------------+
| strdup(str)                    | estrdup(str)                   | pestrdup(str, 1)                |
| strndup(str, len)	             | estrndup(str, len)             | pemalloc() & memcpy()           |
+--------------------------------+--------------------------------+---------------------------------+
| free(ptr)	                     | efree(ptr)	                  | pefree(ptr, 1)                  |
+--------------------------------+--------------------------------+---------------------------------+
| realloc(ptr, newsize)	         | erealloc(ptr, newsize)         | perealloc(ptr, newsize, 1)      |
+--------------------------------+--------------------------------+---------------------------------+
| malloc(count * num + extr)[**] | safe_emalloc(count, num, extr) | safe_pemalloc(count, num, extr) |
+--------------------------------+--------------------------------+---------------------------------+

[*] The `pemalloc()` family include a ‘persistent’ flag which allows them
    to behave like their non-persistent counterparts. For example
    `emalloc(1234)` is the same as `pemalloc(1234, 0)`
       
[**] `safe_emalloc()` and (in PHP 5) `safe_pemalloc()` perform an
     additional check to avoid integer overflows
```

### Setting Up a Build Environment

Now that you’ve covered some of the theory behind the workings of PHP and the Zend Engine, I’ll bet you’d like to dive in and start building something. Before you can do that however, you’ll need to collect some necessary build tools and set up an environment suited to your purposes.

First you’ll need PHP itself, and the set of build tools required by PHP. If you’re unfamiliar with building PHP from source, I suggest you take a look at
http://www.php.net/install.unix.php (Developing PHP extensions for Windows will be covered in a later article). While it might be tempting to use a binary
package of PHP from your distribution of choice, these versions tend to leave out two important `./configure` options that are very handy during the development process. The first is `--enable-debug`. This option will compile PHP with additional symbol information loaded into the executable so that, if a segfault occurs, you’ll be able to collect a core dump from it and use gdb to track down where the segfault occurred and why. The other option depends on which version of PHP you’ll be developing against. In PHP 4.3 this option is named `--enable-experimental-zts`, in PHP 5 and later it’s `--enable-maintainer-zts`. This option will make PHP think its operating in a multi-threaded environment and will allow you to catch common programming mistakes which, while harmless in a non-threaded environment, will cause your extension to be unusable in a multi-threaded one. Once you’ve compiled PHP using these extra options and installed it on your development server (or workstation), you can begin to put together your first extension.

### Hello World

What programming introduction would be complete without the requisite Hello World application? In this case, you’ll be making an extension that exports a single function returning a string containing the words: “Hello World”. In PHP code you’d probably do it something like this:

``` php
<?php
hello_world() {
    return 'Hello World';
}
```

Now you’re going to turn that into a PHP extension. First let’s create a directory called hello under the ext/ directory in your PHP source tree and chdir into that folder. This directory can actually live anywhere inside or outside the PHP tree, but I’d like you to place it here to demonstrate an unrelated concept in a later article. Here you need to create three files: a source file containing your hello_world function, a header file containing references used by PHP to load your extension, and a configuration file used by phpize to prepare your extension for compiling.

``` m4
-- config.m4
PHP_ARG_ENABLE(hello, whether to enable Hello
World support,
[ --enable-hello   Enable Hello World support])
if test "$PHP_HELLO" = "yes"; then
  AC_DEFINE(HAVE_HELLO, 1, [Whether you have Hello World])
  PHP_NEW_EXTENSION(hello, hello.c, $ext_shared)
fi
```

``` c
// php_hello.h
#ifndef PHP_HELLO_H
#define PHP_HELLO_H 1
#define PHP_HELLO_WORLD_VERSION "1.0"
#define PHP_HELLO_WORLD_EXTNAME "hello"

PHP_FUNCTION(hello_world);

extern zend_module_entry hello_module_entry;
#define phpext_hello_ptr &hello_module_entry

#endif
```

``` c
// hello.c
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
#include "php.h"
#include "php_hello.h"

static function_entry hello_functions[] = {
    PHP_FE(hello_world, NULL)
    {NULL, NULL, NULL}
};

zend_module_entry hello_module_entry = {
#if ZEND_MODULE_API_NO >= 20010901
    STANDARD_MODULE_HEADER,
#endif
    PHP_HELLO_WORLD_EXTNAME,
    hello_functions,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
#if ZEND_MODULE_API_NO >= 20010901
    PHP_HELLO_WORLD_VERSION,
#endif
    STANDARD_MODULE_PROPERTIES
};

#ifdef COMPILE_DL_HELLO
ZEND_GET_MODULE(hello)
#endif

PHP_FUNCTION(hello_world) {
    RETURN_STRING("Hello World", 1);
}
```

Most of the code you can see in the example extension above is just glue - protocol language to introduce the extension to PHP and establish a dialogue for them to communicate. Only the last four lines are what you might call “real code” which performs a task on a level that the userspace script might interact with. Indeed the code at this level looks very similar to the PHP code we looked at earlier and can be easily parsed on sight:

Declare a function named hello_world. Have that function return a string: “Hello World”. ...um.. 1? What’s that 1 all about?
Recall that ZE includes a sophisticated memory management layer which ensures that allocated resources are freed when the script exits. In the land of memory management however, it’s a big no-no to free the same block of memory twice. This action, called double freeing, is a common cause of segmentation faults, as it involves the calling program trying to access a block of memory which it no longer owns. Similarly, you don’t want to allow ZE to free a static string buffer (such as “Hello World” in our example extension) as it lives in program space and thus isn’t a data block to be owned by any one process. `RETURN_STRING()` could assume that any strings passed to it need to be copied so that they can be safely freed later; but since it’s not uncommon for an internal function to allocate memory for a string, fill it dynamically, then return it, RETURN_STRING() allows us to specify whether it’s necessary to make a copy of the string value or not. To further illustrate this concept, the following code snippet is identical to its counterpart above:

``` c
PHP_FUNCTION(hello_world) {
    char *str;
    str = estrdup("Hello World");
    RETURN_STRING(str, 0);
}
```

In this version, you manually allocated the memory for the “Hello World” string that will ultimately be passed back to the calling script, then “gave” that memory to `RETURN_STRING()`, using a value of 0 in the second parameter to indicate that it didn’t need to make its own copy, it could have ours.

### Building Your Extension

The final step in this exercise will be building your extension as a dynamically loadable module. If you’ve copied the example above correctly, this should only take three commands run from `ext/hello/`:

``` bash
$ phpize
$ ./configure --enable-hello
$ make
```

After running each of these commands, you should have a `hello.so` file in `ext/hello/modules/` . Now, as with any other PHP extension, you can
just copy this to your extensions directory (`/usr/local/lib/php/extensions/` is the default, check your `php.ini` to be sure) and add the line `extension=hello.so` to your `php.ini` to trigger it to load on startup. For CGI/CLI SAPIs, this simply means the next time PHP is run; for web server SAPIs like Apache, this will be the next time the web server is restarted. Let’s give it a try from the command line for now:

``` bash
$ php -r 'echo hello_world();'
```

If everything’s gone as it should, you should see Hello World output by this script, since the `hello_world()` function in your loaded extension returns that string, and the echo command displays whatever is passed to it (the result of the function in this case).

Other scalars may be returned in a similar fashion, using `RETURN_LONG()` for integer values, `RETURN_DOUBLE()` for floating point values, `RETURN_BOOL()` for `true`/`false` values, and `RETURN_NULL()` for, you guessed it, `NULL` values. Let’s take a look at each of those in action by adding `PHP_FE()` lines to the function_entry struct in hello.c and adding some `PHP_FUNCTION()`s to the end of the file.

``` c
static function_entry hello_functions[] = {
    PHP_FE(hello_world, NULL)
    PHP_FE(hello_long, NULL)
    PHP_FE(hello_double, NULL)
    PHP_FE(hello_bool, NULL)
    PHP_FE(hello_null, NULL)
    {NULL, NULL, NULL}
};

PHP_FUNCTION(hello_long) {
    RETURN_LONG(42);
}
PHP_FUNCTION(hello_double) {
    RETURN_DOUBLE(3.1415926535);
}
PHP_FUNCTION(hello_bool) {
    RETURN_BOOL(1);
}
PHP_FUNCTION(hello_null) {
    RETURN_NULL();
}
```

You’ll also need to add prototypes for these functions alongside the prototype for `hello_world()` in the header file, php_hello.h, so that the build process takes place properly:

``` c
PHP_FUNCTION(hello_world);
PHP_FUNCTION(hello_long);
PHP_FUNCTION(hello_double);
PHP_FUNCTION(hello_bool);
PHP_FUNCTION(hello_null);
```

Since you made no changes to the `config.m4` file, it’s technically safe to skip the `phpize` and `./configure` steps this time and jump straight to
`make`. However, at this stage of the game I’m going to ask you to go through all three build steps again just to make sure you have a nice build. In additional, you should call `make clean` all rather than simply `make` in the last step, to ensure that all source files are rebuilt. Again, this isn’t necessary because of the types of changes you’ve made so far, but better safe than confused. Once the module is built, you’ll again copy it to your extension directory, replacing the old version.

At this point you could call the PHP interpreter again, passing it simple scripts to test out the functions you just added. In fact, why don’t you do that now? I’ll wait here…

Done? Good. If you used `var_dump()` rather than `echo` to view the output of each function then you probably noticed that `hello_bool()` returned true. That’s what the 1 value in `RETURN_BOOL()` represents. Just like in PHP scripts, an integer value of 0 equates to `FALSE`, while any other integer value equates to `TRUE`. Extension authors often use 1 as a matter of convention, and you’re encouraged to do the same, but don’t feel locked into it. For added readability, the `RETURN_TRUE` and `RETURN_FALSE` macros are also available; here’s `hello_bool()` again, this time using `RETURN_TRUE`:

``` c
PHP_FUNCTION(hello_bool) {
    RETURN_TRUE;
}
```

Note that no parentheses were used here. `RETURN_TRUE` and `RETURN_FALSE` are aberrations from the rest of the `RETURN_*()` macros in that way, so be sure not to get caught by this one!

You probably noticed in each of the code samples above that we didn’t pass a zero or one value indicating whether or not the value should be copied. This is because no additional memory (beyond the variable container itself – we’ll delve into this deeper in Part 2) needs to be allocated – or freed - for simple small scalars such as these.

There are an additional three return types: `RESOURCE` (as returned by `mysql_connect()`, `fsockopen()`, and `ftp_connect()` to name but a few), `ARRAY` (also known as a `HASH`), and `OBJECT` (as returned by the keyword new). We’ll look at these in Part II of this series, when we cover variables in depth.

### INI Settings

The Zend Engine provides two approaches for managing INI values. We’ll take a look at the simpler approach for now, and explore the fuller, but more complex, approach later on, when you’ve had a chance to work with global values.

Let’s say you want to define a php.ini value for your extension, `hello.greeting`, which will hold the value used to say hello in your `hello_world()` function. You’ll need to make a few additions to `hello.c` and `php_hello.h` while making a few key changes to the `hello_module_entry` structure. Start off by adding the following prototypes near the userspace function prototypes in `php_hello.h`:

``` c
PHP_MINIT_FUNCTION(hello);
PHP_MSHUTDOWN_FUNCTION(hello);
PHP_FUNCTION(hello_world);
PHP_FUNCTION(hello_long);
PHP_FUNCTION(hello_double);
PHP_FUNCTION(hello_bool);
PHP_FUNCTION(hello_null);
```

Now head over to `hello.c` and take out the current version of `hello_module_entry`, replacing it with the following listing:

``` c
zend_module_entry hello_module_entry = {
#if ZEND_MODULE_API_NO >= 20010901
    STANDARD_MODULE_HEADER,
#endif
    PHP_HELLO_WORLD_EXTNAME,
    hello_functions,
    PHP_MINIT(hello),
    PHP_MSHUTDOWN(hello),
    NULL,
    NULL,
    NULL,
#if ZEND_MODULE_API_NO >= 20010901
    PHP_HELLO_WORLD_VERSION,
#endif
    STANDARD_MODULE_PROPERTIES
};
PHP_INI_BEGIN()
    PHP_INI_ENTRY("hello.greeting", "Hello World", PHP_INI_ALL, NULL)
PHP_INI_END()

PHP_MINIT_FUNCTION(hello) {
    REGISTER_INI_ENTRIES();
    return SUCCESS;
}

PHP_MSHUTDOWN_FUNCTION(hello) {
    UNREGISTER_INI_ENTRIES();
    return SUCCESS;
}
```

Now, you just need to add an #include to the rest of the `#includes` at the top of `hello.c` to get the right headers for INI file support:

``` c
#ifdef HAVE_CONFIG_H
    #include "config.h"
#endif
#include "php.h"
#include "php_ini.h"
#include "php_hello.h"
```

Finally, you can modify your hello_world function to use the INI value:

``` c
PHP_FUNCTION(hello_world) {
    RETURN_STRING(INI_STR("hello.greeting"), 1);
}
```

Notice that you’re copying the value returned by `INI_STR()`. This is because, as far as the PHP variable stack is concerned, this is a static string. In fact, if you tried to modify the string returned by this value, the PHP execution environment would become unstable and might even crash.

The first set of changes in this section introduced two methods you’ll want to become very familiar with: `MINIT`, and `MSHUTDOWN`. As mentioned earlier, these methods are called during the initial startup of the `SAPI` layer and during its final shutdown, respectively. They are not called between or during requests. In this example you’ve used them to register the `php.ini` entries defined in your extension. Later in this series, you’ll find how to use the `MINIT` and `MSHUTDOWN` functions to register resource, object, and stream handlers as well.

In your `hello_world()` function you used `INI_STR()` to retrieve the current value of the `hello.greeting` entry as a string. A host of other functions exist for retrieving values as `longs`, `doubles`, and `Booleans` as shown in the following table, along with a complementary `ORIG` counterpart which provides the value of the referenced INI setting as it was set in `php.ini` (before being altered by `.htaccess` or `ini_set()` statements).

```
+----------------+---------------------+--------------------------+
| Current Value  | Original Value      | Type                     |
+----------------+---------------------+--------------------------+
| INI_STR(name)  | INI_ORIG_STR(name)  | char * (NULL terminated) |
| INI_INT(name)  | INI_ORIG_INT(name)  | signed long              |
| INI_FLT(name)  | INI_ORIG_FLT(name)  | signed double            |
| INI_BOOL(name) | INI_ORIG_BOOL(name) | zend_bool                |
+----------------+---------------------+--------------------------+
```

The first parameter passed to `PHP_INI_ENTRY()` is a string containing the name of the entry to be used in `php.ini`. In order to avoid namespace collisions, you should use the same conventions as with your functions; that is, prefix all values with the name of your extension, as you did with `hello.greeting`. As a matter of convention, a period is used to separate the extension name from the more descriptive part of the ini setting name.

The second parameter is the initial value, and is always given as a `char*` string regardless of whether it is a numerical value or not. This is due primarily to the fact that values in an `.ini` file are inherently textual – being a text file and all. Your use of `INI_INT()`, `INI_FLT()`, or `INI_BOOL()` later in your script will handle type conversions.

The third value you pass is an access mode modifier. This is a bitmask field which determines when and where this INI value should be modifiable. For some, such as `register_globals`, it simply doesn’t make sense to allow the value to be changed from within a script using `ini_set()` because the setting only has meaning during request startup - before the script has had a chance to run. Others, such as `allow_url_fopen`, are administrative settings which you don’t want to allow users on a shared hosting environment to change, either via `ini_set()` or through the use of `.htaccess` directives. A typical value for this parameter might be `PHP_INI_ALL`, indicating that the value may be changed anywhere. Then there’s `PHP_INI_SYSTEM|PHP_INI_PERDIR`, indicating that the setting may be changed in the `php.ini` file, or via an Apache directive in a `.htaccess` file, but not through the use of `ini_set()`. Or there’s `PHP_INI_SYSTEM`, meaning that the value may only be changed in the `php.ini` file and nowhere else.

We’ll skip the fourth parameter for now and only mention that it allows the use of a callback method to be triggered whenever the ini setting is changed, such as with `ini_set()`. This allows an extension to perform more precise control over when a setting may be changed, or trigger a related action dependant on the new setting.

### Global Values

Frequently, an extension will need to track a value through a particular request, keeping that value independent from other requests which may be occurring at the same time. In a non-threaded SAPI that might be simple: just declare a global variable in the source file and access it as needed. The trouble is, since PHP is designed to run on threaded web servers (such as Apache 2 and IIS), it needs to keep the global values used by one thread separate from the global values used by another. PHP greatly simplifies this by using the TSRM (Thread Safe Resource Management) abstraction layer, sometimes referred to as ZTS (Zend Thread Safety). In fact, by this point you’ve already used parts of TSRM and didn’t even know it. (Don’t search too hard just yet; as this series progresses you’ll come to discover it’s hiding everywhere.)

The first part of creating a thread safe global is, as with any global, declaring it. For the sake of this example, you’ll declare one global value which
will start out as a `long` with a value of 0. Each time the `hello_long()` function is called you’ll increment this value and return it. Add the following block of code to php_hello.h just after the `#define PHP_HELLO_H` statement:

``` c
#ifdef ZTS
#include "TSRM.h"
#endif
ZEND_BEGIN_MODULE_GLOBALS(hello)
    long counter;
ZEND_END_MODULE_GLOBALS(hello)

#ifdef ZTS
#define HELLO_G(v) TSRMG(hello_globals_id, zend_hello_globals *, v)
#else
#define HELLO_G(v) (hello_globals.v)
#endif
```

You’re also going to use the `RINIT` method this time around, so you need to declare its prototype in the header:

``` c
PHP_MINIT_FUNCTION(hello);
PHP_MSHUTDOWN_FUNCTION(hello);
PHP_RINIT_FUNCTION(hello);
```

Now let’s go over to `hello.c` and add the following just after your include block:

``` c
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
#include "php.h"
#include "php_ini.h"
#include "php_hello.h"

ZEND_DECLARE_MODULE_GLOBALS(hello)
```

Change hello_module_entry by adding `PHP_RINIT(hello)`:

``` c
zend_module_entry hello_module_entry = {
#if ZEND_MODULE_API_NO >= 20010901
    STANDARD_MODULE_HEADER,
#endif
    PHP_HELLO_WORLD_EXTNAME,
    hello_functions,
    PHP_MINIT(hello),
    PHP_MSHUTDOWN(hello),
    PHP_RINIT(hello),
    NULL,
    NULL,
#if ZEND_MODULE_API_NO >= 20010901
    PHP_HELLO_WORLD_VERSION,
#endif
    STANDARD_MODULE_PROPERTIES
};
```

And modify your `MINIT` function, along with the addition of another couple of functions, to handle initialization upon request startup:

``` c
static void
php_hello_init_globals(zend_hello_globals *hello_globals) {
}
PHP_RINIT_FUNCTION(hello) {
    HELLO_G(counter) = 0;
    return SUCCESS;
}

PHP_MINIT_FUNCTION(hello) {
    ZEND_INIT_MODULE_GLOBALS(hello, php_hello_init_globals, NULL);
    REGISTER_INI_ENTRIES();
    return SUCCESS;
}
```

Finally, you can modify the hello_long() function to use this value:

``` c
PHP_FUNCTION(hello_long) {
    HELLO_G(counter)++;
    RETURN_LONG(HELLO_G(counter));
}
```

In your additions to php_hello.h, you used a pair of macros - `ZEND_BEGIN_MODULE_GLOBALS()` and `ZEND_END_MODULE_GLOBALS()` - to create a struct named `zend_hello_globals` containing one variable of type long. You then conditionally defined `HELLO_G()` to either fetch this value from a thread pool, or just grab it from a global scope – if you’re compiling for a non-threaded environment.

In `hello.c` you used the `ZEND_DECLARE_MODULE_GLOBALS()` macro to actually instantiate the `zend_hello_globals` struct either as a true global (if this is a non-thread-safe build), or as a member of this thread’s resource pool. As extension authors, this distinction is one we don’t need to worry about, as the Zend Engine takes care of the job for us. Finally, in `MINIT`, you used `ZEND_INIT_MODULE_GLOBALS()` to allocate a thread safe resource id – don’t worry about what that is for now.

You may have noticed that `php_hello_init_globals()` doesn’t actually do anything, yet we went to the trouble of declaring `RINIT` to initialize the counter to 0. Why?

The key lies in when the two functions are called. `php_hello_init_globals()` is only called when a new process or thread is started; however, each process can serve more than one request, so using this function to initialize our counter to 0 will only work for the first page request. Subsequent page requests to the same process will still have the old counter value stored here, and hence will not start counting from 0. To initialize the counter to 0 for every single page request, we implemented the `RINIT` function, which as you learned earlier is called prior to every page request. We included the `php_hello_init_globals()` function at this point because you’ll be using it in a few moments, but also because passing a `NULL` to `ZEND_INIT_MODULE_GLOBALS()` for the init function will result in a segfault on non-threaded platforms.

### INI Settings as Global Values

If you recall from earlier, a `php.ini` value declared with `PHP_INI_ENTRY()` is parsed as a string value and converted, as needed, to other formats with `INI_INT()`, `INI_FLT()`, and `INI_BOOL()`. For some settings, that represents a fair amount of unnecessary work duplication as the value is read over and over again during the course of a script’s execution. Fortunately it’s possible to instruct `ZE` to store the `INI` value in a particular data type, and
only perform type conversions when its value is changed. Let’s try that out by declaring another `INI` value, a Boolean this time, indicating whether the counter should increment, or decrement. Begin by changing the `MODULE_GLOBALS` block in `php_hello.h` to the following:

``` c
ZEND_BEGIN_MODULE_GLOBALS(hello)
    long counter;
    zend_bool direction;
ZEND_ENG_MODULE_GLOBALS(hello)
```

Next, declare the `INI` value itself by changing your `PHP_INI_BEGIN()` block thus:


``` c
PHP_INI_BEGIN()
    PHP_INI_ENTRY("hello.greeting", "Hello World", PHP_INI_ALL, NULL)
    STD_PHP_INI_ENTRY("hello.direction", "1", PHP_INI_ALL, OnUpdateBool, direction, zend_hello_globals, hello_globals)
PHP_INI_END()
```

Now initialize the setting in the init_globals method with:


``` c
static void php_hello_init_globals(zend_hello_globals *hello_globals) {
    hello_globals->direction = 1;
}
```

And lastly, use the value of the ini setting in `hello_long()` to determine whether to increment or decrement:

``` c
PHP_FUNCTION(hello_long) {
    if (HELLO_G(direction)) {
        HELLO_G(counter)++;
    } else {
        HELLO_G(counter)--;
    }
    RETURN_LONG(HELLO_G(counter));
}
```

And that’s it. The `OnUpdateBool` method you specified in the `INI_ENTRY` section will automatically convert any value provided in `php.ini`, `.htaccess`, or within a script via `ini_set()` to an appropriate `TRUE`/`FALSE` value which you can then access directly within a script. The last three parameters of `STD_PHP_INI_ENTRY` tell PHP which global variable to change, what the structure of our extension globals looks like, and the name of the global scope container where they’re contained.

### Sanity Check

By now our three files should look similar to the following listings. (A few items have been moved and grouped together, for the sake of readability.)

``` m4
-- config.m4
PHP_ARG_ENABLE(hello, whether to enable Hello
World support,
[ --enable-hello   Enable Hello World support])
if test "$PHP_HELLO" = "yes"; then
  AC_DEFINE(HAVE_HELLO, 1, [Whether you have Hello World])
  PHP_NEW_EXTENSION(hello, hello.c, $ext_shared)
fi
```

``` c
// php_hello.h
#ifndef PHP_HELLO_H
#define PHP_HELLO_H 1
#ifdef ZTS
#include "TSRM.h"
#endif

ZEND_BEGIN_MODULE_GLOBALS(hello)
    long counter;
    zend_bool direction;
ZEND_END_MODULE_GLOBALS(hello)

#ifdef ZTS
#   define HELLO_G(v) TSRMG(hello_globals_id, zend_hello_globals *, v)
#else
#   define HELLO_G(v) (hello_globals.v)
#endif

#define PHP_HELLO_WORLD_VERSION "1.0"
#define PHP_HELLO_WORLD_EXTNAME "hello"

PHP_MINIT_FUNCTION(hello);
PHP_MSHUTDOWN_FUNCTION(hello);
PHP_RINIT_FUNCTION(hello);

PHP_FUNCTION(hello_world);
PHP_FUNCTION(hello_long);
PHP_FUNCTION(hello_double);
PHP_FUNCTION(hello_bool);
PHP_FUNCTION(hello_null);

extern zend_module_entry hello_module_entry;
#define phpext_hello_ptr &hello_module_entry

#endif
```

``` c
// hello.c
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif
#include "php.h"
#include "php_ini.h"
#include "php_hello.h"

ZEND_DECLARE_MODULE_GLOBALS(hello)

static function_entry hello_functions[] = {
    PHP_FE(hello_world, NULL)
    PHP_FE(hello_long, NULL)
    PHP_FE(hello_double, NULL)
    PHP_FE(hello_bool, NULL)
    PHP_FE(hello_null, NULL)
    {NULL, NULL, NULL}
};

zend_module_entry hello_module_entry = {
#if ZEND_MODULE_API_NO >= 20010901
    STANDARD_MODULE_HEADER,
#endif
    PHP_HELLO_WORLD_EXTNAME,
    hello_functions,
    PHP_MINIT(hello),
    PHP_MSHUTDOWN(hello),
    PHP_RINIT(hello),
    NULL,
    NULL,
#if ZEND_MODULE_API_NO >= 20010901
    PHP_HELLO_WORLD_VERSION,
#endif
    STANDARD_MODULE_PROPERTIES
};

#ifdef COMPILE_DL_HELLO
ZEND_GET_MODULE(hello)
#endif

PHP_INI_BEGIN()
    PHP_INI_ENTRY("hello.greeting", "Hello World", PHP_INI_ALL, NULL)
    STD_PHP_INI_ENTRY("hello.direction", "1", PHP_INI_ALL, OnUpdateBool, direction, zend_hello_globals, hello_globals)
PHP_INI_END()

static void php_hello_init_globals(zend_hello_globals *hello_globals) {
    hello_globals->direction = 1;
}

PHP_RINIT_FUNCTION(hello) {
    HELLO_G(counter) = 0;
    return SUCCESS;
}

PHP_MINIT_FUNCTION(hello) {
    ZEND_INIT_MODULE_GLOBALS(hello, php_hello_init_globals, NULL);
    REGISTER_INI_ENTRIES();
    return SUCCESS;
}

PHP_MSHUTDOWN_FUNCTION(hello) {
    UNREGISTER_INI_ENTRIES();
    return SUCCESS;
}

PHP_FUNCTION(hello_world) {
    RETURN_STRING("Hello World", 1);
}

PHP_FUNCTION(hello_long) {
    if (HELLO_G(direction)) {
        HELLO_G(counter)++;
    } else {
        HELLO_G(counter)--;
    }
    RETURN_LONG(HELLO_G(counter));
}

PHP_FUNCTION(hello_double) {
    RETURN_DOUBLE(3.1415926535);
}

PHP_FUNCTION(hello_bool) {
    RETURN_BOOL(1);
}

PHP_FUNCTION(hello_null) {
    RETURN_NULL();
}
```

## Extension Writing Part II: Parameters, Arrays, and ZVALs

### Introduction

In Part One of this series you looked at the basic framework of a PHP extension. You declared simple functions that returned both static and dynamic values to the calling script, defined INI options, and declared internal values (globals). In this tutorial, you’ll learn how to accept values passed into your functions from a calling script and discover how PHP and the Zend Engine manage variables internally.

### Accepting Values

Unlike in userspace code, the parameters for an internal function aren’t actually declared in the function header. Instead, a reference to the parameter list is passed into every function – whether parameters were passed or not – and that function can then ask the Zend Engine to turn them into something usable.

Let’s take a look at this by defining a new function, hello_greetme(), which will accept one parameter and output it along with some greeting text. As before, we’ll be adding code in three places:

In `php_hello.h`, next to the other function prototypes:

``` c
PHP_FUNCTION(hello_greetme);
```

In hello.c, at the end of the hello_functions structure:

``` c
    PHP_FE(hello_bool, NULL)
    PHP_FE(hello_null, NULL)
    PHP_FE(hello_greetme, NULL)
    {NULL, NULL, NULL}
};
```

And down near the end of hello.c after the other functions:

``` c
PHP_FUNCTION(hello_greetme) {
    char *name;
    int name_len;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &name, &name_len) == FAILURE) {
        RETURN_NULL();
    }
    php_printf("Hello %s", name);
    RETURN_TRUE;
}
```

The bulk of the `zend_parse_parameters()` block will almost always look the same. `ZEND_NUM_ARGS()` provides a hint to the Zend Engine about the parameters which are to be retrieved, TSRMLS_CC is present to ensure thread safety, and the return value of the function is checked for `SUCCESS` or `FAILURE`. Under normal circumstances `zend_parse_parameters()` will return `SUCCESS`; however, if a calling script has attempted to pass too many or too few parameters, or if the parameters passed cannot be converted to the proper data type, Zend will automatically output an error message leaving your function to return control back to the calling script gracefully.

In this example you specified `s` to indicate that this function expects one and only one parameter to be passed, and that that parameter should be converted into a `string` data type and populated into the `char *` variable, which will be passed by reference (i.e. by name).

Note that an int variable was also passed into `zend_parse_parameters()` by reference. This allows the Zend Engine to provide the length of the string in bytes so that binary-safe functions don’t need to rely on strlen(name) to determine the string’s length. In fact, using `strlen(name)` may not even give the correct result, as name may contain one or more `NULL` characters prior to the end of the string.

Once your function has the name parameter firmly in hand, the next thing it does is output it as part of a formal greeting. Notice that `php_printf()` is used rather than the more familiar `printf()`. Using this function is important for several reasons. First, it allows the string being output to be processed through PHP’s output buffering mechanism, which may, in addition to actually buffering the data, perform additional processing such as gzip compression. Secondly, while stdout is a perfectly fine target for output when using CLI or CGI, most SAPIs expect output to come via a specific pipe or socket. Therefore, attempting to simply `printf()` to stdout could lead to data being lost, sent out of order, or corrupted, because it bypassed preprocessing.

Finally the function returns control to the calling program by simply returning `TRUE`. While you could allow control to reach the end of your function without explicitly returning a value (it will default to `NULL`), this is considered bad practice. A function that doesn’t have anything meaningful to report should typically return TRUE simply to say, “Everything’s cool, I did what you wanted”.

Because PHP strings may, in fact, contain `NULL`s, the way to output a binary-safe string, including `NULL`s and even characters following `NULL`s, would be to replace the `php_printf()` statement with the following block:

``` c
php_printf("Hello ");
PHPWRITE(name, name_len);
```

This block uses `php_printf()` to handle the strings which are known not to contain `NULL` characters, but uses another macro – `PHPWRITE` – to handle the user-provided string. This macro accepts the length (`name_len`) parameter provided by `zend_parse_parameters()` so that the entire contents of name can be printed out regardless of a stray `NULL`.

`zend_parse_parameters()` will also handle optional parameters. In the next example, you’ll create a function which expects a `long` (PHP’s integer data type), a `double` (float), and an optional `Boolean` value. A userspace declaration for this function might look something like:

``` php
function hello_add($a, $b, $return_long = false) {
    $sum = (int)$a + (float)$b;

    if ($return_long) {
        return intval($sum);
    } else {
        return floatval($sum);
    }
}
```

In C, this function will look like the following (don’t forget to add entries in `php_hello.h` and `hello_functions[]` to enable this when you add it to `hello.c`):

``` c
PHP_FUNCTION(hello_add)
{
    long a;
    double b;
    zend_bool return_long = 0;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ld|b", &a, &b, &return_long) == FAILURE) {
        RETURN_NULL();
    }

    if (return_long) {
        RETURN_LONG(a + b);
    } else {
        RETURN_DOUBLE(a + b);
    }
}
```

This time, your data type string reads as: “I want a (l)ong, then a (d)ouble”. The next character, a pipe, signifies that the rest of the parameter list is optional. If an optional parameter is not passed during the function call, then `zend_parse_parameters()` will not change the value passed into it. The final `b` is, of course, for Boolean. After the data type string, a, `b`, and return_long are passed through by reference so that zend_parse_parameters() can populate them with values.

Warning: while int and long are often used interchangeably on 32-bit platforms, using one in place of the other can be very dangerous when your code is recompiled on 64-bit hardware. So remember to use long for longs, and int for string lengths.

*Table 1* shows the various types, and their corresponding letter codes and C types which can be used with `zend_parse_parameters()`:

```
Table 1: Types and letter codes used in zend_parse_parameters()
+----------+------+---------------+
| Type     | Code | Variable Type |
+----------+------+---------------+
| Boolean  | b    | zend_bool     |
| Long     | l    | long          |
| Double   | d    | double        |
| String   | s    | char *, int   |
| Resource | r    | zval *        |
| Array    | a    | zval *        |
| Object   | o    | zval *        |
| zval     | z    | zval *        |
+----------+------+---------------+
```

You probably noticed right away that the last four types in *Table 1* all return the same data type – a `zval *`. A zval, as you’ll soon learn, is the true data type which all userspace variables in PHP are stored as. The three “complex” data types, `Resource`, `Array` and `Object`, are type-checked by the Zend Engine when their data type codes are used with `zend_parse_parameters()`, but because they have no corresponding data type in C, no conversion is actually performed.

### The ZVAL

The zval, and PHP userspace variables in general, will easily be the most difficult concepts you’ll need to wrap your head around. They will also be the most vital. To begin with, let’s look at the structure of a zval:

``` c
struct {
    union {
        long lval;
        double dval;
        struct {
            char *val;
            int len;
        } str;
        HashTable *ht;
        zend_object_value obj;
    } value;
    zend_uint refcount;
    zend_uchar type;
    zend_uchar is_ref;
} zval;
```

As you can see, every zval has three basic elements in common: `type`, `is_ref`, and `refcount`. `is_ref` and `refcount` will be covered later on in this tutorial; for now let’s focus on type.

By now you should already be familiar with PHP’s eight data types. They’re the seven listed in *Table 1*, plus `NULL`, which despite (or perhaps because of) the fact that it literally is nothing, is a type unto its own. Given a particular zval, the type can be examined using one of three convenience macros: `Z_TYPE(zval)`, `Z_TYPE_P(zval *)`, or `Z_TYPE_PP(zval **)`. The only functional difference between these three is the level of indirection expected in the variable passed into it. The convention of using `_P` and `_PP` is repeated in other macros, such as the `*VAL` macros you’re about to look at.

The value of type determines which portion of the zval‘s value union will be set. The following piece of code demonstrates a scaled down version of `var_dump()`:

``` c
PHP_FUNCTION(hello_dump) {
    zval *uservar;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", uservar) == FAILURE) {
        RETURN_NULL();
    }
    switch (Z_TYPE_P(uservar)) {
        case IS_NULL:
            php_printf("NULL");
            break;
        case IS_BOOL:
            php_printf("Boolean: %s", Z_LVAL_P(uservar) ? "TRUE" : "FALSE");
            break;
        case IS_LONG:
            php_printf("Long: %ld", Z_LVAL_P(uservar));
            break;
        case IS_DOUBLE:
            php_printf("Double: %f", Z_DVAL_P(uservar));
            break;
        case IS_STRING:
            php_printf("String: ");
            PHPWRITE(Z_STRVAL_P(uservar), Z_STRLEN_P(uservar));
            break;
        case IS_RESOURCE:
            php_printf("Resource");
            break;
        case IS_ARRAY:
            php_printf("Array");
            break;
        case IS_OBJECT:
            php_printf("Object");
            break;
        default:
            php_printf("Unknown");
    }

    RETURN_TRUE;
}
```

As you can see, the `Boolean` data type shares the same internal element as the `long` data type. Just as with `RETURN_BOOL()`, which you used in Part One of this series, `FALSE` is represented by 0, while `TRUE` is represented by 1.

When you use `zend_parse_parameters()` to request a specific data type, such as string, the Zend Engine checks the type of the incoming variable. If it matches, Zend simply passes through the corresponding parts of the zval to the right data types. If it’s of a different type, Zend converts it as is appropriate and/or possible, using its usual type-juggling rules.

Modify the `hello_greetme()` function you implemented earlier by separating it out into smaller pieces:

``` c
PHP_FUNCTION(hello_greetme) {
    zval *zname;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &zname) == FAILURE) {
        RETURN_NULL();
    }
    convert_to_string(zname);
    php_printf("Hello ");
    PHPWRITE(Z_STRVAL_P(zname), Z_STRLEN_P(zname));
    RETURN_TRUE;
}
```

This time, `zend_parse_parameters()` was told to simply retrieve a PHP variable (`zval`) regardless of type, then the function explicitly cast the variable as a string (similar to `$zname = (string)$zname;` ), then `php_printf()` was called using the *STRing VALue* of the `zname` structure. As you’ve probably guessed, other `convert_to_*()` functions exist for bool, long, and double.

### Creating ZVALs

So far, the zvals you’ve worked with have been allocated by the Zend Engine and will be freed the same way. Sometimes, however, it’s necessary to create your own zval. Consider the following block of code:

``` c
{
    zval *temp;
    ALLOC_INIT_ZVAL(temp);
    Z_TYPE_P(temp) = IS_LONG;
    Z_LVAL_P(temp) = 1234;
    zval_ptr_dtor(&temp);
}
```

`ALLOC_INIT_ZVAL()`, as its name implies, allocates memory for a `zval *` and initializes it as a new variable. Once that’s done, the `Z_*_P()` macros can be used to set the type and value of this variable. `zval_ptr_dtor()` handles the dirty work of cleaning up the memory allocated for the variable.

The two `Z_*_P()` calls could have actually been reduced to a single statement: `ZVAL_LONG(temp, 1234);`

Similar macros exist for the other types, and follow the same syntax as the `RETURN_*()` macros you saw in Part One of thise series. In fact the `RETURN_*()` macros are just thin wrappers for `RETVAL_*()` and, by extension, `ZVAL_*()`. The following five versions are all identical:

``` c
RETURN_LONG(42);
```
``` c
RETVAL_LONG(42);
return;
```
``` c
ZVAL_LONG(return_value, 42);
return;
```
``` c
Z_TYPE_P(return_value) = IS_LONG;
Z_LVAL_P(return_value) = 42;
return;
```
``` c
return_value->type = IS_LONG;
return_value->value.lval = 42;
return;
```

If you’re sharp, you’re thinking about the impact of how these macros are defined on the way they’re used in functions like `hello_long()`. “Where does `return_value` come from and why isn’t it being allocated with `ALLOC_INIT_ZVAL()`?”, you might be wondering.

While it may be hidden from you in your day-to-day extension writing, `return_value` is actually a function parameter defined in the prototype of every `PHP_FUNCTION()` definition. The Zend Engine allocates memory for it and initializes it as `NULL` so that even if your function doesn’t explicitly set it, a value will still be available to the calling program. When your internal function finishes executing, it passes that value to the calling program, or frees it if the calling program is written to ignore it.

### Arrays

Since you’ve used PHP in the past, you’ve already recognized an array as a variable whose purpose is to carry around other variables. The way this is represented internally is through a structure known as a `HashTable`. When creating arrays to be returned to PHP, the simplest approach involves using one of the functions listed in *Table 2*.

```
Table 2: zval array creation functions
+-----------------------+-----------------------------------------+--------------------------+
| PHP Syntax            | C Syntax (arr is a zval*) Meaning       |                          |
+-----------------------+-----------------------------------------+--------------------------+
| $arr = array();       | array_init(arr);                        | Initialize a new array   |
+-----------------------+-----------------------------------------+--------------------------+
| $arr[] = NULL;        | add_next_index_null(arr);               | Add a value of a given   |
| $arr[] = 42;          | add_next_index_long(arr, 42);           | type to a numerically    |
| $arr[] = true;        | add_next_index_bool(arr, 1);            | indexed array            |
| $arr[] = 3.14;        | add_next_index_double(arr, 3.14);       |                          |
| $arr[] = 'foo';       | add_next_index_string(arr, "foo", 1);   |                          |
| $arr[] = $myvar;      | add_next_index_zval(arr, myvar);        |                          |
+-----------------------+-----------------------------------------+--------------------------+
| $arr[0] = NULL;       | add_index_null(arr, 0);                 | Add a value of a given   |
| $arr[1] = 42;         | add_index_long(arr, 1, 42);             | type to a specific       |
| $arr[2] = true;       | add_index_bool(arr, 2, 1);              | index in an array        |
| $arr[3] = 3.14;       | add_index_double(arr, 3, 3.14);         |                          |
| $arr[4] = 'foo';      | add_index_string(arr, 4, "foo", 1);     |                          |
| $arr[5] = $myvar;     | add_index_zval(arr, 5, myvar);          |                          |
+-----------------------+-----------------------------------------+--------------------------+
| $arr['abc'] = NULL;   | add_assoc_null(arr, "abc");             | Add a value of a given   |
| $arr['def'] = 711;    | add_assoc_long(arr, "def", 711);        | type to an associatively |
| $arr['ghi'] = true;   | add_assoc_bool(arr, "ghi", 1);          | indexed array            |
| $arr['jkl'] = 1.44;   | add_assoc_double(arr, "jkl", 1.44);     |                          |
| $arr['mno'] = 'baz';  | add_assoc_string(arr, "mno", "baz", 1); |                          |
| $arr['pqr'] = $myvar; | add_assoc_zval(arr, "pqr", myvar);      |                          |
+-----------------------+-----------------------------------------+--------------------------+
```

As with the `RETURN_STRING()` macro, the `add_*_string()` functions take a 1 or a 0 in the final parameter to indicate whether the string contents should be copied. They also have a kissing cousin in the form of an `add_*_stringl()` variant for each. The `l` indicates that the length of the string will be explicitly provided (rather than having the Zend Engine determine this with a call to `strval()`, which is binary-unsafe).

Using this binary-safe form is as simple as specifying the length just before the duplication parameter, like so:

``` c
add_assoc_stringl(arr, "someStringVar", "baz", 3, 1);
```

Using the `add_assoc_*()` functions, all array keys are assumed to contain no `NULL`s – the `add_assoc_*()` functions themselves are not binary-safe with respect to keys. Using keys with `NULL`s in them is discouraged (as it is already a technique used with protected and private object properties), but if doing so is necessary, you’ll learn how you can do it soon enough, when we get into the `zend_hash_*()` functions later.

To put what you’ve just learned into practice, create the following function to return an array of values to the calling program. Be sure to add entries to `php_hello.h` and `hello_functions[]` so as to properly declare this function.

``` c
PHP_FUNCTION(hello_array) {
    char *mystr;
    zval *mysubarray;
    array_init(return_value);
    add_index_long(return_value, 42, 123);
    add_next_index_string(return_value, "I should now be found at index 43", 1);
    add_next_index_stringl(return_value, "I'm at 44!", 10, 1);
    mystr = estrdup("Forty Five");
    add_next_index_string(return_value, mystr, 0);
    add_assoc_double(return_value, "pi", 3.1415926535);
    ALLOC_INIT_ZVAL(mysubarray);
    array_init(mysubarray);
    add_next_index_string(mysubarray, "hello", 1);
    add_assoc_zval(return_value, "subarray", mysubarray);
}
```

Building this extension and issuing `var_dump(hello_array());` gives:

``` php
array(6) {
  [42]=>
  int(123)
  [43]=>
  string(33) "I should now be found at index 43"
  [44]=>
  string(10) "I'm at 44!"
  [45]=>
  string(10) "Forty Five"
  ["pi"]=>
  float(3.1415926535)
  ["subarray"]=>
  array(1) {
    [0]=>
    string(5) "hello"
  }
}
```
Reading values back out of arrays means extracting them as `zval **`s directly from a `HashTable` using the `zend_hash` family of functions from the `ZENDAPI`. Let’s start with a simple function which accepts one array as a parameter:

``` php
function hello_array_strings($arr) {
    if (!is_array($arr))
        return NULL;
    printf("The array passed contains %d elements\n", count($arr));
    foreach($arr as $data) {
        if (is_string($data))
            echo "$data\n";
    }
}
```

Or, in C:

``` c
PHP_FUNCTION(hello_array_strings)
{
    zval *arr, **data;
    HashTable *arr_hash;
    HashPosition pointer;
    int array_count;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &arr) == FAILURE) {
        RETURN_NULL();
    }

    arr_hash = Z_ARRVAL_P(arr);
    array_count = zend_hash_num_elements(arr_hash);

    php_printf("The array passed contains %d elements\n", array_count);

    for(
            zend_hash_internal_pointer_reset_ex(arr_hash, &pointer);
            zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS;
            zend_hash_move_forward_ex(arr_hash, &pointer)
    ) {
        if (Z_TYPE_PP(data) == IS_STRING) {
            PHPWRITE(Z_STRVAL_PP(data), Z_STRLEN_PP(data));
            php_printf("\n");
        }
    }
    RETURN_TRUE;
}
```

In this function only those array elements which are of type string are output, in order to keep the function brief. You may be wondering why we didn’t just use `convert_to_string()` as we did in the `hello_greetme()` function earlier. Let’s give that a shot; replace the for loop above with the following:

``` c
    for(
            zend_hash_internal_pointer_reset_ex(arr_hash, &pointer);
            zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS;
            zend_hash_move_forward_ex(arr_hash, &pointer)
    ) {
        convert_to_string_ex(data);
        PHPWRITE(Z_STRVAL_PP(data), Z_STRLEN_PP(data));
        php_printf("\n");
    }
```

Now compile your extension again and run the following userspace code through it:

``` php
<?php
$a = array('foo',123);
var_dump($a);
hello_array_strings($a);
var_dump($a);
```

Notice that the original array was changed! Remember, the `convert_to_*()` functions have the same effect as calling `set_type()`. Since you’re working with the same array that was passed in, changing its type here will change the original variable. In order to avoid this, you need to first make a copy of the zval. To do this, change that for loop again to the following:

``` c
    for(
        zend_hash_internal_pointer_reset_ex(arr_hash, &pointer);
        zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS;
        zend_hash_move_forward_ex(arr_hash, &pointer)
    ) {
        zval temp;
        temp = **data;
        zval_copy_ctor(&temp);
        convert_to_string(&temp);
        PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
        php_printf("\n");
        zval_dtor(&temp);
    }
```

The more obvious part of this `version – temp = **data` – just copies the data members of the original zval, but since a zval may contain additional resource allocations like `char *` strings, or `HashTable *` arrays, the dependent resources need to be duplicated with `zval_copy_ctor()`. From there it’s just an ordinary convert, print, and a final `zval_dtor()` to get rid of the resources used by the copy.

If you’re wondering why you didn’t do a `zval_copy_ctor()` when we first introduced `convert_to_string()`, it’s because the act of passing a variable into a function automatically performs a copy separating the zval from the original variable. This is only done on the base zval through, so any subordinate resources (such as array elements and object properties) still need to be separated before use.

Now that you’ve seen array values, let’s extend the exercise a bit by looking at the keys as well:

``` c
for(
        zend_hash_internal_pointer_reset_ex(arr_hash, &pointer);
        zend_hash_get_current_data_ex(arr_hash, (void**) &data, &pointer) == SUCCESS;
        zend_hash_move_forward_ex(arr_hash, &pointer)
) {
    zval temp;
    char *key;
    int key_len;
    long index;

    if (zend_hash_get_current_key_ex(arr_hash, &key, &key_len, &index, 0, &pointer) == HASH_KEY_IS_STRING) {
        PHPWRITE(key, key_len);
    } else {
        php_printf("%ld", index);
    }

    php_printf(" => ");

    temp = **data;
    zval_copy_ctor(&temp);
    convert_to_string(&temp);
    PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
    php_printf("\n");
    zval_dtor(&temp);
}
```

Remember that arrays can have numeric indexes, associative string keys, or both. Calling `zend_hash_get_current_key_ex()` makes it possible to fetch either type from the current position in the array, and determine its type based on the return values, which may be any of `HASH_KEY_IS_STRING`, `HASH_KEY_IS_LONG`, or `HASH_KEY_NON_EXISTANT`. Since `zend_hash_get_current_data_ex()` was able to return a `zval **`, you can safely assume that `HASH_KEY_NON_EXISTANT` will not be returned, so only the `IS_STRING` and `IS_LONG` possibilities need to be checked.

There’s another way to iterate through a `HashTable`. The Zend Engine exposes three very similar functions to accommodate this task: `zend_hash_apply()`, `zend_hash_apply_with_argument()`, and `zend_hash_apply_with_arguments()`. The first form just loops through a `HashTable`, the second form allows a single argument to be passed through as a `void *`, while the third form allows an unlimited number of arguments via a `vararg` list. `hello_array_walk()` shows each of these in action:

``` c
static int php_hello_array_walk(zval **element TSRMLS_DC) {
    zval temp;
    temp = **element;

    zval_copy_ctor(&temp);
    convert_to_string(&temp);
    PHPWRITE(Z_STRVAL(temp), Z_STRLEN(temp));
    php_printf("\n");
    zval_dtor(&temp);

    return ZEND_HASH_APPLY_KEEP;
}

static int php_hello_array_walk_arg(zval **element, char *greeting TSRMLS_DC) {
    php_printf("%s", greeting);
    php_hello_array_walk(element TSRMLS_CC);

    return ZEND_HASH_APPLY_KEEP;
}

static int php_hello_array_walk_args(zval **element, int num_args, var_list args, zend_hash_key *hash_key) {
    char *prefix = va_arg(args, char*);
    char *suffix = va_arg(args, char*);
    TSRMLS_FETCH();

    php_printf("%s", prefix);
    php_hello_array_walk(element TSRMLS_CC);
    php_printf("%s\n", suffix);

    return ZEND_HASH_APPLY_KEEP;
}

PHP_FUNCTION(hello_array_walk) {
    zval *zarray;
    int print_newline = 1;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &zarray) == FAILURE) {
        RETURN_NULL();
    }

    zend_hash_apply(Z_ARRVAL_P(zarray), (apply_func_t)php_hello_array_walk TSRMLS_CC);
    zend_hash_apply_with_argument(Z_ARRVAL_P(zarray), (apply_func_arg_t)php_hello_array_walk_arg, "Hello " TSRMLS_CC);
    zend_hash_apply_with_arguments(Z_ARRVAL_P(zarray), (apply_func_args_t)php_hello_array_walk_args, 2, "Hello ", "Welcome to my extension!");

    RETURN_TRUE;
}
```

By now you should be familiar enough with the usage of the functions involved that most of the above code will be obvious. The array passed to `hello_array_walk()` is looped through three times, once with no arguments, once with a single argument, and a third time with two arguments. In this design, the `walk_arg()` and `walk_args()` functions actually rely on the no-argument `walk()` function to do the job of converting and printing the zval, since the job is common across all three.

In this block of code, as in most places where you’ll use `zend_hash_apply()`, the `apply()` functions return `ZEND_HASH_APPLY_KEEP`. This tells the `zend_hash_apply()` function to leave the element in the `HashTable` and continue on with the next one. Other values which can be returned here are: `ZEND_HASH_APPLY_REMOVE`, which does just what it says – removes the current element and continues applying at the next – and `ZEND_HASH_APPLY_STOP`, which will halt the array walk at the current element and exit the `zend_hash_apply()` function completely.

The less familiar component in all this is probably `TSRMLS_FETCH()`. As you may recall from Part One, the `TSRMLS_*` macros are part of the *Thread Safe Resource Management* layer, and are necessary to keep one thread from trampling on another. Because the multi-argument version of `zend_hash_apply()` uses a `vararg` list, the `tsrm_ls` marker doesn’t wind up getting passed into the `walk()` function. In order to recover it for use when we call back into `php_hello_array_walk()`, your function calls `TSRMLS_FETCH()` which performs a lookup to find the correct thread in the resource pool. (Note: This method is substantially slower than passing the argument directly, so use it only when unavoidable.)

Iterating through an array using this foreach-style approach is a common task, but often you’ll be looking for a specific value in an array by index number or by associative key. This next function will return a value from an array passed in the first parameter based on the offset or key specified in the second parameter.

``` c
PHP_FUNCTION(hello_array_value) {
    zval *zarray, *zoffset, **zvalue;
    long index = 0;
    char *key = NULL;
    int key_len = 0;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "az", &zarray, &zoffset) == FAILURE) {
        RETURN_NULL();
    }

    switch (Z_TYPE_P(zoffset)) {
        case IS_NULL:
            index = 0;
            break;
        case IS_DOUBLE:
           index = (long)Z_DVAL_P(zoffset);
            break;
        case IS_BOOL:
        case IS_LONG:
        case IS_RESOURCE:
            index = Z_LVAL_P(zoffset);
            break;
        case IS_STRING:
            key = Z_STRVAL_P(zoffset);
            key_len = Z_STRLEN_P(zoffset);
            break;
        case IS_ARRAY:
            key = "Array";
            key_len = sizeof("Array") - 1;
            break;
        case IS_OBJECT:
            key = "Object";
            key_len = sizeof("Object") - 1;
            break;
        default:
            key = "Unknown";
            key_len = sizeof("Unknown") - 1;
    }

    if (key && zend_hash_find(Z_ARRVAL_P(zarray), key, key_len + 1, (void**)&zvalue) == FAILURE) {
        RETURN_NULL();
    } else if (!key && zend_hash_index_find(Z_ARRVAL_P(zarray), index, (void**)&zvalue) == FAILURE) {
        RETURN_NULL();
    }

    *return_value = **zvalue;
    zval_copy_ctor(return_value);
}
```

This function starts off with a switch block that treats type conversion in much the same way as the Zend Engine would. NULL is treated as 0, Booleans are treated as their corresponding 0 or 1 values, doubles are cast to longs (and truncated in the process) and resources are cast to their numerical value. The treatment of resource types is a hangover from PHP 3, when resources really were just numbers used in a lookup and not a unique type unto themselves.

Arrays and objects are simply treated as a string literal of `Array` or `Object`, since no honest attempt at conversion would actually make sense. The final default condition is put in as an ultra-careful catchall just in case this extension gets compiled against a future version of PHP, which may have additional data types.

Since key is only set to non-NULL if the function is looking for an associative key, it can use that value to decide whether it should use an associative or index based lookup. If the chosen lookup fails, it’s because the key doesn’t exist, and the function therefore returns NULL to indicate failure. Otherwise that zval is copied into `return_value`.

### Symbol Tables as Arrays

If you’ve ever used the `$GLOBALS` array before, you know that every variable you declare and use in the global scope of a PHP script also appears in this array. Recalling that the internal representation of an array is a `HashTable`, one question comes to mind: “Is there some special place where the `GLOBALS` array can be found?” The answer is “Yes”. It’s in the *Executor Globals* structure as `EG(symbol_table)`, which is of type `HashTable` (not `HashTable *`, mind you, just `HashTable`).

You already know how to find associatively keyed elements in an array, and now that you know where to find the global symbol table, it should be a cinch to look up variables from extension code:

`` c
PHP_FUNCTION(hello_get_global_var)
{
    char *varname;
    int varname_len;
    zval **varvalue;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &varname, &varname_len) == FAILURE) {
        RETURN_NULL();
    }

    if (zend_hash_find(&EG(symbol_table), varname, varname_len + 1, (void**)&varvalue) == FAILURE) {
        php_error_docref(NULL TSRMLS_CC, E_NOTICE, "Undefined variable: %s", varname);
        RETURN_NULL();
    }

    *return_value = **varvalue;
    zval_copy_ctor(return_value);
}
```

This should all be intimately familiar to you by now. The function accepts a string parameter and uses that to find a variable in the global scope which it returns as a copy.

The one new item here is `php_error_docref()`. You’ll find this function, or a near sibling thereof, throughout the PHP source tree. The first parameter is an alternate documentation reference (the current function is used by default). Next is the ubiquitous `TSRMLS_CC`, followed by a severity level for the error, and finally there’s a `printf()`-style format string and associated parameters for the actual text of the error message. It’s important to always provide some kind of meaningful error whenever your function reaches a failure condition. In fact, now would be a good time to go back and add an error statement to `hello_array_value()`. The Sanity Check section at the end of this tutorial will include these as well.

In addition to the global symbol table, the Zend Engine also keeps track of a reference to the local symbol table. Since internal functions don’t have symbol tables of their own (why would they need one after all?) the local symbol table actually refers to the local scope of the userland function that called the current internal function. Let’s look at a simple function which sets a variable in the local scope:

``` c
PHP_FUNCTION(hello_set_local_var)
{
    zval *newvar;
    char *varname;
    int varname_len;
    zval *value;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sz", &varname, &varname_len, &value) == FAILURE) {
        RETURN_NULL();
    }

    ALLOC_INIT_ZVAL(newvar);
    *newvar = *value;
    zval_copy_ctor(newvar);
    zend_hash_add(EG(active_symbol_table), varname, varname_len + 1, &newvar, sizeof(zval*), NULL);

    RETURN_TRUE;
}
```

Absolutely nothing new here. Go ahead and build what you’ve got so far and run some test scripts against it. Make sure that what you expect to happen, does happen.

### Reference Counting

So far, the only zvals we’ve added to `HashTables` have been newly created or freshly copied ones. They’ve stood alone, occupying their own resources and living nowhere but that one `HashTable`. As a language design concept, this approach to creating and copying variables is “good enough”, but since you’re accustomed to programming in C, you know that it’s not uncommon to save memory and CPU time by not copying a large block of data unless you absolutely have to. Consider this userspace block of code:

``` php
<?php
    $a = file_get_contents('fourMegabyteLogFile.log');
    $b = $a;
    unset($a);
```

If `$a` were copied to `$b` by doing a `zval_copy_ctor()` (which performs an `estrndup()` on the string contents in the process) then this short script would actually use up eight megabytes of memory to store two identical copies of the four megabyte file. Unsetting `$a` in the final step only adds insult to injury, since the original string gets `efree()`d. Had this been done in C it would have been a simple matter of: `b = a; a = NULL;`

Fortunately, the Zend Engine is a bit smarter than that. When `$a` is first created, an underlying zval is made for it of type string, with the contents of the log file. That zval is assigned to the $a variable through a call to `zend_hash_add()`. When `$a` is copied to `$b`, however, the Engine does something similar to the following:

``` c
{
    zval **value;
    zend_hash_find(EG(active_symbol_table), "a", sizeof("a"), (void**)&value);
    ZVAL_ADDREF(*value);
    zend_hash_add(EG(active_symbol_table), "b", sizeof("b"), value, sizeof(zval*));
}
```

Of course, the real code is much more complex, but the important part to focus on here is `ZVAL_ADDREF()`. Remember that there are four principle elements in a zval. You’ve already seen type and value; this time you’re working with `refcount`. As the name may imply, `refcount` is a counter of the number of times a particular zval is referenced in a symbol table, array, or elsewhere.

When you used `ALLOC_INIT_ZVAL()`, `refcount` this was set to 1 so you didn’t have to do anything with it in order to return it or add it into a `HashTable` a single time. In the code block above, you’ve retrieved a zval from a `HashTable`, but not removed it, so it has a `refcount` which matches the number of places it’s already referenced from. In order to reference it from another location, you need to increase its reference count.

When the userspace code calls `unset($a)`, the Engine performs a `zval_ptr_dtor()` on the variable. What you didn’t see, in prior uses of `zval_ptr_dtor()`, is that this call doesn’t necessarily destroy the zval and all its contents. What it actually does is decrease its refcount. If, and only if, the `refcount` reaches zero, does the Zend Engine destroy the zval...

### Copies versus References

There are two ways to reference a zval. The first, demonstrated above, is known as copy-on-write referencing. The second form, full referencing, is what a userspace script writer is more familiar with in relationship to the word “reference” and occurs with userspace code like: `$a = &$b;`.

In a zval, these two types are differentiated with the member value `is_ref`, which will be 0 for copy references, and non-zero for full references. Note that it’s not possible for a zval to be both a copy style reference and a full reference. So if a variable starts off being `is_ref`, and is then assigned to a new variable as a copy, a full copy must be performed. Consider the following userspace code:

``` php
<?php
    $a = 1;
    $b = &$a;
    $c = $a;
```

In this block of code, a zval is created for `$a`, initially with `is_ref` set to 0 and `refcount` set to 1. When `$a` is referenced to `$b`, `is_ref` gets changed to 1, and `refcount` increases to 2. When a copy is made into `$c`, the Zend Engine can’t just simply increase the `refcount` to 3 since `$c` would then be seen as a full reference to `$a`. Turning off `is_ref` wouldn’t work either since `$b` would now be seen as a copy of `$a`, rather than a reference. So at this point a new zval is allocated, and the value of the original is copied into it with `zval_copy_ctor()`. The original zval is left with `is_ref==1` and `refcount==2`, while the new zval has `is_ref=0` and `refcount=1`. Now look at that same code block done in a slightly different order:

``` php
<?php
    $a = 1;
    $c = $a;
    $b = &$a;
```

The end result is the same, with `$b` being a full reference to `$a`, and `$c` being a copy of `$a`. This time, however, the internal actions are slightly different. As before, a new zval is created for $a at the beginning with `is_ref==0` and `refcount=1`. The `$c = $a;` statement then assigns the same zval to the `$c` variable while incrementing the `refcount` to 2, and leaving `is_ref` as 0. When the Zend Engine encounters `$b = &$a`; it wants to just set `is_ref` to 1, but of course can’t since that would impact `$c`. Instead it creates a new zval and copies the contents of the original into it with `zval_copy_ctor()`, then decrements refcount in the original zval to signify that `$a` is no longer using that zval. Instead, it sets the new zval‘s `is_ref` to 1, its refcount to 2, and updates the `$a` and `$b` variables to refer to it.

## Extension Writing Part III: Resources

### Introduction

Up until now, you’ve worked with concepts that are familiar and map easily to
userspace analogies. In this tutorial, you’ll dig into the inner workings of a
more alien data type – completely opaque in userspace, but with behavior that
should ultimately inspire a sense of déjà vu.

### Resources

While a PHP zval can represent a wide range of internal data types, one data
type that is impossible to represent fully within a script is the pointer.
Representing a pointer as a value becomes even more difficult when the structure
your pointer references is an opaque typedef. Since there’s no meaningful way to
present these complex structures, there’s also no way to act upon them
meaningfully using traditional operators. The solution to this problem is to
simply refer to the pointer by an essentially arbitrary label called a resource.

In order for the resource’s label to have any kind of meaning to the Zend Engine,
its underlying data type must first be registered with PHP. You’ll start out by
defining a simple data structure in php_hello.h. You can place it pretty much
anywhere but, for the sake of this exercise, put it after the #define statements,
and before the `PHP_MINIT_FUNCTION` declaration. You’re also defining a constant,
which will be used for the resource’s name as shown during a call to `var_dump()`.

``` c
typedef struct _php_hello_person {
    char *name;
    int name_len;
    long age;
} php_hello_person;
#define PHP_HELLO_PERSON_RES_NAME "Person Data"
```

Now, open up `hello.c` and add a true global integer before your
`ZEND_DECLARE_MODULE_GLOBALS` statement:

``` c
int le_hello_person;
```

List entry identifiers (`le_*`) are one of the very few places where you’ll
declare true, honest to goodness global variables within a PHP extension. These
values are simply used with a lookup table to associate resource types with
their textual names and their destructor methods, so there’s nothing about them
that needs to be threadsafe. Your extension will generate a unique number for
each resource type it exports during the MINIT phase. Add that to your extension
now, by placing the following line at the top of `PHP_MINIT_FUNCTION(hello)`:

``` c
le_hello_person = zend_register_list_destructors_ex(NULL, NULL, PHP_HELLO_PERSON_RES_NAME, module_number);
```

### Initializing Resources

Now that you’ve registered your resource, you need to do something with it. Add
the following function to hello.c, along with its matching entry in the
`hello_functions` structure, and as a prototype in `php_hello.h`:

``` c
PHP_FUNCTION(hello_person_new)
{
    php_hello_person *person;
    char *name;
    int name_len;
    long age;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sl", &name, &name_len, &age) == FAILURE) {
        RETURN_FALSE;
    }

    if (name_len < 1) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "No name given, person resource not created.");
        RETURN_FALSE;
    }

    if (age < 0 || age > 255) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "Nonsense age (%d) given, person resource not created.", age);
        RETURN_FALSE;
    }

    person = emalloc(sizeof(php_hello_person));
    person->name = estrndup(name, name_len);
    person->name_len = name_len;
    person->age = age;

    ZEND_REGISTER_RESOURCE(return_value, person, le_hello_person);
}
```

Before allocating memory and duplicating data, this function performs a few
sanity checks on the data passed into the resource: Was a name provided? Is this
person’s age even remotely within the realm of a human lifespan? Of course,
anti-senescence research could make the data type for age (and its sanity-checked
limits) seem like the Y2K bug someday, but it’s probably safe to assume no-one
will be older than 255 anytime soon.

Once the function has satisfied its entrance criteria, it’s all down to
allocating some memory and putting the data where it belongs. Lastly,
`return_value` is populated with a newly registered resource. This function
doesn’t need to understand the internals of the data struct; it only needs to
know what its pointer address is, and what resource type that data is associated
with.

### Accepting Resources as Function Parameters

From the previous tutorial in this series, you already know how to use
`zend_parse_parameters()` to accept a resource parameter. Now it’s time to apply
that to recovering the data that goes with a given resource. Add this next
function to your extension:

``` c
PHP_FUNCTION(hello_person_greet)
{
    php_hello_person *person;
    zval *zperson;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "r", &zperson) == FAILURE) {
        RETURN_FALSE;
    }

    ZEND_FETCH_RESOURCE(person, php_hello_person*, &zperson, -1, PHP_HELLO_PERSON_RES_NAME, le_hello_person);

    php_printf("Hello ");
    PHPWRITE(person->name, person->name_len);
    php_printf("!\nAccording to my records, you are %d years old.\n", person->age);

    RETURN_TRUE;
}
```

The important parts of the functionality here should be easy enough to parse.
`ZEND_FETCH_RESOURCE()` wants a variable to drop the pointer value into. It also
wants to know what the variable’s internal type should look like, and it needs
to know where to get the resource identifier from.

The -1 in this function call is an alternative to using &zperson to identify the
resource. If any numeric value is provided here other than -1, the Zend Engine
will attempt to use that number to identify the resource rather than the `zval*`
parameter’s data. If the resource passed does not match the resource type
specified by the last parameter, an error will be generated using the resource
name given in the second to last parameter.

There is more than one way to skin a resource though. In fact the following four
code blocks are all effectively identical:

``` c
ZEND_FETCH_RESOURCE(person, php_hello_person *, &zperson, -1, PHP_HELLO_PERSON_RES_NAME, le_person_name);
ZEND_FETCH_RESOURCE(person, php_hello_person *, NULL, Z_LVAL_P(zperson), PHP_HELLO_PERSON_RES_NAME, le_person_name);

person = (php_hello_person *) zend_fetch_resource(&zperson TSRMLS_CC, -1, PHP_HELLO_PERSON_RES_NAME, NULL, 1, le_person_name);
ZEND_VERIFY_RESOURCE(person);

person = (php_hello_person *) zend_fetch_resource(&zperson TSRMLS_CC, -1, PHP_HELLO_PERSON_RES_NAME, NULL, 1, le_person_name);
if (!person) {
    RETURN_FALSE;
}
```

The last couple of forms are useful in situations where you’re not in a
`PHP_FUNCTION()`, and therefore have no `return_value` to assign; or when it’s
perfectly reasonable for the resource type to not match, and simply returning
`FALSE` is not what you want.

However you choose to retrieve your resource data from the parameter, the result
is the same. You now have a familiar C struct that can be accessed in exactly
the same way as you would any other C program. At this point the struct still
‘belongs’ to the resource variable, so your function shouldn’t free the pointer
or change reference counts prior to exiting. So how are resources destroyed?

### Destroying Resources

Most PHP functions that create resource parameters have matching functions to
free those resources. For example, `mysql_connect()` has `mysql_close()`,
`mysql_query()` has `mysql_free_result()`, `fopen()` has `fclose()`, and so on
and so forth. Experience has probably taught you that if you simply `unset()`
variables containing resource values, then whatever real resource they’re
attached to will also be freed/closed. For example:

``` php
<?php
    $fp = fopen('foo.txt','w');
    unset($fp);
```

The first line of this snippet opens a file for writing, `foo.txt`, and assigns
the stream resource to the variable `$fp`. When the second line unsets `$fp`,
PHP automatically closes the file – even though fclose() was never called. How
does it do that?

The secret lies in the `zend_register_resource()` call you made in your `MINIT`
function. The two `NULL` parameters you passed correspond to cleanup (or dtor)
functions. The first is called for ordinary resources, and the second for
persistent ones. We’ll focus on ordinary resources for now and come back to
persistent resources later on, but the general semantics are the same. Modify
your zend_register_resource line as follows:

``` c
le_hello_person = zend_register_list_destructors_ex(php_hello_person_dtor, NULL, PHP_HELLO_PERSON_RES_NAME, module_number);
```
and create a new function located just above the `MINIT` method:

``` c
static void php_hello_person_dtor(zend_rsrc_list_entry *rsrc TSRMLS_DC)
{
    php_hello_person *person = (php_hello_person*)rsrc->ptr;
    if (person) {
        if (person->name) {
            efree(person->name);
        }
        efree(person);
    }
}
```

As you can see, this simply frees any allocated buffers associated with the
resource. When the last userspace variable containing a reference to your
resource goes out of scope, this function will be automatically called so that
your extension can free memory, disconnect from remote hosts, or perform other
last minute cleanup.

### Destroying a Resource by Force

If calling a resource’s dtor function depends on all the variables pointing to
it going out of scope, then how do functions like `fclose()` or
`mysql_free_result()` manage to perform their job while references to the
resource still exist? Before I answer that question, I’d like you to try out the
following:

``` php
<?php
$fp = fopen('test', 'w');

var_dump($fp);
fclose($fp);
var_dump($fp);
```

In both calls to `var_dump()`, you can see the numeric value of the resource
number, so you know that a reference to your resource still exists; yet the
second call to `var_dump()` claims the type is ‘unknown’. This is because the
resource lookup table which the Zend Engine keeps in memory, no longer contains
the file handle to match that number – so any attempt to perform a
`ZEND_FETCH_RESOURCE()` using that number will fail.

`fclose()`, like so many other resource-based functions, accomplishes this by
using `zend_list_delete()`. Perhaps obviously, perhaps not, this function
deletes an item from a list, specifically a resource list. The simplest use of
this would be:

``` c
PHP_FUNCTION(hello_person_delete)
{
    zval *zperson;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "r", &zperson) == FAILURE) {
        RETURN_FALSE;
    }

    zend_list_delete(Z_LVAL_P(zperson));
    RETURN_TRUE;
}
```

Of course, this function will destroy *any* resource type regardless of whether
it’s our person resource, a file handle, a MySQL connection, or whatever. In
order to avoid causing potential trouble for other extensions and making
userspace code harder to debug, it is considered good practice to first verify
the resource type. This can be done easily by fetching the resource into a dummy
variable using `ZEND_FETCH_RESOURCE()`. Go ahead and add that to your function,
between the `zend_parse_parameters()` call and `zend_list_delete()`.

### Persistent Resources

If you’ve used `mysql_pconnect()`, `popen()` or any of the other persistent
resource types, then you’ll know that it’s possible for a resource to stick
around, not just after all the variables referencing it have gone out of scope,
but even after a request completes and a new one begins. These resources are
called persistent resources, because they persist throughout the life of the
SAPI unless deliberately destroyed.

The two key differences between standard resources and persistent ones are the
placement of the dtor function when registering, and the use of `pemalloc()`
rather than `emalloc()` for data allocation.

Let’s build a version of our the person resource that can remain persistent.
Start by adding another `zend_register_resource()` line to `MINIT`. Don’t forget
to define the `le_hello_person_persist` variable next to `le_hello_person`:


```
PHP_MINIT_FUNCTION(hello)
{
    le_hello_person = zend_register_list_destructor_ex(php_hello_person_dtor, NULL, PHP_HELLO_PERSON_RES_NAME, module_number);
    le_hello_person_persist = zend_register_list_destructor_ex (NULL, php_hello_person_persist_dtor, PHP_HELLO_PERSON_RES_NAME, module_number);
    ...
```

The basic syntax is the same, but this time you’ve specified the destructor
function in the second parameter to `zend_register_resource()` as opposed to the
first. All that really distinguishes one of these from the other is when the
dtor function is actually called. A dtor function passed in the first parameter
is called with the active request shutdown, while a dtor function passed in the
second parameter isn’t called until the module is unloaded during final shutdown.

Since you’ve referenced a new resource dtor function, you’ll need to define it.
Adding this familiar looking method to `hello.c` somewhere above the `MINIT`
function should do the trick:

``` c
static void php_hello_person_persist_dtor(zend_rsrc_list_entry *rsrc TSRMLS_DC)
{
    php_hello_person *person = (php_hello_person*)rsrc->ptr;
    if (person) {
        if (person->name) {
            pefree(person->name, 1);
    }
        pefree(person, 1);
    }
}
```

Now you need a way to instantiate a persistent version of the person resource.
The established convention is to create a new function with a ‘p’ prefix in the
name. Add this function to your extension:

``` c
PHP_FUNCTION(hello_person_pnew)
{
    php_hello_person *person;
    char *name;
    int name_len;
    long age;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sl", &name, &name_len, &age) == FAILURE) {
        RETURN_FALSE;
    }

    if (name_len < 1) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "No name given, person resource not created.");
        RETURN_FALSE;
    }

    if (age < 0 || age > 255) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "Nonsense age (%d) given, person resource not created.", age);
        RETURN_FALSE;
    }

    person = pemalloc(sizeof(php_hello_person), 1);
    person->name = pemalloc(name_len + 1, 1);
    memcpy(person->name, name, name_len + 1);
    person->name_len = name_len;
    person->age = age;

    ZEND_REGISTER_RESOURCE(return_value, person, le_hello_person_persist);
}
```

As you can see, this function differs only slightly from `hello_person_new()`. In practice, you’ll typically see these kinds of paired userspace functions implemented as wrapper functions around a common core. Take a look through the source at other paired resource creation functions to see how this kind of duplication is avoided.

Now that your extension is creating both types of resources, it needs to be able to handle both types. Fortunately, `ZEND_FETCH_RESOURCE` has a sister function that is up to the task. Replace your current call to `ZEND_FETCH_RESOURCE` in `hello_person_greet()` with the following:

``` c
ZEND_FETCH_RESOURCE2(person, php_hello_person*, &zperson, -1, PHP_HELLO_PERSON_RES_NAME , le_hello_person, le_hello_person_persist);
```

This will load your person variable with appropriate data, regardless of whether or not a persistent resource was passed.

The functions these two FETCH macros call will actually allow you to specify any number of resource types, but it’s rare to need more than two. Just in case,s here’s the last statement rewritten using the base function:

``` c
person = (php_hello_person*) zend_fetch_resource(&zperson TSRMLS_CC, -1, PHP_HELLO_PERSON_RES_NAME, NULL, 2, le_hello_person, le_hello_person_persist);
ZEND_VERIFY_RESOURCE(person);
```

There are two important things to notice here. Firstly, you can see that the `FETCH_RESOURCE` macros automatically attempt to verify the resource. Expanded out, the `ZEND_VERIFY_RESOURCE` macro in this case simply translates to:

``` c
if (!person) {
    RETURN_FALSE;
}
```

Of course, you don’t always want your extension function to exit just because a resource couldn’t be fetched, so you can use the real `zend_fetch_resource()` function to try to fetch the resource type, but then use your own logic to deal with NULL values being returned.

### Finding Existing Persistent Resources

A persistent resource is only as good as your ability to reuse it. In order to reuse it, you’ll need somewhere safe to store it. The Zend Engine provides for this through the `EG(persistent_list)` executor global, a `HashTable` containing `list_entry` structures which is normally used internally by the Eengine. Modify `hello_person_pnew()` according to the following:

``` c
PHP_FUNCTION(hello_person_pnew)
{
    php_hello_person *person;
    char *name, *key;
    int name_len, key_len;
    long age;
    list_entry *le, new_le;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sl", &name, &name_len, &age) == FAILURE) {
        RETURN_FALSE;
    }

    if (name_len < 1) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "No name given, person resource not created.");
        RETURN_FALSE;
    }

    if (age < 0 || age > 255) {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "Nonsense age (%d) given, person resource not created.", age);
        RETURN_FALSE;
    }

    /* Look for an established resource */
    key_len = spprintf(&key, 0, "hello_person_%s_%d\n", name, age);
    if (zend_hash_find(&EG(persistent_list), key, key_len + 1, &le) == SUCCESS) {
        /* An entry for this person already exists */
        ZEND_REGISTER_RESOURCE(return_value, le->ptr, le_hello_person_persist);
        efree(key);
        return;
    }

    /* New person, allocate a structure */
    person = pemalloc(sizeof(php_hello_person), 1);
    person->name = pemalloc(name_len + 1, 1);
    memcpy(person->name, name, name_len + 1);
    person->name_len = name_len;
    person->age = age;

    ZEND_REGISTER_RESOURCE(return_value, person, le_hello_person_persist);

    /* Store a reference in the persistence list */
    new_le.ptr = person;
    new_le.type = le_hello_person_persist;
    zend_hash_add(&EG(persistent_list), key, key_len + 1, &new_le, sizeof(list_entry), NULL);

    efree(key);
}
```

This version of `hello_person_pnew()` first checks for an existing `php_hello_person` structure in the `EG(persistent_list)` global and, if available, uses that rather than waste time and resources on reallocating it. If it does not exist yet, the function allocates a new structure populated with fresh data and adds that structure to the persistent list instead. Either way, the function leaves you with a new structure registered as a resource within the request.

The persistent list used to store pointers is always local to the current process or thread, so there’s never any concern that two requests might be looking at the same data at the same time. If one process deliberately closes a persistent resource PHP will handle it, removing the reference to that resource from the persistent list so that future invocations don’t try to use the freed data.
