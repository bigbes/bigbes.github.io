# Objects

**branch: oop**

This section will cover creating objects. Objects are like associative arrays++, they allow you to attach almost any functionality you want to a PHP variable.

You can create an object in much the same way that you’d create an array:

``` c
PHP_FUNCTION(makeObject) {
    object_init(return_value);

    // add a couple of properties
    zend_update_property_string(NULL, return_value, "name", strlen("name"), "yig" TSRMLS_CC);
    zend_update_property_long(NULL, return_value, "worshippers", strlen("worshippers"), 4 TSRMLS_CC);
}
```

If you call `var_dump(makeObject())`, you’ll see something like:

``` php
object(stdClass)#1 (2) {
  ["name"]=>
  string(3) "yig"
  ["worshippers"]=>
  int(4)
}
```

# Classes

**branch: cultists**

You create a class by designing a class template, stored in a zend_class_entry.

For our extension, we’ll make a new class, Cultist. We want a standard cultist template, but every individual cultist is unique.

I like to give each class its own C file to keep things tidy, but that’s not necessary if it’s more logical to group them together or something. However, it’s my tutorial, so we’re splitting it out.

Add two new files to your extension directory: `cultist.c` and `cultist.h`. Add the new C file to your `config.m4`, so it will get compiled into your extension:

``` m4
PHP_NEW_EXTENSION(rlyeh, php_rlyeh.c cultist.c, $ext_shared)
```

Note that there is no comma between `php_rlyeh.c` and `cultist.c`.

Now we want to add our Cultist class. Open up cultist.c and add the following code:

``` c
#include <php.h>
#include "cultist.h"
 
zend_class_entry *rlyeh_ce_cultist;
 
static function_entry cultist_methods[] = {
  PHP_ME(Cultist, sacrifice, NULL, ZEND_ACC_PUBLIC)
  {NULL, NULL, NULL}
};
 
void rlyeh_init_cultist(TSRMLS_D) {
  zend_class_entry ce;
 
  INIT_CLASS_ENTRY(ce, "Cultist", cultist_methods);
  rlyeh_ce_cultist = zend_register_internal_class(&ce TSRMLS_CC);
 
  /* fields */
  zend_declare_property_bool(rlyeh_ce_cultist, "alive", strlen("alive"), 1, ZEND_ACC_PUBLIC TSRMLS_CC);
}
 
PHP_METHOD(Cultist, sacrifice) {
  // TODO            
}
```

You might recognize the function_entry struct from our original extension: methods are just grouped into function_entrys per class.

The real meat-and-potatoes is in `rlyeh_init_cultist`. This function defines the class entry for cultist, giving it methods (`cultist_methods`), constants, and properties.

There are tons of flags that can be set for methods and properties. Some of the most common are:

``` c
ZEND_ACC_STATIC
ZEND_ACC_PUBLIC
ZEND_ACC_PROTECTED
ZEND_ACC_PRIVATE
ZEND_ACC_CTOR
ZEND_ACC_DTOR
ZEND_ACC_DEPRECATED
```

Currently we’re just using `ZEND_ACC_PUBLIC` for our sacrifice function, but this could be OR-ed with any of the other flags (for example, if we decided `sacrifice2()` had a better API, we could change sacrifice‘s flags to `ZEND_ACC_PUBLIC|ZEND_ACC_DEPRECATED` and PHP would warn the user if they tried to use it).

In `cultist.h`, define all of the functions used above:

``` c
#ifndef CULTIST_H
#define CULTIST_H
 
void rlyeh_init_cultist(TSRMLS_D);
 
PHP_METHOD(Cultist, sacrifice);
 
#endif
```

Now we have to tell the extension to load this class on startup. Thus, we want to call `rlyeh_init_cultist` in our MINIT function and include the cultist.h header file. Open up `php_rlyeh.c` and add the following:

``` c
// at the top
#include "cultist.h"
 
// our existing MINIT function from part 3
PHP_MINIT_FUNCTION(rlyeh) {
  rlyeh_init_cultist(TSRMLS_C);
}
```

Because we changed `config.m4`, we have to do `phpize && ./configure && make install`, not just `make install`, otherwise `cultist.c` won’t be added to the `Makefile`.

Now if we run `var_dump(new Cultist());`, we will see something like:

``` php
object(Cultist)#1 (1) {
  ["alive"]=>
  bool(true)
}
```

Creating a new class instance

We can also initialize cultists from C. Let’s add a static function to create a cultist. Open `cultist.c` and add the following:

``` c
static function_entry cultist_methods[] = {
  PHP_ME(Cultist, sacrifice, NULL, ZEND_ACC_PUBLIC)
  PHP_ME(Cultist, createCultist, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_STATIC)
  {NULL, NULL, NULL}
};

PHP_METHOD(Cultist, createCultist) {
   object_init_ex(return_value, rlyeh_ce_cultist);
}
```

Now we can call `Cultist::createCultist()` to create a new cultist.

What if creating new cultists takes some setup, so we’d like to have a constructor? Well, the constructor is just a method, so we can add that:

``` c
static function_entry cultist_methods[] = {
  PHP_ME(Cultist, __construct, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_CTOR)
  PHP_ME(Cultist, sacrifice, NULL, ZEND_ACC_PUBLIC)
  PHP_ME(Cultist, createCultist, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_STATIC)
  {NULL, NULL, NULL}
};
 
PHP_METHOD(Cultist, __construct) {
  // do setup
}
```

Now PHP will automatically call our `Cultist::__construct` when we call new Cultist. However, `createCultist` won’t: it’ll just set the defaults and return. We have to modify createCultist to call a PHP method from C.

# CALLING METHOD-TO-METHOD

**branch: m2m**

First, add this enormous block to your `php_rlyeh.h` file:

``` c
#define PUSH_PARAM(arg) zend_vm_stack_push(arg TSRMLS_CC)
#define POP_PARAM() (void)zend_vm_stack_pop(TSRMLS_C)
#define PUSH_EO_PARAM()
#define POP_EO_PARAM()
 
#define CALL_METHOD_BASE(classname, name) zim_##classname##_##name
 
#define CALL_METHOD_HELPER(classname, name, retval, thisptr, num, param)      \
  PUSH_PARAM(param); PUSH_PARAM((void*)num);                                  \
  PUSH_EO_PARAM();                                                            \
  CALL_METHOD_BASE(classname, name)(num, retval, NULL, thisptr, 0 TSRMLS_CC); \
  POP_EO_PARAM();                                                             \
  POP_PARAM(); POP_PARAM();
 
#define CALL_METHOD(classname, name, retval, thisptr)                     \
  CALL_METHOD_BASE(classname, name)(0, retval, NULL, thisptr, 0 TSRMLS_CC);
 
#define CALL_METHOD1(classname, name, retval, thisptr, param1)   \
  CALL_METHOD_HELPER(classname, name, retval, thisptr, 1, param1);
 
#define CALL_METHOD2(classname, name, retval, thisptr, param1, param2) \
  PUSH_PARAM(param1);                                                  \
  CALL_METHOD_HELPER(classname, name, retval, thisptr, 2, param2);     \
  POP_PARAM();
 
#define CALL_METHOD3(classname, name, retval, thisptr, param1, param2, param3) \
  PUSH_PARAM(param1); PUSH_PARAM(param2);                                      \
  CALL_METHOD_HELPER(classname, name, retval, thisptr, 3, param3);             \
  POP_PARAM(); POP_PARAM();
```

These macros let you call PHP functions from C.

Add the following to `cultist.c`:

``` c
#include "php_rlyeh.h"

PHP_METHOD(Cultist, createCultist) {
  object_init_ex(return_value, rlyeh_ce_cultist);
  CALL_METHOD(Cultist, __construct, return_value, return_value);
}
```

# THIS
**branch: this**

We’ve pretty much just been dealing with `return_values`, but now that we’re working with objects we can also access **this**. To get **this**, use the `getThis()` macro.

For example, suppose we want to set a couple of properties in the constructor:

``` c
PHP_METHOD(Cultist, __construct) {
  char *name;
  int name_len;
  // defaults                                                                                                                                                               
  long health = 10, sanity = 4;
 
  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s|ll", &name, &name_len, &health, &sanity) == FAILURE) {
    return;
  }
 
  zend_update_property_stringl(rlyeh_ce_cultist, getThis(), "name", strlen("name"), name, name_len TSRMLS_CC);
  zend_update_property_long(rlyeh_ce_cultist, getThis(), "health", strlen("health"), health TSRMLS_CC);
  zend_update_property_long(rlyeh_ce_cultist, getThis(), "sanity", strlen("sanity"), sanity TSRMLS_CC);
}
```

Note the `zend_parse_parameters` argument: `“s|ll”`. The pipe character (`“|”`) means, “every argument after this is optional.” Thus, at least 1 argument is required (in this case, the cultist’s name), but health and sanity are optional.

Now, if we create a new cultist, we get something like:

``` php
$ php -r 'var_dump(new Cultist("Todd"));'
object(Cultist)#1 (4) {
  ["alive"]=>
  bool(true)
  ["name"]=>
  string(4) "Todd"
  ["health"]=>
  int(10)
  ["sanity"]=>
  int(4)
}
```

# Attaching Structs

As mentioned earlier, you can attach a struct to an object. This lets the object carry around some information that is invisible to PHP, but usable to your extension.

You have to set up the `zend_class_entry` in a special way when you create it. First, add the struct to your `cultist.h` file, as well as the extra function declaration we’ll be using:

``` c
typedef struct _cult_secrets {
    // required
    zend_object std;
 
    // actual struct contents
    int end_of_world;
    char *prayer;
} cult_secrets;
 
zend_object_value create_cult_secrets(zend_class_entry *class_type TSRMLS_DC);
void free_cult_secrets(void *object TSRMLS_DC);
// existing init function
void rlyeh_init_cultist(TSRMLS_D) {
  zend_class_entry ce;
 
  INIT_CLASS_ENTRY(ce, "Cultist", cultist_methods);
  // new line!
  ce.create_object = create_cult_secrets;
  rlyeh_ce_cultist = zend_register_internal_class(&ce TSRMLS_CC);
 
  /* fields */
  zend_declare_property_bool(rlyeh_ce_cultist, "alive", strlen("alive"), 1, ZEND_ACC_PUBLIC TSRMLS_CC);
}
 
zend_object_value create_cult_secrets(zend_class_entry *class_type TSRMLS_DC) {
  zend_object_value retval;
  cult_secrets *intern;
  zval *tmp;
 
  // allocate the struct we're going to use
  intern = (cult_secrets*)emalloc(sizeof(cult_secrets));
  memset(intern, 0, sizeof(cult_secrets));
 
  // create a table for class properties
  zend_object_std_init(&intern->std, class_type TSRMLS_CC);
  zend_hash_copy(intern->std.properties,
     &class_type->default_properties,
     (copy_ctor_func_t) zval_add_ref,
     (void *) &tmp,
     sizeof(zval *));
 
  // create a destructor for this struct
  retval.handle = zend_objects_store_put(intern, (zend_objects_store_dtor_t) zend_objects_destroy_object, free_cult_secrets, NULL TSRMLS_CC);
  retval.handlers = zend_get_std_object_handlers();
 
  return retval;
}
 
// this will be called when a Cultist goes out of scope
void free_cult_secrets(void *object TSRMLS_DC) {
  cult_secrets *secrets = (cult_secrets*)object;
  if (secrets->prayer) {
    efree(secrets->prayer);
  }
  efree(secrets);
}
```

If we want to access this, we can fetch the struct from `getThis()` with something like:

``` c
PHP_METHOD(Cultist, getDoomsday) {
  cult_secrets *secrets;
 
  secrets = (cult_secrets*)zend_object_store_get_object(getThis() TSRMLS_CC);
 
  RETURN_LONG(secrets->end_of_world);
}
```

# Exceptions

** branch: exceptions **

All exceptions must descend from the base PHP Exception class, so this is also an intro to class inheritance.

Aside from extending Exception, custom exceptions are just normal classes. So, to create a new one, open up `php_rlyeh.c` and add the following:

``` c
// include exceptions header
#include <zend_exceptions.h>
 
zend_class_entry *rlyeh_ce_exception;
 
void rlyeh_init_exception(TSRMLS_D) {
  zend_class_entry e;
 
  INIT_CLASS_ENTRY(e, "MadnessException", NULL);
  rlyeh_ce_exception = zend_register_internal_class_ex(&e, (zend_class_entry*)zend_exception_get_default(TSRMLS_C), NULL TSRMLS_CC);
}
 
PHP_MINIT_FUNCTION(rlyeh) {
  rlyeh_init_exception(TSRMLS_C);
}
```

Don’t forget to declare `rlyeh_init_exception` in `php_rlyeh.h`.

Note that we could add our own methods to `MadnessException` with the third argument to `INIT_CLASS_ENTRY`, but we’ll just leave it with the default exception methods it inherits from `Exception`.

# Throwing Exceptions

An exception isn’t much good unless we can throw it. Let’s add a method that can throw it:

``` c
zend_function_entry rlyeh_functions[] = {
  PHP_FE(cthulhu, NULL)
  PHP_FE(lookAtMonster, NULL)
  { NULL, NULL, NULL }
};
 
PHP_FUNCTION(lookAtMonster) {
  zend_throw_exception(rlyeh_ce_exception, "looked at the monster too long", 1000 TSRMLS_CC);
}
```

The 1000 is the exception code, you can set that to whatever you want (users can access it from the exception with the `getCode()` method).

Now, if we compile and install, we can run `lookAtMonster()` and we’ll get:

``` php
Fatal error: Uncaught exception 'MadnessException' with message 'looked at the monster too long' in Command line code:1
Stack trace:
#0 Command line code(1): lookAtMonster()
#1 {main}
  thrown in Command line code on line 1
```
