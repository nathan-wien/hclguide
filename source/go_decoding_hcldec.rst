.. _go-decoding-hcldec:

Decoding With Dynamic Schema with the ``hcldec`` package
========================================================

In section :ref:`go-decoding-gohcl`, we saw the most straightforward way to
access the content from an HCL file, decoding directly into a Go value whose
type is known at application compile time.

For some applications, it is not possible to know the schema of the entire
configuration when the application is built. For example, `HashiCorp Terraform`_
uses HCL as the foundation of its configuration language, but parts of the
configuration are handled by plugins loaded dynamically at runtime, and so
the schemas for these portions cannot be encoded directly in the Terraform
source code.

HCL's `hcldec <https://pkg.go.dev/github.com/hashicorp/hcl/v2@v2.16.0/hcldec>`_ package offers a different approach to decoding that allows
schemas to be created at runtime, and the result to be decoded into
dynamically-typed data structures.

The sections below are an overview of the main parts of package ``hcldec``.
For full details, see
`the package godoc <https://godoc.org/github.com/hashicorp/hcl2/hcldec>`_.

.. _`HashiCorp Terraform`: https://www.terraform.io/

Decoder Specification
---------------------

Whereas ``gohcl`` infers the expected schema by using reflection against
the given value, ``hcldec`` obtains schema through a decoding *specification*,
which is a set of instructions for mapping HCL constructs onto a dynamic
data structure.

The ``hcldec`` package contains a number of different specifications, each
implementing ``hcldec.Spec`` and having a ``Spec`` suffix on its name.
Each spec has two distinct functions:

* Adding zero or more validation constraints on the input configuration file.

* Producing a result value based on some elements from the input file.

The most common pattern is for the top-level spec to be a
``hcldec.ObjectSpec`` with nested specifications defining either blocks
or attributes, depending on whether the configuration file will be
block-structured or flat.

.. code-block:: go

  spec := hcldec.ObjectSpec{
      "io_mode": &hcldec.AttrSpec{
          Name: "io_mode",
          Type: cty.String,
      },
      "services": &hcldec.BlockMapSpec{
          TypeName:   "service",
          LabelNames: []string{"type", "name"},
          Nested:     hcldec.ObjectSpec{
              "listen_addr": &hcldec.AttrSpec{
                  Name:     "listen_addr",
                  Type:     cty.String,
                  Required: true,
              },
              "processes": &hcldec.BlockMapSpec{
                  TypeName:   "service",
                  LabelNames: []string{"name"},
                  Nested:     hcldec.ObjectSpec{
                      "command": &hcldec.AttrSpec{
                          Name:     "command",
                          Type:     cty.List(cty.String),
                          Required: true,
                      },
                  },
              },
          },
      },
  }
  val, moreDiags := hcldec.Decode(f.Body, spec, nil)
  diags = append(diags, moreDiags...)

The above specification expects a configuration shaped like our example in
:ref:`intro`, and calls for it to be decoded into a dynamic data structure
that would have the following shape if serialized to JSON:

.. code-block:: JSON

  {
    "io_mode": "async",
    "services": {
      "http": {
        "web_proxy": {
          "listen_addr": "127.0.0.1:8080",
          "processes": {
            "main": {
              "command": ["/usr/local/bin/awesome-app", "server"]
            },
            "mgmt": {
              "command": ["/usr/local/bin/awesome-app", "mgmt"]
            }
          }
        }
      }
    }
  }
