Özel nesne depolama
===================

Önceki bölüm, basit iç sınıfların oluşturulması için zemin hazırladı. Burada açıklanan özelliklerin çoğu oldukça basit
olmalıdır, çünkü kullanıcı tarafındaki PHP'de olduğu gibi çalışırlar, sadece daha ayrıntılı olarak ifade edilirler. Öte
yandan, bu bölüm, kullanıcı tarafındaki sınıflarda mevcut olmayan alanlara girecektir: Özel nesne depolamasının
oluşturulması ve erişilmesi.

Nesneler nasıl oluşturulur?
---------------------------

İlk adım olarak, nesnenin PHP'de nasıl yaratıldığına bakalım. Bunun için ``object_and_properties_init`` makrosu ya da
basit kuzenlerinden biri kullanılır ::

    // SomeClass türünde bir nesne oluşturun ve özelliklerine properties_hashtable'dan verin
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_and_properties_init(obj, class_entry_of_SomeClass, properties_hashtable);

    // SomeClass türünde bir nesne oluşturun (varsayılan özelliklerle)
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_init_ex(obj, class_entry_of_SomeClass);
    // = object_and_properties_init(obj, class_entry_of_SomeClass, NULL)

    // Varsayılan bir nesne oluşturun (stdClass)
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_init(obj);
    // = object_init_ex(obj, NULL) = object_and_properties_init(obj, NULL, NULL)

Son durumda, yani bir ``stdClass`` nesnesini oluştururken muhtemelen daha sonra özellikler eklemek isteyeceksiniz. Bu
genellikle önceki bölümdeki ``zend_update_property`` işlevleri ile yapılmaz, bunun yerine ``add_property`` makroları
kullanılır::

    add_property_long(obj, "id", id);
    add_property_string(obj, "name", name, 1); // 1, dizenin kopyalanması gerektiği anlamına gelir
    add_property_bool(obj, "isAdmin", is_admin);
    // ayrıca _null(), _double(), _stringl(), _resource() and _zval()

Peki, bir nesne oluşturulduğunda gerçekte ne olur? Öğrenmek için ``_object_and_properties_init`` işlevine bakalım::

    ZEND_API int _object_and_properties_init(
        zval *arg, zend_class_entry *class_type, HashTable *properties ZEND_FILE_LINE_DC TSRMLS_DC
    ) {
        zend_object *object;

        if (class_type->ce_flags
            & (ZEND_ACC_INTERFACE|ZEND_ACC_IMPLICIT_ABSTRACT_CLASS|ZEND_ACC_EXPLICIT_ABSTRACT_CLASS)
        ) {
            char *what = (class_type->ce_flags & ZEND_ACC_INTERFACE)                 ? "interface"
                       : ((class_type->ce_flags & ZEND_ACC_TRAIT) == ZEND_ACC_TRAIT) ? "trait"
                       : "abstract class";
            zend_error(E_ERROR, "Cannot instantiate %s %s", what, class_type->name);
        }

        zend_update_class_constants(class_type TSRMLS_CC);

        Z_TYPE_P(arg) = IS_OBJECT;
        if (class_type->create_object == NULL) {
            Z_OBJVAL_P(arg) = zend_objects_new(&object, class_type TSRMLS_CC);
            if (properties) {
                object->properties = properties;
                object->properties_table = NULL;
            } else {
                object_properties_init(object, class_type);
            }
        } else {
            Z_OBJVAL_P(arg) = class_type->create_object(class_type TSRMLS_CC);
        }
        return SUCCESS;
    }

İşlev temel olarak üç şey yapar: İlk önce sınıfın gerçek anlamda başlatılabileceğini doğrular, ardından sınıf
sabitlerini çözer (bu yalnızca ilk başlatmada yapılır ve ayrıntıları burada önemli değildir). Bundan sonra önemli kısım
gelir: İşlev, sınıfın ``create_object`` işleyicisine sahip olup olmadığını kontrol eder. Eğer bir tanesine sahipse,
varsayılan ``zend_objects_new`` uygulaması yoksa, kullanılır (ve ayrıca özellikler başlatılır).

Sonra, ``zend_objects_new`` aşağıdakileri yapar::

    ZEND_API zend_object_value zend_objects_new(
        zend_object **object, zend_class_entry *class_type TSRMLS_DC
    ) {
        zend_object_value retval;

        *object = emalloc(sizeof(zend_object));
        (*object)->ce = class_type;
        (*object)->properties = NULL;
        (*object)->properties_table = NULL;
        (*object)->guards = NULL;
        retval.handle = zend_objects_store_put(*object,
            (zend_objects_store_dtor_t) zend_objects_destroy_object,
            (zend_objects_free_object_storage_t) zend_objects_free_object_storage,
            NULL TSRMLS_CC
        );
        retval.handlers = &std_object_handlers;
        return retval;
    }

Yukarıdaki kod üç ilginç şey içeriyor. Öncelikle tanımlandığı gibi, ``zend_object`` yapısı::

    typedef struct _zend_object {
        zend_class_entry *ce;
        HashTable *properties;
        zval **properties_table;
        HashTable *guards; /* protects from __get/__set ... recursion */
    } zend_object;

Bu "standart" nesne yapısıdır. Oluşturma için kullanılan sınıf girdisini, bir hashtable özelliğini, bir "tablo"
özelliğini ve özyinelemeyi koruma için bir karma tablosunu içerir. ``properties`` ve ``properties_table`` arasındaki
farkın tam olarak ne olduğu, bu bölümün sonraki bölümlerinde ele alınacaktır; bu noktada, yalnızca ikincisinin sınıfta
ilan edilen özellikler için kullanıldığını ve ilk olarak ilan edilmeyen özellikler için kullanıldığını bilmelisiniz.
``guards`` mekanizmasının nasıl işlediği de daha sonra ele alınacaktır.

``zend_objects_new`` işlevi yukarıda belirtilen standart nesne yapısını tahsis eder ve başlatır. Daha sonra nesne
verisini nesne deposuna koymak için ``zend_objects_store_put`` çağırır. Nesne deposu dinamik olarak yeniden
boyutlandırılmış bir ``zend_object_store_bucket``\s dizisinden başka bir şey değildir::

    typedef struct _zend_object_store_bucket {
        zend_bool destructor_called;
        zend_bool valid;
        union _store_bucket {
            struct _store_object {
                void *object;
                zend_objects_store_dtor_t dtor;
                zend_objects_free_object_storage_t free_storage;
                zend_objects_store_clone_t clone;
                const zend_object_handlers *handlers;
                zend_uint refcount;
                gc_root_buffer *buffered;
            } obj;
            struct {
                int next;
            } free_list;
        } bucket;
    } zend_object_store_bucket;

Buradaki ana bölüm, ``void *object`` nesnesinin üyesinde saklanan nesneyi içeren ``_store_object`` yapısı ve ardından
yıkım, serbest bırakma ve klonlama işleyicileridir. Bu yapıda bazı ek şeyler de var, örneğin “refcount”
özelliğine sahip, çünkü nesne deposundaki bir nesneye aynı anda birkaç zval'dan başvuruda bulunabiliyor ve PHP'nin kaç
tane izini tutması gerektiği oradaki referanslar tarafından sonradan serbest bırakılabilir. Ek olarak, ``handlers``
işleyicileri nesnesi de saklanır (bu, imha için gereklidir) ve bir GC kök tamponu (PHP'ler döngü toplayıcısının nasıl
çalıştığı daha sonraki bir bölümde ele alınacaktır).

``zend_objects_new`` işlevine geri dönersek, yaptığı en son şey, işleyicileri varsayılan ``std_object_handlers``'a
ayarlamaktır.

create_object öğesini geçersiz kılma
------------------------------------

Özel nesne depolamayı kullanmak istediğinizde, temel olarak yukarıdaki üç adımı tekrarlayacaksınız: İlk önce, standart
nesneyi bir altyapı olarak içerecek olan nesneyi ayırır ve başlatırsınız. Sonra onu birkaç işleyiciyle birlikte nesne
deposuna koyarsınız. Ve son olarak, nesne işleyicileri yapınızı atarsınız.

Bunu yapmak için `create_object` sınıf işleyicisini geçersiz kılmanız gerekir. İşte bunun nasıl göründüğünü gösteren
basit bir örnek (satır içi açıklamalarla)::

    zend_class_entry *test_ce;

    /* Nesnelerimiz için kullanılacak nesne işleyicileri depolamak için (gerçek bir global) değişkene ihtiyacımız var.
     * Nesne işleyicileri MINIT'te başlatıldı. */
    static zend_object_handlers test_object_handlers;

    /* Özel nesne yapımız. İlk üye olarak bir `zend_object` değeri (bir işaretçi değil!) içermesi gerekir, 
     * ardından istediği ilave özellikleri takip edin. */
    typedef struct _test_object {
        zend_object std;
        long additional_property;
    } test_object;

    /* Bu, nesne serbest bırakıldığında çağrılacak olan işleyicidir.
     * Bu işleyici std nesnesini imha etmelidir (bu özellik özellikleri vb. Serbest bırakacaktır)
     * ve ayrıca nesne yapısını kendisi serbest bırakmalıdır.
     * (Ve tahsis edilen başka kaynaklar varsa, bunların da burada serbest bırakılması gerekir.) */
    static void test_free_object_storage_handler(test_object *intern TSRMLS_DC)
    {
        zend_object_std_dtor(&intern->std TSRMLS_CC);
        efree(intern);
    }

    /* Bu nesne oluşturmak için kullanılan işleyicidir. Sınıf girişini alır (bunu genişleten sınıflar
     * için de kullanılır, bu nedenle sınıf girişinin iletilmesi gerekir) ve bir nesne değeri (nesne 
     * deposuna bir tutamaç ve nesne işleyicisine bir işaretçi yapısı döndürür). */
    zend_object_value test_create_object_handler(zend_class_entry *class_type TSRMLS_DC)
    {
        zend_object_value retval;

        /* Dahili nesne yapısını atayın ve sıfırlayın. Kural olarak, iç yapıyı tutan değişken genellikle 
         * `intern` olarak adlandırılır. */
        test_object *intern = emalloc(sizeof(test_object));
        memset(intern, 0, sizeof(test_object));

        /* Temel std zend_object başlatılması gerekiyor.  */
        zend_object_std_init(&intern->std, class_type TSRMLS_CC);

        /* Özellikleri kendiniz kullanmasanız bile, object_properties_init () öğesini
         * çağırmalısınız, çünkü extending sınıfları özellikleri kullanabilir. (Genelde
         * yapacağınız işlerin çoğu, genişleyen sınıfları kırmamak adınadır). */
        object_properties_init(&intern->std, class_type);

        /* Dahili nesneyi, varsayılan dtor işleyicisi ve özel serbest işleyicimiz ile,
         * nesne deposuna yerleştirin. Son NULL parametresi şimdilik boş bırakılan klon 
         * işleyicisidir. */
        retval.handle = zend_objects_store_put(
            intern,
            (zend_objects_store_dtor_t) zend_objects_destroy_object,
            (zend_objects_free_object_storage_t) test_free_object_storage_handler,
            NULL TSRMLS_CC
        );

        /* Özelleştirilmiş nesne işleyicileri atama */
        retval.handlers = &test_object_handlers;

        return retval;
    }

    /* Şimdilik fonksiyon yok */
    const zend_function_entry test_functions[] = {
        PHP_FE_END
    };

    PHP_MINIT_FUNCTION(test2)
    {
        /* Her zamanki sınıf kaydı... */
        zend_class_entry tmp_ce;
        INIT_CLASS_ENTRY(tmp_ce, "Test", test_functions);
        test_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);

        /* Sınıf girişine nesne oluşturma işleyicisini atayın */
        test_ce->create_object = test_create_object_handler;

        /* Özel nesne işleyicileri varsayılan nesne işleyicileri için ilklendir. Daha sonra 
         * normal olarak bireysel işleyicileri geçersiz kılarsınız, ancak şimdilik bunları 
         * varsayılanlara bırakalım. */
        memcpy(&test_object_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));

        return SUCCESS;
    }

Yukarıdaki kod henüz pek kullanışlı değil, ancak hemen hemen tüm iç PHP sınıflarının temel yapısını gösterir.

Object store handlers
---------------------

As already mentioned there are three object storage handlers: One for destruction, one for freeing and one for cloning.

What is a bit confusing at first is that there is both a dtor handler and a free handler, which sounds like they do
about the same thing. The reason is that PHP has a two-phase object destruction system, where first the destructor is
called and then the object is freed. Both phases can happen separately from each other.

In particular this happens with all objects which are still alive when the script ends. For them PHP will first call all
dtor handlers (right after calling any registered shutdown functions), but will only free the objects at a later point
in time, as part of the executor shutdown. This separation of destruction and freeing is necessary to ensure that no
destructors are run during the shutdown sequence, otherwise you could get into situations where userland code is
executed in a half-shutdown environment. Without this separation any ``zval_ptr_dtor`` call during shutdown could blow
up.

Another peculiarity of dtor handlers is that they *aren't* necessarily called. E.g. if a destructor calls ``die`` the
remaining destructors are skipped.

So basically the difference between the two handlers is that dtor can run userland code, but isn't necessarily called,
free on the other hand is always called, but mustn't execute any PHP code. That's why in most cases you will only
specify a custom free handler and use ``zend_objects_destroy_object`` as the dtor handler, which provides the default
behavior of calling ``__destruct`` (if it exists). Once again, even if you don't use ``__destruct`` yourself you should
still specify this handler, otherwise inheriting classes won't be able to use it either.

Now only the clone handler is left. Here the semantics should be straightforward, but the use is a bit more tricky.
This is how such a clone handler might look like::

    static void test_clone_object_storage_handler(
        test_object *object, test_object **object_clone_target TSRMLS_DC
    ) {
        /* Create a new object */
        test_object *object_clone = emalloc(sizeof(test_object));
        zend_object_std_init(&object_clone->std, object->std.ce TSRMLS_CC);
        object_properties_init(&object_clone->std, object->std.ce);

        /* Do any additional cloning stuff here */
        object_clone->additional_property = object->additional_property;

        /* Return the cloned object */
        *object_clone_target = object_clone;
    }

The clone handler is then passed as the last argument to ``zend_objects_store_put``::

    retval.handle = zend_objects_store_put(
        intern,
        (zend_objects_store_dtor_t) zend_objects_destroy_object,
        (zend_objects_free_object_storage_t) test_free_object_storage_handler,
        (zend_objects_store_clone_t) test_clone_object_storage_handler
        TSRMLS_CC
    );

But this is not yet enough to make the clone handler work: By default the object storage clone handler is simply
ignored. To make it work you have to replace the default clone handler in the object handlers structure with
``zend_objects_store_clone_obj``::

    memcpy(&test_object_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
    test_object_handler.clone_obj = zend_objects_store_clone_obj;

But overwriting the standard clone handler (``zend_objects_clone_obj``) comes with its own set of problems: Now
properties (as in real properties, not the ones in the custom object storage) won't be copied and also the ``__clone``
method won't be called. That's why most internal classes instead directly specify their own object handler for cloning,
rather than going the extra round through the object storage clone handler. This approach comes with a bit more
boilerplate. For example, this is how the default clone handler looks like::

    ZEND_API zend_object_value zend_objects_clone_obj(zval *zobject TSRMLS_DC)
    {
        zend_object_value new_obj_val;
        zend_object *old_object;
        zend_object *new_object;
        zend_object_handle handle = Z_OBJ_HANDLE_P(zobject);

        /* assume that create isn't overwritten, so when clone depends on the
         * overwritten one then it must itself be overwritten */
        old_object = zend_objects_get_address(zobject TSRMLS_CC);
        new_obj_val = zend_objects_new(&new_object, old_object->ce TSRMLS_CC);

        zend_objects_clone_members(new_object, new_obj_val, old_object, handle TSRMLS_CC);

        return new_obj_val;
    }

This function first fetches the ``zend_object*`` structure from the object store using ``zend_objects_get_address``,
then creates a new object with the same class entry (using ``zend_objects_new``) and then calls
``zend_objects_clone_members``, which will (as the name says) clone the properties, but will also call the ``__clone``
method if it exists.

A custom object cloning handler looks similar, with the main difference being that instead of calling
``zend_objects_new`` we'll rather call our ``create_object`` handler::

    static zend_object_value test_clone_handler(zval *object TSRMLS_DC)
    {
        /* Get the internal structure of the old object */
        test_object *old_object = zend_object_store_get_object(object TSRMLS_CC);

        /* Create a new object with the same class entry. This will only give us back the
         * zend_object_value, but not the actual internal structure of the new object. */
        zend_object_value new_object_val = test_create_object_handler(Z_OBJCE_P(object) TSRMLS_CC);

        /* To get the internal structure we need to fetch it from the object store using the
         * handle we got from the create_object handler. */
        test_object *new_object = zend_object_store_get_object_by_handle(
            new_object_val.handle TSRMLS_CC
        );

        /* Clone properties and call __clone */
        zend_objects_clone_members(
            &new_object->std, new_object_val,
            &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC
        );

        /* Here comes the actual custom cloning code */
        new_object->additional_property = old_object->additional_property;

        return new_object_val;
    }

    /* ... */
    test_object_handler.clone_obj = test_clone_handler;

Interacting with the object store
---------------------------------

In the above code samples you have already seen several functions for interacting with the object store. The first one
was ``zend_objects_store_put``, which is used for inserting objects into the store. Also three functions for getting
objects back from the store were mentioned:

``zend_object_store_get_object_by_handle()``, as the name already says, gets an object from the store given its handle.
This function is used when you have an object handle, but don't have the associated zval (like in the clone handler).
In most other cases on the other hand you'll use the ``zend_object_store_get_object()`` function which accepts a zval
and will extract the handle from it.

The third getter function that was used is ``zend_objects_get_address()``, which does the exact same thing as
``zend_object_store_get_object()``, but returns the result as a ``zend_object*`` rather than a ``void*``. As such this
function is pretty useless because C allows implicit casts from ``void*`` to other pointer types.

The most important of these functions is ``zend_object_store_get_object()``. You will be using it a lot. Pretty much
all methods will look similar to this::

    PHP_METHOD(Test, foo)
    {
        zval *object;
        test_object *intern;

        if (zend_parse_parameters_none() == FAILURE) {
            return;
        }

        object = getThis();
        intern = zend_object_store_get_object(object TSRMLS_CC);

        /* Do some stuff here, like returning an internal property: */
        RETURN_LONG(intern->additional_property);
    }

There are some more functions provided by the object store, e.g. for managing the object refcount, but those are rarely
used directly, so they aren't covered here.