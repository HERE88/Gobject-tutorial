# Signals

## Signals

Signals provide a means of communication between objects.
Signals are emitted when something happens or completes.

The steps to program a signal is shown below. 

1. Register a signal.
A signal belongs to an object, so the registration is done in the class initialization function of the object.
2. Write a signal handler.
A signal handler is a function that is invoked when the signal is emitted.
3. Connect the signal and handler.
Signals are connected to handlers with `g_connect_signal` or its family functions.
4. Emit the signal.

Step one and Four are done in the object to which the signal belongs.
Step three is usually done outside the object.

The process of signals is complicated and it takes long to explain all the features.
The contents of this section is limited to the minimum things to write a simple signal and not necessarily accurate.
If you need an accurate information, refer to GObject API reference.
There are three parts which describe signals.

- [I Concepts, signals](https://developer.gnome.org/gobject/stable/signal.html)
- [II API reference, Signals](https://developer.gnome.org/gobject/stable/gobject-Signals.html#)
- [IV Tutorial, How to create and use signals](https://developer.gnome.org/gobject/stable/howto-signals.html)

## Signal registration

An example in this section is a signal emitted when division-by-zero happens.
First, we need to determine the name of the signal.
Signal name consists of letters, digits, dash (`-`) and underscore (`_`).
The first character of the name must be a letter.
So, a string "div-by-zero"is appropriate for the signal name.

There are four functions to register a signal.
We will use `g_signal_new` for "div-by-zero" signal.

~~~C
guint
g_signal_new (const gchar *signal_name,
              GType itype,
              GSignalFlags signal_flags,
              guint class_offset,
              GSignalAccumulator accumulator,
              gpointer accu_data,
              GSignalCMarshaller c_marshaller,
              GType return_type,
              guint n_params,
              ...);
~~~

It needs a lot to explain each parameter.
At present I just show you `g_signal_new` function call extracted from `tdouble.c`.

~~~C
t_double_signal =
g_signal_newv ("div-by-zero",
               G_TYPE_FROM_CLASS (class),
               G_SIGNAL_RUN_LAST | G_SIGNAL_NO_RECURSE | G_SIGNAL_NO_HOOKS,
               NULL /* closure */,
               NULL /* accumulator */,
               NULL /* accumulator data */,
               NULL /* C marshaller */,
               G_TYPE_NONE /* return_type */,
               0     /* n_params */,
               NULL  /* param_types */);
~~~

- `t_double_signal` is a static guint variable.
Guint type is the same as unsigned int.
It is set the signal id.
- The second parameter is the type (GType) of the object the signal belongs to.
`G_TYPE_FROM_CLASS (class)` returns the type corresponds to the class (`class` is a pointer to the class of the object).
- The third parameter is a signal flag.
Lots of pages are necessary to explain this flag.
So, I want leave them out now.
The argument above can be used in many cases.
- The return type is G_TYPE_NONE which means no value is returned by the signal handler.
- `n_params` is a number of parameters.
This signal doesn't have parameters, so it is zero.
And no parameter type is necessary, so the last argument is NULL.

This function call is in the class initialization function.

## Signal handler

Signal handler is a function that is called when the signal is emitted.
The handler has two parameters.

- The instance to which the signal belongs
- A pointer to a user data which is given in the signal connection.

The example signal doesn't need user data.
You can leave out the second parameter.

~~~C
void dev_by_zero_cb (TDouble *d, gpointer user_data) { ... ... ...};
~~~

Or

~~~C
void dev_by_zero_cb (TDouble *d) { ... ... ...};
~~~

If a signal has parameters, the parameters are between the instance and user data.
For example, the handler of "response" signal of GtkDialog is:

~~~C
void user_function (GtkDialog *dialog, int response_id, gpointer user_data);
~~~

`resoponse_id` is the parameter of the signal.
"response" signal is emitted when a user clicks on a button like "OK" or "Cancel".
The argument of `response_id` is a value shows what button has been clicked.

The handler of "div-by-zero" signal just shows an error message.

~~~C
static void
div_by_zero_cb (TDouble *d, gpointer user_data) {
  g_print ("\nError: division by zero.\n\n");
}
~~~

## Signal connection

A signal and a handler are connected with the function `g_signal_connect`.

~~~C
g_signal_connect (d1, "div-by-zero", G_CALLBACK (div_by_zero_cb), NULL);
~~~

- `d1` is an instance the signal belongs.
- The second argument is the signal name.
- The third argument is the signal handler.
It must be casted by `G_CALLBACK`.
- The last argument is an user data.
The signal doesn't need a user data, so NULL is given.

## Signal emission

Signals are emitted on the object.
The following is a part of `tdouble.c`.

~~~C
TDouble *
t_double_div (TDouble *self, TDouble *other) {
... ... ...
  if ((! t_double_get_value (other, &value)))
    return NULL;
  else if (value == 0) {
    g_signal_emit (self, t_double_signal, 0);
    return NULL;
  }
  return t_double_new (self->value / value);
}
~~~

If the divisor is zero, the signal is emitted.
`g_sigmal_emit` has three parameters.

- The first parameter is the instance that emits the signal.
- The second parameter is the signal id.
Signal id has been set with `g_signal_new` function.
- The third parameter is detail.
"dev-by-zero" signal doesn't have an detail, so the argument is zero.
Detail isn't explained in this section.
If you want to know details, refer to GObject API referemce.

If a signal has parameters, they are fourth and subsequent arguments.

## Example code

A sample program is in [src/tdouble3](tdouble3).

tdouble.c

@@@include
tdouble3/tdouble.c
@@@

main.c

@@@include
tdouble3/main.c
@@@

To compile, change your current directory to src/tdouble3 and type as follows.

~~~
$ meson _build
$ ninja -C _build
~~~

Then, Executable file `tdouble` is created in `_build` directory.
Execute it.

@@@shell
cd tdouble3; _build/tdouble
@@@

## Default signal handler

You may have thought that it was strange that the error message was set in `main.c`.
Indeed, the error happens in `tdouble.c` so the message should been emitted in `tdouble.c` itself.
GObject system has a default signal handler that is set in the object itself.
A default signal handler is also called "default handler" or "object method handler".

You can set a default handler with `g_signal_new_class_handler`.

~~~c
guint
g_signal_new_class_handler (const gchar *signal_name,
                            GType itype,
                            GSignalFlags signal_flags,
                            GCallback class_handler, /*default signal handler */
                            GSignalAccumulator accumulator,
                            gpointer accu_data,
                            GSignalCMarshaller c_marshaller,
                            GType return_type,
                            guint n_params,
                            ...);
~~~

The difference from `g_signal_new` is the fourth parameter.
`g_signal_new` sets a default handler with the offset of the function pointer in the class structure.
If an object is derivable, it has its own class area, so you can set a default handler with `g_signal_new`.
But a final type object doesn't have its own class area, so it's impossible to set a default handler with `g_signal_new`.
That's the reason why we use `g_signal_new_class_handler`.

`tdouble.c` is changed.
`div_by_zero_default_cb` function is added and `g_signal_new_class_handler` replaces `g_signal_new`.

@@@include
tdouble4/tdouble.c div_by_zero_default_cb t_double_class_init
@@@

`g_signal_connect` and `div_by_zero_cb` are removed from `main.c`.

Compile and execute it.

@@@shell
cd tdouble4; _build/tdouble
@@@

If you want to connect your handler (user-provided handler) to the signal, you can still use `g_signal_connect`.
Add the following in `main.c`.

~~~C
static void
div_by_zero_cb (TDouble *d, gpointer user_data) {
  g_print ("\nError happens in main.c.\n");
}

int
main (int argc, char **argv) {
... ... ...
  g_signal_connect (d1, "div-by-zero", G_CALLBACK (div_by_zero_cb), NULL);
... ... ...
}
~~~

Then, both the user-provided handler and default handler are called when the signal is emitted.
Compile and execute it, then the following is shown on your display.

~~~
10.000000 + 20.000000 = 30.000000
10.000000 - 20.000000 = -10.000000
10.000000 * 20.000000 = 200.000000
10.000000 / 20.000000 = 0.500000

Error happens in main.c.

Error: division by zero.

-10.000000 = -10.000000
~~~

This tells us that the user-provided handler is called first, then the default handler is called.
If you want your handler called after the default handler, then you can use `g_signal_connect_after`.
Add the lines below to `main.c` again.

~~~C
static void
div_by_zero_after_cb (TDouble *d, gpointer user_data) {
  g_print ("\nError has happened in main.c and an error message has been displayed.\n");
}

int
main (int argc, char **argv) {
... ... ...
  g_signal_connect_after (d1, "div-by-zero", G_CALLBACK (div_by_zero_after_cb), NULL);
... ... ...
}
~~~

Compile and execute it, then:

~~~
10.000000 + 20.000000 = 30.000000
10.000000 - 20.000000 = -10.000000
10.000000 * 20.000000 = 200.000000
10.000000 / 20.000000 = 0.500000

Error happens in main.c.

Error: division by zero.

Error has happened in main.c and an error message has been displayed.

-10.000000 = -10.000000
~~~

The source files are in [src/tdouble5](tdouble5).

## Signal flag

The order that handlers are called is described in {GObject API reference](https://developer.gnome.org/gobject/stable/gobject-Signals.html#gobject-Signals.description).

The order depends on the signal flag which is set in `g_signal_new` or `g_signal_new_class_handler`.
There are three flags which relate to the order of handlers' invocation.

- `G_SIGNAL_RUN_FIRST`: the default handler is called before any user provided handler.
- `G_SIGNAL_RUN_LAST`: the default handler is called after the normal user provided handler (not connected with `g_signal_connect_after`).
- `G_SIGNAL_RUN_CLEANUP`: the default handler is called after any user provided handler.

`G_SIGNAL_RUN_LAST` is the most appropriate in many cases.

Other signal flags are described in [GObject API reference](https://developer.gnome.org/gobject/stable/gobject-Signals.html#GSignalFlags).
