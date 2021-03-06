Querying the Database
======================

This chapter discusses how the database contents generated by Joern
can be queried to locate interesting code. We begin by reviewing the
basics of the graph traversal language Gremlin and proceed to discuss
how to select start nodes. The remainder of this chapter deals with
code retrieval based on syntax, taint-style queries and finally,
traversals in the function symbol graph.

Gremlin Basics
---------------
In this section, we will give a brief overview of the most basic
functionality offered by the graph traversal language Gremlin
developed by Marko A. Rodriguez. For detailed documentation of
language features, please refer to
http://tinkerpop.apache.org/docs/3.0.1-incubating/ .

Gremlin is a language designed to describe walks in property graphs. A
property graph is simply a graph where key-value pairs can be attached
to nodes and edges. (From a programmatic point of view, you can simply
think of it as a graph that has hash tables attached to nodes and
edges.) In addition, each edge has a type, and that's all you need to
know about property graphs for now.

Graph traversals proceed in two steps to uncover to search a database
for sub-graphs of interest:

1. **Start node selection.** All traversals begin by selecting a set of
nodes from the database that serve as starting points for walks in the
graph.

2. **Walking the graph.** Starting at the nodes selected in the
previous step, the traversal walks along the graph edges to reach
adjacent nodes according to properties and types of nodes and
edges. The final goal of the traversal is to determine all nodes that
can be reached by the traversal. You can think of a graph traversal as
a sub-graph description that must be fulfilled in order for a node to
be returned.

The simplest way to select start nodes is to perform a lookup based on
the unique node key.

.. code-block:: none

	// Lookup node with given key
	g.V.has('_key', key)

Walking the graph can now be achieved by attaching so called
*Gremlin steps* to the start node. Each of these steps processes
all nodes returned by the previous step, similar to the way Unix
pipelines connect shell programs. While learning Gremlin, it can thus
be a good idea to think of the dot-operator as an alias for the unix
pipe operator ``|``. The following is a list of examples.

.. code-block:: none

	// Traverse to nodes connected to start node by outgoing edges
	g.V.has('_key', key).out()

	// Traverse to nodes two hops away.
	g.V.has('_key', key).out().out()

	// Traverse to nodes connected to start node by incoming edges
	g.V.has('_key', key).in()

	// All nodes connected by outgoing AST edges (filtering based
	// on edge types)
	g.V.has('_key', key).out(AST_EDGE)

	// Filtering based on properties:
	g.V.has('_key', key).out().has('type', typeOfInterest)	

	// Filtering based on edge properties
	g.V.has('_key', key).outE(AST_EDGE).has(propKey, propValue).inV()


Start Node Selection
---------------------

In practice, the ids or keys of interesting start nodes are rarely
known. Instead, start nodes are selected based on node properties, for
example, one may want to select all calls to function ``memcpy`` as
start nodes. For example, to retrieve all AST nodes representing
callees with a name containing the substring `cpy`, one may issue the
following query:

.. code-block:: none

	g.V.has('type', 'Callee').has('code', textRegex('.*cpy.*')).code	


This is quite lengthly. Fortunately, once you identify a common
operation, you can create a custom step to perform this operation in
the future. We have collected common and generic steps of this type in
the query library `joern-lang`, which you find in
`projects/joern-lang`. In particular, we have defined the step
`getCallsToRegex` in `lookup.groovy`, so the previous query can also
be written as:

.. code-block:: none

	getCallsToRegex("*cpy*")

Please do not hesitate to contribute short-hands for common lookup
operations to include in ``lookup.groovy``.

Traversing Syntax Trees
------------------------

In the previous section, we outlined how nodes can be selected based
on their properties. As outline in Section `Gremlin Basics`_, these
selected nodes can now be used as starting points for walks in the
property graph.

As an example, consider the task of finding all multiplications in
first arguments of calls to the function ``malloc``. To solve this
problem, we can first determine all call expressions to ``malloc``
and then traverse from the call to its first argument in the syntax
tree. We then determine all multiplicative expressions that are child
nodes of the first argument.

In principle, all of these tasks could be solved using the elementary
Gremlin traversals presented in Section `Gremlin Basics`_. However,
traversals can be greatly simplified by introducing the following
user-defined gremlin-steps (see ``joernsteps/ast.py``).

.. code-block:: none

	// Traverse to parent nodes in the AST
	parents()

	// Traverse to child nodes in the AST
	children()

	// Traverse to i'th children in the AST
	ithChildren()

	// Traverse to enclosing statement node
	statements()

	// Traverse to all nodes of the AST
	// rooted at the input node
	astNodes()

Additionally, ``calls.groovy`` introduces user-defined
steps for traversing calls, and in particular the step
``ithArguments`` that traverses to i'th arguments of a given a call
node. Using these steps, the exemplary traversal for multiplicative
expressions inside first arguments to ``malloc`` simply becomes:

.. code-block:: none

	getCallsTo('malloc').ithArguments('0')
	.astNodes()
	.hasRegex(NODE_TYPE, '.*Mul.*')


Syntax-Only Descriptions
------------------------

The file ``composition.groovy`` offers a number of
elementary functions to combine other traversals and lookup
functions. These composition functions allow arbitrary syntax-only
descriptions to be constructed (see `Modeling and Discovering
Vulnerabilities with Code Property Graphs
<http://user.informatik.uni-goettingen.de/~fyamagu/pdfs/2014-oakland.pdf>`_
). For example, to select all functions that contain a call to ``foo``
AND a call to ``bar``, lookup functions can simply be chained, e.g.,

.. code-block:: none

	getCallsTo('foo').getCallsTo('bar')

returns functions calling both ``foo`` and ``bar``. Similarly,
functions calling `foo` OR `bar` can be selected as follows:

.. code-block:: none

	OR( getCallsTo('foo'), getCallsTo('bar') )


Finally, the ``not``-traversal allows all nodes to be selected
that do NOT match a traversal. For example, to select all functions
calling `foo` but not `bar`, use the following traversal:

.. code-block:: none

	getCallsTo('foo').not{ getCallsTo('bar') }

Traversing the Symbol Graph
----------------------------

As outlined in Section :doc:`databaseOverview`, the symbols used and
defined by statements are made explicit in the graph database by
adding symbol nodes to functions (see Appendix D of `Modeling and Discovering
Vulnerabilities with Code Property Graphs
<http://user.informatik.uni-goettingen.de/~fyamagu/pdfs/2014-oakland.pdf>`_). We
provide utility traversals to make use of this in order to determine
symbols defining variables, and thus simple access to types used by
statements and expressions. In particular, the file
``symbolGraph.groovy`` contains the following steps:

.. code-block:: none

	// traverse from statement to the symbols it uses
	uses()

	// traverse from statement to the symbols it defines
	defines()

	// traverse from statement to the definitions
	// that it is affected by (assignments and
	// declarations)
	definitions()

As an example, consider the task of finding all third arguments to
``memcpy`` that are defined as parameters of a function. This can be achieved using the traversal

.. code-block:: none

	getArguments('memcpy', '2').definitions()
	.filter{it.type == TYPE_PARAMETER}

where ``getArguments`` is a lookup-function defined in
``lookup.py``.

As a second example, we can traverse to all functions that use a
symbol named ``len`` in a third argument to ``memcpy`` that is not
used by any condition in the function, and hence, may not be checked.

.. code-block:: none

	getArguments('memcpy', '2').uses()
	.filter{it.code == 'len'}
	.filter{
		it.in('USES')
		.filter{it.type == 'Condition'}.toList() == []
	}

This example also shows that traversals can be performed inside
filter-expressions and that at any point, a list of nodes that the
traversal reaches can be obtained using the function ``toList``
defined on all Gremlin steps.

Taint-Style Descriptions
-------------------------

The last example already gave a taste of the power you get when you
can actually track where identifiers are used and defined. However,
using only the augmented function symbol graph, you cannot be sure the
definitions made by one statement actually *reach* another
statement. To ensure this, the classical *reaching definitions*
problem needs to be solved. In addition, you cannot track whether
variables are sanitized on the way from a definition to a statement.

Fortunately, joern allows you to solve both problems using the
traversal ``unsanitized``. As an example, consider the case where
you want to find all functions where a third argument to ``memcpy``
is named ``len`` and is passed as a parameter to the function and a
control flow path exists satisfying the following two conditions:


* The variable ``len`` is not re-defined on the way.
* The variable is not used inside a relational or equality expression
  on the way, i.e., its numerical value is not ``checked'' against
  some other variable.

You can use the following traversal to achieve this:

.. code-block:: none

	getArguments('memcpy', '2')
	.sideEffect{ paramName = '.*len.*' }
	.unsanitized({ it, s -> it.isCheck(paramName) })
	.match{ it.type == "Parameter" && it.code.matches(paramName) }.code

where ``isCheck`` is a traversal defined in ``misc.groovy``
to check if a symbol occurs inside an equality or relational
expression and `match` looks for nodes matches the closure passed
to it in the given syntax tree.

Note, that in the above example, we are using a regular expression to
determine arguments containing the sub-string ``len`` and that one may
want to be a little more exact here. Also, we use the Gremlin step
``sideEffect`` to save the regular expression in a variable, simply so
that we do not have to re-type the regular expression over and over.
