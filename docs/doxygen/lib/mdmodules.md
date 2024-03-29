mdrun modules {#page_mdmodules}
=============

Currently, most of mdrun is constructed as a set of C routines calling each
other, and sharing data through a couple of common data structures (t_inputrec,
t_forcerec, t_state etc.) that flow throughout the code.

The electric field code (in `src/gromacs/applied-forces/`) implements an
alternative concept that allows keeping everything related to the electric
field functionality in a single place.  At least for most special-purpose
functionality, this would hopefully provide a more maintainable approach that
would also support more easily adding new functionality.  Some core features
may still need stronger coupling than this provides.

The rest of the page documents those parts of the modularity mechanism that
have taken a clear form.  Generalizing and designing other parts may require
more code to be converted to modules to have clearer requirements on what the
mechanism needs to support and what is the best way to express that in a
generally usable form.

Handling mdp input
------------------

To accept parameters from an mdp file, a module needs to implement two methods
in gmx::IInputRecExtension.

initMdpOptions() should declare the required input parameters using the options
module.  In most cases, the parameters should be declared as nested sections
instead of a flat set of options.  The structure used should be such that in
the future, we can get the input values from a structured mdp file (e.g., JSON
or XML), where the structure matches the declared options.  As with other uses
of the options module, the module needs to declare local variables where the
values from the options will be assigned.  The defined structure will also be
used for storing in the tpr file (see below).

initMdpTransform() should declare the mapping from current flat mdp format to
the structured format defined in initMdpOptions().  For now, this makes it
possible to have an internal forward-looking structured representation while
the input is still a flat list of values, but in the future it also allows
supporting both formats side-by-side as long as that is deemed necessary.

On the implementation side, the framework (and other code that interacts with
the modules) will do the following things to make mdp input work:

* When grompp reads the mdp file, it will first construct a flat
  KeyValueTreeObject, where each input option is set as a property.

  It then calls initMdpTransform() for the module(s), and uses the produced
  transform to convert the flat tree into a structured tree, performing any
  defined conversions in the process.  This transformation is one-way only,
  although the framework keeps track of the origin of each value to provide
  sensible error messages that have the original mdp option name included.
  Because the mapping is not two-way, there is some loss of functionality
  related to mdp file handling in grompp (in particular, the `-po` option) for
  options belonging to the new-style modules.

  It calls initMdpOptions() for the module(s), initializing a single Options
  object that has the input options.

  It processes the structured tree using the options in two steps:

  * For any option that is not specified in the input, it adds a property to
    the tree with a default value.  For options specified in the input, the
    values in the tree are converted to native values for the options (e.g.,
    from string to int for integer options).
  * It assigns the values from the tree to the Options object.  This will make
    the values available in the local variables the module defined in
    initMdpOptions().

  Note that currently, the module(s) cannot use storeIsSet() in options to know
  whether a particular option has been provided from the mdp file.  This will
  always return true for all the options.  This is a limitation in the current
  implementation, but it also, in part, enforces that the mdp file written out
  by `gmx grompp -po` cannot produce different behavior because of set/not-set
  differences.

* When grompp writes the tpr file, it writes the structured tree (after the
  default value and native value conversion) into the tpr file.

* When mdrun reads the tpr file, it reads the structured tree.
  It then broadcasts the structure to all ranks.  Each rank calls
  initMdpOptions() for the modules, and assigns the values from the tree to the
  Options object.  After this, the modules will be exactly in the same state as
  in grompp.

* When other tools (gmx dump or gmx check in particular) read the tpr file,
  they read the structured tree.  In principle, they could operate directly on
  this tree (and `gmx dump` in particular does, with the `-orgir` option).
  However, in the future with proper tpr backward compatibility support, they
  need to call to the modules to ensure that the tree has the structure that
  this version expects, instead of what the original version that wrote the
  file had.  Currently, these tools only call initMdpOptions() and do the basic
  default+native value conversion.

* Any code that is not interested in the parameters for these modules can just
  read the t_inputrec from the tpr file and ignore the tree.

* For compatibility with old tpr files that did not yet have the structured
  tree, the I/O code converts old values for the modules to parameters in the
  structured tree (in tpxio.cpp).

Currently, there is no mechanism for changing the mdp input parameters (adding
new or removing old ones) that would maintain tpr and mdp backward
compatibility.  The vision for this is to use the same transformation engine as
for initMdpTransform() to support specifying version-to-version conversions for
any changed options, and applying the necessary conversions in sequence.  The
main challenge is keeping track of the versions to know which conversions to
apply.
