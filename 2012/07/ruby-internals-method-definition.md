# Ruby Internal: Method Definiton

In this post I'm going to walk through what happens when you define a
method in a class or Object in Ruby (1.9.3 in this case).

When you're writing a C extension, Ruby provides several functions that
you can use. The most common are:

```c
void rb_define_method(VALUE klass, const char *name,
            VALUE (*func)(), int argc)

void rb_define_singleton_method(VALUE object, const char *name,
              VALUE (*func)(), int argc)
```

If you were to define the `size` method on string, you would have
something like:

```c
rb_define_method(rb_cString, "size", rb_str_length, 0);
```

Where `rb_str_length` is a pointer to the function you want to be
called.

But what does `rb_define_method` really do? To answer that question we
need to look inside `class.c` for its function defintion.

### rb_define_method

This is how `rb_define_method` looks like:

```c
void
rb_define_method(VALUE klass, const char *name, VALUE (*func)(ANYARGS),
int argc)
{
    rb_add_method_cfunc(klass, rb_intern(name), func, argc, NOEX_PUBLIC);
}
```

The `rb_add_method_cfunc` is a general function for adding methods to a
class. The `NOEX_PUBLIC` constant tells Ruby that the method is public.
Let's see how this function works:

```c
void
rb_add_method_cfunc(VALUE klass, ID mid, VALUE (*func)(ANYARGS), int argc, rb_method_flag_t noex)
{
  if (func != rb_f_notimplement) {
    rb_method_cfunc_t opt;
    opt.func = func;
    opt.argc = argc;
    rb_add_method(klass, mid, VM_METHOD_TYPE_CFUNC, &opt, noex);
  }
  else {
    rb_define_notimplement_method_id(klass, mid, noex);
  }
}
```

`rb_f_notimplement` checks wherever the method should not be
implemented (ie: if it's equal to
[NotImplementedError](http://www.ruby-doc.org/core-1.9.3/NotImplementedError.html)).
In our case though, the comparision will return false so we move on and
create a `rb_method_cfunc_t` struct which contains the function and
arguments of the method we want to define. We then pass this object to
`rb_add_method`, which we'll look at next.

### rb_add_method

It turns out that you can define lots of different methods, and the job
of `rb_add_method` is to act as a dispatcher based on the method type.

```c
d_entry_t *
rb_add_method(VALUE klass, ID mid, rb_method_type_t type, void *opts, rb_method_flag_t noex)
{
  rb_thread_t *th;
  rb_control_frame_t *cfp;
  int line;
  rb_method_entry_t *me = rb_method_entry_make(klass, mid, type, 0, noex);
  rb_method_definition_t *def = ALLOC(rb_method_definition_t);
  me->def = def;
  def->type = type;
  def->original_id = mid;
  def->alias_count = 0;
  switch (type) {
    case VM_METHOD_TYPE_ISEQ:
  def->body.iseq = (rb_iseq_t *)opts;
  break;
      case VM_METHOD_TYPE_CFUNC:
  def->body.cfunc = *(rb_method_cfunc_t *)opts;
  break;
      case VM_METHOD_TYPE_ATTRSET:
      case VM_METHOD_TYPE_IVAR:
  def->body.attr.id = (ID)opts;
  def->body.attr.location = Qfalse;
  th = GET_THREAD();
  cfp = rb_vm_get_ruby_level_next_cfp(th, th->cfp);
  if (cfp && (line = rb_vm_get_sourceline(cfp))) {
      VALUE location = rb_ary_new3(2, cfp->iseq->location.path, INT2FIX(line));
      def->body.attr.location = rb_ary_freeze(location);
  }
  break;
      case VM_METHOD_TYPE_BMETHOD:
  def->body.proc = (VALUE)opts;
  break;
      case VM_METHOD_TYPE_NOTIMPLEMENTED:
  def->body.cfunc.func = rb_f_notimplement;
  def->body.cfunc.argc = -1;
  break;
      case VM_METHOD_TYPE_OPTIMIZED:
  def->body.optimize_type = (enum method_optimized_type)opts;
  break;
      case VM_METHOD_TYPE_ZSUPER:
      case VM_METHOD_TYPE_UNDEF:
  break;
      default:
  rb_bug("rb_add_method: unsupported method type (%d)\n", type);
    }
    if (type != VM_METHOD_TYPE_UNDEF) {
  method_added(klass, mid);
    }
    return me;
}
```

The first interesting part is how the eventually returned value is created:

```c
rb_method_entry_t *me = rb_method_entry_make(klass, mid, type, 0, noex);
```

#### rb_method_entry_make

`rb_method_entry_make` is a long method, but it's worth posting its
content as you can get many insights into what happens behind the scene
in C. This is, for example, the place where Ruby checks if you have
enough permissions to create methods, or if the class is frozen.

```c
tatic rb_method_entry_t *
rb_method_entry_make(VALUE klass, ID mid, rb_method_type_t type,
         rb_method_definition_t *def, rb_method_flag_t noex)
{
  rb_method_entry_t *me;
#if NOEX_NOREDEF
  VALUE rklass;
#endif
  st_table *mtbl;
  st_data_t data;

  if (NIL_P(klass)) {
    klass = rb_cObject;
  }
  if (rb_safe_level() >= 4 && (klass == rb_cObject || !OBJ_UNTRUSTED(klass))) {
    rb_raise(rb_eSecurityError, "Insecure: can't define method");
  }
  if (!FL_TEST(klass, FL_SINGLETON) &&
    type != VM_METHOD_TYPE_NOTIMPLEMENTED &&
    type != VM_METHOD_TYPE_ZSUPER &&
    (mid == rb_intern("initialize") || mid == rb_intern("initialize_copy"))) {
    noex = NOEX_PRIVATE | noex;
  }
  else if (FL_TEST(klass, FL_SINGLETON) &&
     type == VM_METHOD_TYPE_CFUNC &&
     mid == rb_intern("allocate")) {
    rb_warn("defining %s.allocate is deprecated; use rb_define_alloc_func()",
      rb_class2name(rb_ivar_get(klass, attached)));
    mid = ID_ALLOCATOR;
  }

  rb_check_frozen(klass);
#if NOEX_NOREDEF
  rklass = klass;
#endif
  klass = RCLASS_ORIGIN(klass);
  mtbl = RCLASS_M_TBL(klass);

  /* check re-definition */
  if (st_lookup(mtbl, mid, &data)) {
    rb_method_entry_t *old_me = (rb_method_entry_t *)data;
    rb_method_definition_t *old_def = old_me->def;

    if (rb_method_definition_eq(old_def, def)) return old_me;
#if NOEX_NOREDEF
    if (old_me->flag & NOEX_NOREDEF) {
      rb_raise(rb_eTypeError, "cannot redefine %"PRIsVALUE"#%"PRIsVALUE,
         rb_class_name(rklass), rb_id2str(mid));
    }
#endif
  rb_vm_check_redefinition_opt_method(old_me, klass);

  if (RTEST(ruby_verbose) &&
      type != VM_METHOD_TYPE_UNDEF &&
      old_def->alias_count == 0 &&
      old_def->type != VM_METHOD_TYPE_UNDEF &&
      old_def->type != VM_METHOD_TYPE_ZSUPER) {
      rb_iseq_t *iseq = 0;

      rb_warning("method redefined; discarding old %s", rb_id2name(mid));
      switch (old_def->type) {
        case VM_METHOD_TYPE_ISEQ:
    iseq = old_def->body.iseq;
    break;
        case VM_METHOD_TYPE_BMETHOD:
    iseq = rb_proc_get_iseq(old_def->body.proc, 0);
    break;
        default:
    break;
      }
      if (iseq && !NIL_P(iseq->location.path)) {
    int line = iseq->line_info_table ? rb_iseq_first_lineno(iseq) : 0;
    rb_compile_warning(RSTRING_PTR(iseq->location.path), line,
           "previous definition of %s was here",
           rb_id2name(old_def->original_id));
      }
  }

  rb_unlink_method_entry(old_me);
    }

    me = ALLOC(rb_method_entry_t);

    rb_clear_cache_by_id(mid);

    me->flag = NOEX_WITH_SAFE(noex);
    me->mark = 0;
    me->called_id = mid;
    me->klass = klass;
    me->def = def;
    if (def) def->alias_count++;

    /* check mid */
    if (klass == rb_cObject && mid == idInitialize) {
  rb_warn("redefining Object#initialize may cause infinite loop");
    }
    /* check mid */
    if (mid == object_id || mid == id__send__) {
  if (type == VM_METHOD_TYPE_ISEQ) {
      rb_warn("redefining `%s' may cause serious problems", rb_id2name(mid));
  }
    }

    st_insert(mtbl, mid, (st_data_t) me);

    return me;
}
```

The rest of this function deals with actually inserting the method into a
methods table and overwriting any existing defintion. The resulting
method definition is then passed back to `rb_add_method`.

#### Back to rb_add_method

Now that we know what `*me` is, let's look at how it's used. Here's a
stripped down version of the method, focusing only on the most
interesting bits.

```c
rb_method_entry_t *
rb_add_method(VALUE klass, ID mid, rb_method_type_t type, void *opts, rb_method_flag_t noex)
{
  /* other code here */
  rb_method_entry_t *me = rb_method_entry_make(klass, mid, type, 0, noex);
  rb_method_definition_t *def = ALLOC(rb_method_definition_t);
  me->def = def;
  def->type = type;
  def->original_id = mid;
  def->alias_count = 0;
  switch (type) {
    /* other code here */
    case VM_METHOD_TYPE_CFUNC:
      def->body.cfunc = *(rb_method_cfunc_t *)opts;
      break;
    /* other code here */

  return me;
}
```

As you can see, the method itself is responsible for keeping track of
the origin name (mid), the number of aliases, and the type. In the
switch statement we add a pointer to the c function we want to use, and
then we finally return the method itself.

### Conclusion

There is a huge deal of complexity going on when definining a method in
Ruby. I plan to dig into how methods are stored and retrevied in a
future article, so stay tuned!
