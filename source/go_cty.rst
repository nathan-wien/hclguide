Package ``cty``
===============

Types and Values With ``cty``
-----------------------------

HCL's expression interpreter is implemented in terms of another library called
``cty``, which provides a type system which HCL builds on and a robust
representation of dynamic values in that type system. You could think of
``cty`` as being a bit like Go's own ``reflect``, but for the
results of HCL expressions rather than Go programs.

The full details of this system can be found in
`its own repository <https://github.com/zclconf/go-cty>`_, but this section
will cover the most important highlights, because ``hcldec`` specifications
include ``cty`` types (as seen in the above example) and its results are
``cty`` values.

``hcldec`` works directly with ``cty`` — as opposed to converting values
directly into Go native types — because the functionality of the ``cty``
packages then allows further processing of those values without any loss of
fidelity or range. For example, ``cty`` defines a JSON encoding of its
values that can be decoded losslessly as long as both sides agree on the value
type that is expected, which is a useful capability in systems where some sort
of RPC barrier separates the main program from its plugins.

Types are instances of ``cty.Type``, and are constructed from functions
and variables in ``cty`` as shown in the above example, where the string
attributes are typed as ``cty.String``, which is a primitive type, and the list
attribute is typed as ``cty.List(cty.String)``, which constructs a new list
type with string elements.

Values are instances of ``cty.Value``, and can also be constructed from
functions in ``cty``, using the functions that include ``Val`` in their
names or using the operation methods available on ``cty.Value``.

In most cases you will eventually want to use the resulting data as native Go
types, to pass it to non-``cty``-aware code. To do this, see the guides
on
`Converting between types <https://github.com/zclconf/go-cty/blob/master/docs/convert.md>`_
(staying within ``cty``) and
`Converting to and from native Go values <https://github.com/zclconf/go-cty/blob/master/docs/gocty.md>`_.

Partial Decoding
----------------

Because the ``hcldec`` result is always a value, the input is always entirely
processed in a single call, unlike with ``gohcl``.

However, both ``gohcl`` and ``hcldec`` take ``hcl.Body`` as
the representation of input, and so it is possible and common to mix them both
in the same program.

A common situation is that ``gohcl`` is used in the main program to
decode the top level of configuration, which then allows the main program to
determine which plugins need to be loaded to process the leaf portions of
configuration. In this case, the portions that will be interpreted by plugins
are retained as opaque ``hcl.Body`` until the plugins have been loaded,
and then each plugin provides its ``hcldec.Spec`` to allow decoding the
plugin-specific configuration into a ``cty.Value`` which be
transmitted to the plugin for further processing.

In our example from :ref:`intro`, perhaps each of the different service types
is managed by a plugin, and so the main program would decode the block headers
to learn which plugins are needed, but process the block bodies dynamically:

.. code-block:: go

   type ServiceConfig struct {
     Type         string   `hcl:"type,label"`
     Name         string   `hcl:"name,label"`
     PluginConfig hcl.Body `hcl:",remain"`
   }
   type Config struct {
     IOMode   string          `hcl:"io_mode"`
     Services []ServiceConfig `hcl:"service,block"`
   }

   var c Config
   moreDiags := gohcl.DecodeBody(f.Body, nil, &c)
   diags = append(diags, moreDiags...)
   if moreDiags.HasErrors() {
       // (show diags in the UI)
       return
   }

   for _, sc := range c.Services {
       pluginName := block.Type

       // Totally-hypothetical plugin manager (not part of HCL)
       plugin, err := pluginMgr.GetPlugin(pluginName)
       if err != nil {
           diags = diags.Append(&hcl.Diagnostic{ /* ... */ })
           continue
       }
       spec := plugin.ConfigSpec() // returns hcldec.Spec

       // Decode the block body using the plugin's given specification
       configVal, moreDiags := hcldec.Decode(sc.PluginConfig, spec, nil)
       diags = append(diags, moreDiags...)
       if moreDiags.HasErrors() {
           continue
       }

       // Again, hypothetical API within your application itself, and not
       // part of HCL. Perhaps plugin system serializes configVal as JSON
       // and sends it over to the plugin.
       svc := plugin.NewService(configVal)
       serviceMgr.AddService(sc.Name, svc)
   }


Variables and Functions
-----------------------

The final argument to ``hcldec.Decode`` is an expression evaluation context,
just as with ``gohcl.DecodeBlock``.

This object can be constructed using
:ref:`the gohcl helper function <go-decoding-gohcl-evalcontext>` as before if desired, but
you can also choose to work directly with ``hcl.EvalContext`` as
discussed in :ref:`go-expression-eval`:

.. code-block:: go

    ctx := &hcl.EvalContext{
        Variables: map[string]cty.Value{
            "pid": cty.NumberIntVal(int64(os.Getpid())),
        },
    }
    val, moreDiags := hcldec.Decode(f.Body, spec, ctx)
    diags = append(diags, moreDiags...)

As you can see, this lower-level API also uses ``cty``, so it can be
particularly convenient in situations where the result of dynamically decoding
one block must be available to expressions in another block.
