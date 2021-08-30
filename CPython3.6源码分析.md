
## 整体


## 对象源码分析

### PyObject/PyObjectType
#### 前言
```c
// Include/object.h
/* Object and type object interface */

/*
Objects are structures allocated on the heap.  Special rules apply to
the use of objects to ensure they are properly garbage-collected.
Objects are never allocated statically or on the stack; they must be
accessed through special macros and functions only.  (Type objects are
exceptions to the first rule; the standard types are represented by
statically initialized type objects, although work on type/class unification
for Python 2.2 made it possible to have heap-allocated type objects too).

An object has a 'reference count' that is increased or decreased when a
pointer to the object is copied or deleted; when the reference count
reaches zero there are no references to the object left and it can be
removed from the heap.

An object has a 'type' that determines what it represents and what kind
of data it contains.  An object's type is fixed when it is created.
Types themselves are represented as objects; an object contains a
pointer to the corresponding type object.  The type itself has a type
pointer pointing to the object representing the type 'type', which
contains a pointer to itself!).

Objects do not float around in memory; once allocated an object keeps
the same size and address.  Objects that must hold variable-size data
can contain pointers to variable-size parts of the object.  Not all
objects of the same type have the same size; but the size cannot change
after allocation.  (These restrictions are made so a reference to an
object can be simply a pointer -- moving an object would require
updating all the pointers, and changing an object's size would require
moving it if there was another object right next to it.)

Objects are always accessed through pointers of the type 'PyObject *'.
The type 'PyObject' is a structure that only contains the reference count
and the type pointer.  The actual memory allocated for an object
contains other data that can only be accessed after casting the pointer
to a pointer to a longer structure type.  This longer type must start
with the reference count and type fields; the macro PyObject_HEAD should be
used for this (to accommodate for future changes).  The implementation
of a particular object type can cast the object pointer to the proper
type and back.

A standard interface exists for objects that contain an array of items
whose size is determined when the object is allocated.
*/
```
开篇一段注释，从注释中能提取到很多要点：

    对象堆分配、从不栈分配
    垃圾回收之引用计数
    对象、类型对象、type
    容器对象可变依据：持有指针
    基石对象 PyObject 与类型转换


#### PyObject
```c
/* Nothing is actually declared to be a PyObject, but every pointer to a Python object can be cast to a PyObject*. This is inheritance built by hand.
Similarly every pointer to a variable-size Python object can, in addition, be cast to PyVarObject*.
*/
// ifdef Py_TRACE_REFS，定义双向链表存储所有堆上存活对象指针
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;           # 引用计数
    struct _typeobject *ob_type;    # 类型对象指针
} PyObject;

typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

PyObject 结构体中包含：

    指向类型对象 _typeobject 的指针 ob_type
    用于垃圾回收的引用计数 ob_refcnt

对于容器对象，PyObject_VAR_HEAD 用 ob_size 代表元素个数。


#### PyTypeObject

```c
// Doc\includes\typestruct.h
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                    or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* More standard operations (here for binary compatibility) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;

    /* Functions to access object as input/output buffer */
    PyBufferProcs *tp_as_buffer;

    /* Flags to define presence of optional/expanded features */
    unsigned long tp_flags;

    const char *tp_doc; /* Documentation string */

    /* call function for all accessible objects */
    traverseproc tp_traverse;

    /* delete references to contained objects */
    inquiry tp_clear;

    /* rich comparisons */
    richcmpfunc tp_richcompare;

    /* weak reference enabler */
    Py_ssize_t tp_weaklistoffset;

    /* Iterators */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Attribute descriptor and subclassing stuff */
    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free; /* Low-level free-memory routine */
    inquiry tp_is_gc; /* For PyObject_IS_GC */
    PyObject *tp_bases;
    PyObject *tp_mro; /* method resolution order */
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    /* Type attribute cache version tag. Added in version 2.6 */
    unsigned int tp_version_tag;

    destructor tp_finalize;

} PyTypeObject;
```

创建对象之前，必须知道申请的内存空间大小，而这些元信息就存储在对象的类型对象中。含有头域PyObject_VAR_HEAD，表明类型对象本身是 可变长对象。

结构体内存储大量信息，主要包括：

    常规信息：类型名、Doc、tp_itemsize、tp_basicsize等
    常规方法指针：tp_new、tp_init、tp_free等
    函数簇：PyNumberMethods、PySequenceMethods等

```c
// Include/object.h.301
typedef struct {
    lenfunc mp_length;
    binaryfunc mp_subscript;
    objobjargproc mp_ass_subscript;
} PyMappingMethods;
```
在函数簇 PyMappingMethods 中，定义了支持映射的对象应该支持的操作。反过来说，一旦定义了 其中的方法，那么该对象就支持该方法。
正因为 PyTypeObject 中同时定义了三种函数簇，所以才可以实现鸭子类型。





### PyLongObject
### PyBytesObject/PyUnicodeObject
### PyListObject
### PyDictObject
### PyCodeObject/PyFrameObject
### Python 一般字节码执行
### Python 控制字节码执行
### Python 异常控制
### Python 函数机制
### Python 类机制
### Python 自定义类
### Python 环境初始化
### Python 多线程机制
### Python 内存管理机制
### Python 垃圾回收
