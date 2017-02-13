<!-- MarkdownTOC -->

- Using Variables
  - Working with input
  - A note on return values
- Zvals
  - Accessing array elements
  - Iterating through arrays

<!-- /MarkdownTOC -->


# Using Variables

In the previous sections, we got PHP set up and created our first extension. In this section, we’ll look at how to use more of the PHP API.

## Working with input

**branch: zend_parse_parameters**

Our existing extension is nice, but it isn’t very interactive. We can modify this function to accept variables as arguments using the `zend_parse_parameters` function:

``` c
PHP_FUNCTION(cthulhu) {
    // boolean type
    zend_bool english = 0;
 
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "b", &english) == FAILURE) {
        return;
    }
 
    if (english) {
        php_printf("In his house at R'lyeh dead Cthulhu waits dreaming.\n");
    }
    else {
        php_printf("Ph'nglui mglw'nafh Cthulhu R'lyeh wgah'nagl fhtagn.\n");
    }
}
```

Try re-compiling the extension and calling `cthulhu(true);` and `cthulhu(false);`.

If you try calling `cthulhu();` (no arguments), you’ll notice that `zend_parse_parameters` takes care of warning you about it:

``` bash
$ php -r 'cthulhu();'
```

Warning: `cthulhu()` expects exactly 1 parameter, 0 given in Command line code on line 1

## A note on return values

`zend_parse_parameters` and many other PHP API function return `SUCCESS` or `FAILURE`, which are int values. Irritatingly, `SUCCESS` is 0 (false in C) and `FAILURE` is -1 (true in C)! So, you generally can’t say `if (some_php_api_func())`, you have to say `if (some_php_api_func() == SUCCESS)`.

`zend_parse_parameters` input

The parameters passed to `zend_parse_parameters` are:

`ZEND_NUM_ARGS()` - The number of arguments passed in (you can hard-code this, but using `ZEND_NUM_ARGS()` will automatically grab that info for you).
`TSRMLS_CC` - You’ll see this magic variable all over the place in PHP extensions. It’s a macro that defines `, <thread_info> ` (or ` ` if threading is disabled). Note that, because it includes a comma, there’s no comma between `ZEND_NUM_ARGS()` and `TSRMLS_CC`. You don’t have to worry about it or do anything with it, just pass it around.
`b` - This is a string describing the arguments you expect. Common values are:

```
+---+----------+----------------+
| b | boolean  | zend_bool      |
| s | string   | char * and int |
| l | long     | long           |
| d | double   | double         |
| a | array    | zval *         |
| o | object   | zval *         |
| z | any type | zval *         |
+---+----------+----------------+
```
Except for `b`, `l`, and `d`, `zend_parse_parameters` does not create a copy of the parameter, it just returns the address. Thus, you generally shouldn’t free this memory, as the calling function “owns” it.

The options listed above can be combined. For example, suppose we had a function that took a number of times to append a given string to a given array. We’d expect it to look something like:

``` c
PHP_FUNCTION(chant) {
  int str_len;
  long num;
  char *str;
  zval *arr;
 
  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "lsa", &num, &str, &str_len, &arr) == FAILURE) {
    return;
  }
 
  /* function body */
}
```

Note that you always pass in the address of the variable, not the variable itself.

&english - A list of addresses to use to store passed-in values. `zend_bool` is just a typedefed numeric type to represent booleans. Note that you must always use `long` for integers (not int). The long type is a different size on 32-bit and 64-bit machines (except on Windows!), so you’ll get weird segfaults if you use another numeric type on certain platforms.

# Zvals

**branch: types **

You may have noticed above that arrays, objects, and any type are all returned a zvals by `zend_parse_parameters`. This is because every PHP variable is, under the covers, a C struct called a zval. For example, if you say `$x = "foo"; $y = 123; $z = array();`, then `$x`, `$y`, and `$z` are all zvals.

If you want to be able to communicate information from C to PHP, it’s important to understand how to work with zvals. A zval is defined as:

``` c
struct _zval_struct {
    zvalue_value value;
    zend_uint refcount__gc;
    zend_uchar type;
    zend_uchar is_ref__gc;
};
```
The main components of this struct are:

`value` - The actual value of the variable. This is defined as a union of:

``` c
typedef union _zvalue_value {
    long lval;
    double dval;
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;
    zend_object_value obj;
} zvalue_value;
```

These field correspond to the following types:

```
+------+-------------------------------+
| lval | Longs and booleans            |
| dval | Doubles                       |
| str  | Strings                       |
| ht   | Arrays and associative arrays |
| obj  | Objects                       |
+------+-------------------------------+
```

`refcount__gc` -  A reference count for garbage collection. When you (or PHP) asks for a zval to be destroyed, the zval destructor decrements the `refcount` and checks if it is 0. If the `refcount` is greater than 0, it will just decremented the `refcount` and return. Once the `refcount` is 0, the zval will actually be freed.
`type` - The type of this zval. This tells PHP which union element to look for and what to do when it finds it. There are human-readable macros for each type:
``` c
#define IS_NULL	          0
#define IS_LONG	          1
#define IS_DOUBLE         2
#define IS_BOOL	          3
#define IS_ARRAY          4
#define IS_OBJECT         5
#define IS_STRING         6
#define IS_RESOURCE       7
#define IS_CONSTANT       8
#define IS_CONSTANT_ARRAY 9
``

This tutorial will only cover working with types 0-6. I’ve found that, with object-oriented PHP, the last three are less useful.

Resources are a way of binding C structs to PHP variables (e.g., passing a database connection around with the defunct mysql extension), but objects provide a nicer way of doing struct attachment. The Developer Zone tutorial goes into quite a lot of detail on resources, if you’re interested.

The field type tells PHP which union element to look at for the zval’s value. For example, if `zval_p->type == IS_STRING`, the value for the zval should be in the `zval_p->value.str` field.

This field also determines how PHP interprets the value field. For example, `lval` does double duty for longs and booleans. So, if you have `lval` set to 1 and `zval_p->type==IS_LONG`, it will be displayed as 1. If you have `zval_p->type==IS_BOOL`, it will be displayed as true.

``` c
// add function entries for each function you define
zend_function_entry rlyeh_functions[] = {
  PHP_FE(cthulhu, NULL)
  PHP_FE(makeBool, NULL)
  PHP_FE(makeLong, NULL)
  { NULL, NULL, NULL }
};

PHP_FUNCTION(makeBool) {
    Z_TYPE_P(return_value) = IS_BOOL;
    Z_LVAL_P(return_value) = 1;
}

PHP_FUNCTION(makeLong) {
    Z_TYPE_P(return_value) = IS_LONG;
    Z_LVAL_P(return_value) = 1;
}
```

// don't forget to add declarations for these functions to your header file, too!
`return_value` is passed into `PHP_FUNCTION`s and holds the value that is returned (it defaults to null).

If you compile this new code and run:

``` php
var_dump(makeBool());
var_dump(makeLong());
```

You’ll see something like:

``` php
bool(true)
int(1)
```

`is_ref__gc` - If this is a PHP reference.

# Accessing the contents of a zval

The internals of a zval are subject to change, so you should always PHP’s zval macros instead of touching the gooey innards (e.g., don’t actually set a value by putting `zval_p->value.lval` in your code).

As shown in the example above, you can safely manipulate said innards through these macros:

``` c
zval *zval_p;

long l        = Z_LVAL_P(zval_p)
zend_bool b   = Z_BVAL_P(zval_p)
double d      = Z_DVAL_P(zval_p)
char *str     = Z_STRVAL_P(zval_p)
int str_len   = Z_STRLEN_P(zval_p)
HashTable *ht = Z_ARRVAL_P(zval_p)

// objects are a bit complicated... suffice to know that these exist:
Z_OBJPROP_P(zval_p)
Z_OBJCE_P(zval_p)
Z_OBJVAL_P(zval_p)
Z_OBJ_HANDLE_P(zval_p)
Z_OBJ_HT_P(zval_p)
Z_OBJ_HANDLER_P(zval_p, h)
Z_OBJDEBUG_P(zval_p,is_tmp)
```

As you can see, you can extract each part of a zval’s value using a macro. The ones listed above work on zval pointers (`zval *`s). If you are using zval or `zval **`, there are analogous helpers with one fewer or one more P, respectively. For example:

``` c
long get_long(zval z) {
  return Z_LVAL(z);
}

// or 

long get_long(zval **zval_pp) {
  return Z_LVAL_PP(zval_pp);
}
```

# Creating zvals

Before using a zval, you must make sure that its `refcount`, type, and value are set correctly. For scalar types, you can set value and type using a single macro: `ZVAL_type`.

``` c
ZVAL_NULL(zval_p);
ZVAL_BOOL(zval_p, 0);
ZVAL_LONG(zval_p, 123);
ZVAL_DOUBLE(zval_p, 12.3);
```

For strings, it is a little trickier because you have to allocate space for the string or let PHP know that you’ve already allocated space for it.

Thus, `ZVAL_STRING` takes an argument that tells PHP whether or not to make a copy of the string for the zval. Basically, this should be 0 if you’ve already created a special instance of the string for this zval and 1 if you haven’t.

``` c
// "bar" is on the stack and zval_p is on the heap, so we 
// want to make a copy of "bar" on the heap
ZVAL_STRING(zval_p, "bar", 1);
// this means "copy" ------^

// copy "bar" to the heap
char *str = estrdup("bar");
ZVAL_STRING(zval_p, str, 0);
// "don't copy" ---------^
```

Which brings us to the next section, memory management.

# Memory Management

**branch: mm**

PHP uses its own memory pool and allocation/deallocation functions, which you should generally use instead of `malloc`, `free`, and friends.

PHP has similar functions to the standard C library, only everything is prefixed with an “e”:

``` c
void* emalloc(size_t size);
void* ecalloc(size_t size);
void* erealloc(size_t size);

void efree(void* ptr);

char* estrdup(char* str);
char* estrndup(char* str, int len);
```

If you are used to C programming where you check if memory was successfully allocated `(x = malloc(sizeof(x)); if (!x) return 0;)`, know that this is not strictly necessary in PHP. PHP’s memory management functions will exit PHP if you run out of memory, so if `emalloc` returns, it returned some memory.

Remember how you compiled PHP with `--enable-maintainer-zts` at the beginning? Well, here’s the payoff: it will let you know about any memory leaks it detects. For example, try adding a function to your extension:

``` c
// add to function_entry table and header file, too
PHP_FUNCTION(leak) {
    emalloc(20);
}
```

Now, if you recompile your extension and run `leak()`, you’ll see:

```
[Wed Aug 10 16:34:42 2011]  Script:  '-'
/Users/k/php-5.3.6/Zend/zend_builtin_functions.c(1360) :  Freeing 0x100AA75E0 (3 bytes), script=-
=== Total 1 memory leaks detected ===
```

This can make tracking down memory leaks much easier. (Getting friendly with valgrind is a good idea, too.)

# Creating and destroying zvals

Zvals can be created using `emalloc`, but I’d recommend generally using a different macro: `MAKE_STD_ZVAL`. This macro not only allocates a zval, but it also sets the refcount and isref fields, so you don’t have to worry about setting those yourself.

``` c
zval *zval_p;
MAKE_STD_ZVAL(zval_p);
```

If you need to destroy a zval, use `zval_ptr_dtor`, which takes a `zval **` (not a `zval *`).

``` c
zval *zval_p;
MAKE_STD_ZVAL(zval_p);

zval_ptr_dtor(&zval_p);
// back to square one
```

`zval_ptr_dtor` decrements the `refcount` by 1. If the `refcount` is still greater than 0, then `zval_ptr_dtor` will just return. If this makes the `refcount` 0, it also destroys the current zval. If this zval is a string, array, or object, PHP will take care of freeing the associated memory. Thus, you should generally not call free on a zval (as this will cause leaks: orphaned strings or objects with no zval pointing to them).

Also, you should always make sure that you have set the zval to the correct type before calling `zval_ptr_dtor`: if you call it on garbage, it can segfault if it tries to free, say, an string that was actually an invalid pointer.

# The persistence of memory

**branch: persistence**

Theoretically, all memory allocated with emalloc is freed after each request (I say theoretically because in my experience, it’s not so much freed as leaked). If you want something to hang around for longer than a single request, you’ll need to use persistent memory. Persistent memory hangs around for longer than one request (generally), up to the lifetime of the PHP process.

To allocate persistent memory, use `pe`-prefixed memory allocation functions, instead of `e`-prefixed.

``` c
void* pemalloc(size_t size, int persistent);
void* pecalloc(size_t size, int persistent);
void* perealloc(size_t size, int persistent);

void pefree(void* ptr, int persistent);

char* pestrdup(char* str, int persistent);
char* pestrndup(char* str, int len, int persistent);
```

The “persistent” option lets you choose whether you want to allocate persistent memory (1) or transitory memory (0, normal `e`-allocation behavior).

# Search and destroy: finding and cleaning up persistent memory

Suppose your extension allocates a persistent struct in one HTTP request. How do you find it during the next HTTP request?

There are three steps:

* You have to create a type for this memory.
* You have to link this type to a destructor, so that PHP knows how to clean up the memory.
* You have to insert your allocated memory into PHP’s persistent memory hash.

# Persistent Gods

To try out persistent memory, we want a struct that should persist for multiple requests. Great Old Ones are pretty darn persistent, so we’ll create an old_one struct in `php_rlyeh.h`:

``` c
typedef struct _old_one {
    char *name;
    int worshippers;
} old_one;
```

Now we need to creating a type for it. Near the beginning of `php_rlyheh.c`, add an int, named anything you want. This integer will hold the numeric type for Great Old Ones.

``` c
// traditionally these start with "le_", which stands 
// for "list entry"
int le_old_one;
```

Now we need to link the `le_old_one` type up to a destructor. We’ll do this when our module is first loaded, in the magical `PHP_MINIT_FUNCTION(rlyeh)` function:

``` c
// add MINIT to the module description:
zend_module_entry rlyeh_module_entry = {
  STANDARD_MODULE_HEADER,
  PHP_RLYEH_EXTNAME,
  rlyeh_functions,
  PHP_MINIT(rlyeh),
  NULL,
  NULL,
  NULL,
  NULL,
  PHP_RLYEH_VERSION,
  STANDARD_MODULE_PROPERTIES
};

// add this to php_rlyeh.h
PHP_MINIT_FUNCTION(rlyeh) {
    le_old_one = zend_register_list_destructors_ex(NULL, rlyeh_old_one_pefree, "Great Old One", module_number);
}
```

Also, add a line to `php_rlyeh.h`:

``` c
PHP_MINIT_FUNCTION(rlyeh);
```

`zend_register_list_destructors_ex` says, “make a new type for `le_old_one`. If you have to automatically free something of this type, call `rlyeh_old_one_pefree` on it.”

Persistent destructors always take a `zend_rsrc_list_entry`: this is the container PHP holds list entries (which is how we’re storing persistent memory). So, our destructor would look like:

``` c
void rlyeh_old_one_pefree(zend_rsrc_list_entry *rsrc TSRMLS_DC) {
    old_one *god = rsrc->ptr;

    // free the char* field, if set
    if (god->name) {
        pefree(god->name, 1);
    }

    pefree(god, 1);
}
```

Now we are ready to create some Great Old Ones!

Let’s make a new function: `getYig()`. If there’s already been an old_one created, it’ll return information about it, otherwise it’ll create a new one.

``` c
PHP_FUNCTION(getYig) {
    zend_rsrc_list_entry *le;
    char *key = "yig";
 
    if (zend_hash_find(&EG(persistent_list), key, strlen(key)+1, (void**)&le) == FAILURE) {
        // need to create a new god
        zend_rsrc_list_entry nle;
        old_one *yig;
 
        yig = (old_one*)pemalloc(sizeof(old_one), 1);
        yig->name = pestrdup("Yig", 1);
        yig->worshippers = 4;
 
        php_printf("creating a new god\n");
 
        nle.ptr = yig;
        nle.type = le_old_one;
        nle.refcount = 1;
 
        zend_hash_update(&EG(persistent_list), key, strlen(key)+1, (void*)&nle, sizeof(zend_rsrc_list_entry), NULL);
    }
    else {
        old_one *god = le->ptr;
 
        php_printf("fetched %s: %d worshippers\n", god->name, god->worshippers);
    }
}
```

Note that `zend_hash_update` and `zend_hash_find` take the key length+1. The PHP API is a bit inconsistent about this: the best way to figure out if a function takes length or length+1 is to look at the source or find an example of it being used in another extension.

If you have a web server set up, add your extension to the `php.ini` it’s using (warning: this is probably a different `php.ini` than the command-line client uses). Restart it and load a page that calls `getYig()` a couple of times. The first time you’ll see “creating”, the next times you’ll see “fetched…”.

In the code above, you may notice that we use hash functions (`zend_hash_find` and `zend_hash_add`) to manipulate the `EG(persistent_list)`. `EG(persistent_list)` is actually a `HashTable` that you can use to store persistent memory. However, the name reveals something that I find interesting about PHP internals: all `HashTables` (associative arrays) are lists, too (they keep the elements in order and you can access elements by index or key).

And speaking of hashes and lists…

# Arrays

## Creating arrays

**branch: array**

To create an array or associative array, use `array_init()`.

You can insert new elements to an associative array with one of these functions:

``` c
add_assoc_long(zval *zval_p, char *key, long n)
add_assoc_null(zval *zval_p, char *key)
add_assoc_bool(zval *zval_p, char *key, zend_bool b) 
add_assoc_double(zval *zval_p, char *key, double d) 
add_assoc_string(zval *zval_p, char *key, char *str, int duplicate) 
add_assoc_stringl(zval *zval_p, char *key, char *str, int length, int duplicate)
add_assoc_zval(zval *zval_p, char *key, zval *value) 
You can also “push” new elements to the array with related functions:

add_next_index_long(zval *zval_p, long n);
add_next_index_null(zval *zval_p);
add_next_index_bool(zval *zval_p, int b);
add_next_index_resource(zval *zval_p, int r);
add_next_index_double(zval *zval_p, double d);
add_next_index_string(zval *zval_p, const char *str, int duplicate);
add_next_index_stringl(zval *zval_, const char *str, uint length, int duplicate);
add_next_index_zval(zval *zval_p, zval *value);
```

Let’s use this to fill in the function we started in the `zend_parse_parameters` function:

``` c
PHP_FUNCTION(chant) {
  int str_len, i;
  long num;
  char *str;
  zval *arr;
 
  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "lsa", &num, &str, &str_len, &arr) == FAILURE) {
    return;
  }
 
  // sanity check
  if (num < 0 || num > 100) {
    return;
  } 
 
  for (i=0; i<num; i++) {
    add_next_index_stringl(arr, str, str_len, 1);
  }
}
```

If we run this function, we can see that appends strings to the array correctly.

``` php
<?php
 
$a = array();
chant(6, "derp", $a);
echo join($a, "\n")."\n";
 ```

This should output:

``` bash
derp
derp
derp
derp
derp
derp
```

## Accessing array elements

To find an element in an associative array, use one of the `zend_hash` functions.

``` c
PHP_FUNCTION(findMonster) {
  int monster_len;
  char *monster;
  zval *list, **desc;
 
  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sa", &monster, &monster_len, &list) == FAILURE) {
    return;
  }
 
  if (zend_hash_find(Z_ARRVAL_P(list), monster, monster_len+1, (void**)&desc) == FAILURE) {
    RETURN_NULL();
  }
 
  RETURN_STRINGL(Z_STRVAL_PP(desc), Z_STRLEN_PP(desc), 1);
}
```

Note that the fourth argument is the pointer to a pointer to a pointer. I was skeptical about that for a while, but there it is.

Also, we are using a couple of new macros for setting the return value. These `RETURN_type` macros just set the return_value we were manipulating directly earlier.

Now, if we run something like:

``` php
<?php
 
$a = array("Tsathoggua" => "The Toad God",
           "Yig" => "Father of Serpents",
           "Ythogtha" => "The Thing in the Pit");
 
var_dump(findMonster("Yig", $a));
```

We’ll get “Father of Serpents”.

There are also a couple other hash functions you’ll probably find useful for your code:

``` c
int zend_hash_find(const HashTable *ht, const char *arKey, uint nKeyLength, void **pData);
int zend_hash_add(const HashTable *ht, const char *arKey, uint nKeyLength, void *pData, int pDataSize, void **pDest);
int zend_hash_add(const HashTable *ht, const char *arKey, uint nKeyLength, void *pData, int pDataSize, void **pDest);
int zend_hash_num_elements(const HashTable *ht);
int zend_hash_exists(const HashTable *ht, const char *arKey, uint nKeyLength);
```

Note that these functions do not add references to array elements. Thus, you should not, for example, do zend_hash_find and then call zval_ptr_dtor on the element found or your array will be in a weird half-freed state and PHP will try to double-free the element when the array is properly destroyed. Therefore, if you want to use an array element outside of the context of the array, you should add a reference to it, first. (We avoid that in the situation above by returning duplicates of the element’s string value.)

## Iterating through arrays

You can iterate through an array, element by element, but it ain’t pretty. Here’s the standard for-loop you need:

``` c
HashTable *hindex = Z_ARRVAL_P(zval_p);
HashPosition pointer;
zval **data;
 
for(zend_hash_internal_pointer_reset_ex(hindex, &pointer);
    zend_hash_get_current_data_ex(hindex, (void**)&data, &pointer) == SUCCESS;
    zend_hash_move_forward_ex(hindex, &pointer)) {
 
  char *key;
  uint key_len, key_type;
  ulong index;
 
  key_type = zend_hash_get_current_key_ex(hindex, &key, &key_len, &index, 0, &pointer);
 
  switch (key_type) {
  case HASH_KEY_IS_STRING:
    // associative array keys
    php_printf("key: %s\n", key);
    break;
  case HASH_KEY_IS_LONG:
    // numeric indexes
    php_printf("index: %d\n", index);
    break;
  default:
    php_printf("error\n");
  }
  }
```
Now let’s never speak of it again.

