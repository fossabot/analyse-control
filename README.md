Analyse-Control
===============

**Extract the control flow graph from a script.**
Control flow refers to what order a set of instructions execute in. By using
conditional statements and loops, the order of a set of instructions can be
changed. This library extracts all possible execution flows through a script as
a graph of nodes.

    Author: Olli Jones
    License: MIT

```zsh
npm install analyse-control --save
```

## Introduction

Given this example piece of [ECMAScript 5](https://github.com/estree/estree/blob/master/es5.md) (JavaScript) as input:

```javascript
{
  helloWorld();
}
```

An abstract syntax tree (AST) generated by
[acorn](https://www.npmjs.com/package/acorn) would look similar to this:

```json
{
  "type": "Program",
  "body": [
    {
      "type": "BlockStatement",
      "body": [
        {
          "type": "ExpressionStatement",
          "expression": {
            "type": "CallExpression",
            "callee": {
              "type": "Identifier",
              "name": "helloWorld"
            },
            "arguments": []
          }
        }
      ]
    }
  ]
}
```

This contains 5 nested AST types:

 - [Program](https://github.com/estree/estree/blob/master/es5.md#programs)
 - [BlockStatement](https://github.com/estree/estree/blob/master/es5.md#blockstatement)
 - [ExpressionStatement](https://github.com/estree/estree/blob/master/es5.md#expressionstatement)
 - [CallExpression](https://github.com/estree/estree/blob/master/es5.md#callexpression)
 - [Identifier](https://github.com/estree/estree/blob/master/es5.md#identifier)

When thinking about how this program would execute in an interpreter, we would
expect the code to step through the elements, first evaluating what value is
set for `helloWorld`, then calling the function `helloWorld()`, etc, until
the Program has been fully evaluated.

This library will represent this control flow, as a graph where control moves
from one statement to the next:

 - `Program: hoist`
 - `Program: enter`
 - `BlockStatement: enter`
 - `ExpressionStatement: enter`
 - `CallExpression: enter`
 - `Identifier: enter`
 - `Identifier: exit`
 - `CallExpression: exit`
 - `ExpressionStatement: exit`
 - `BlockStatement: exit`
 - `Program: exit`


 > *Hoist* refers to variable hoisting, there's a good
 [w3 tutorial](https://www.w3schools.com/js/js_hoisting.asp) that gives a good
 overview

Analyse-control allows for the inspection of possible control flow execution
without actually executing any of the given code.

The bullet point list above, for example, can be generated using:

```javascript
var flow = require("analyse-control")(ast).getStartOfFlow();

while(flow != undefined) {
  if (flow.isHoist()) {
    console.log(flow.getNode().type + ": hoist");
  }
  if (flow.isEnter()) {
    console.log(flow.getNode().type + ": enter");
  }
  if (flow.isExit()) {
    console.log(flow.getNode().type + ": exit");
  }
  flow = flow.getForwardFlows()[0];
}
```

Notice that `getForwardFlows()` returns an array. This is because:
 - When encountering a fork in the control flow graph, ie. an `IfStatement`,
    `WhileStatement`, or similar conditional expression: both possible control
    flow progressions are returned.
 - The control flow could also terminate (in non-looping programs) through the
    use of a `ThrowStatement` or the program naturally coming to the end of the script: then the array will be empty.

The example code below shows how this functionality can be used to calculate
the unique number of possible execution paths through a branching algorithm:

## Example usage

  > Look in [./test/integration.js](./test/integration.js) for more examples

```javascript
var analyse = require("analyse-control");
var acorn = require("acorn");

var ast = acorn.parse([
  "if (x) {",
  "  hello();",
  "} else {",
  "  world();",
  "}"
].join("\n"));

var flow = analyse(ast);

function countBranches(node, visited) {
  var nodeId = node.getId();
  var outgoing = node.getForwardFlows();
  if (visited.indexOf(nodeId) != -1) {
    // we've visited this node before - we're in a loop
    return Infinity;
  }
  if (outgoing.length == 0) {
    // we've reached the termination of this single control flow
    return 1;
  }
  // we can count the number of execution paths, by adding up the
  // counts from recursively exploring all branches from this node:
  return outgoing.reduce((counter, node) => (
    counter + countBranches(node, visited.concat(nodeId))
  ), 0);
}

console.log(
  "There are " +
  countBranches(flow.getStartOfFlow(), []) +
  " possible paths through the code");
// prints "There are 2 possible paths through the code"
```

Appending an additional `IfStatement` should correctly return "4 possible
paths". Similarly adding a `WhileStatement` will return "Infinity possible
paths" (because the loop has infinite variations on how many times it will
execute).

## API documentation

### `analyse(ast: AST) : Graph`

`require('analyse-control')` returns a method that when given an AST (abstract
syntax tree) from a parser like [acorn](https://www.npmjs.com/package/acorn) it
will return a control flow graph "Graph".

### `Graph.getStartOfFlow() : FlowNode`

The control flow graph can either be traversed from start to finish, or from
finish to start. `getStartOfFlow()` will get the first executed node of the
AST. This will probably be of type [Program](https://github.com/estree/estree/blob/master/es5.md#programs).

`Graph.getStartOfFlow().isHoist()` will be true.

### `Graph.getEndOfFlow() : FlowNode`

Similar to the method above, but this method will return the last executed
node, which will also be [Program](https://github.com/estree/estree/blob/master/es5.md#programs).

`Graph.getEndOfFlow().isExit()` will be true.

### `FlowNode.isHoist() : Boolean`

JavaScript programs have their `var` definitions hoisted.

For example:
```javascript
var x = "hello";
function y() {
  return x;
  var x;
}
y() == undefined;
```

Even though `x` is seemingly defined as `"hello"` inside `y()`, the second
declaration inside the function is *hoisted* before the return statement. Thus
the output is `undefined`.

This library will correctly return `hoist` flows to cover this behaviour. The
FlowNodes will be marked as `isHoist() == true`

Note this library specifically adheres to the
[Chrome/IE/Safari](http://code.google.com/p/v8/issues/detail?id=437)
standard way of hoisting non-standard conditional declarations.

### `FlowNode.isEnter() : Boolean`

Flow can enter and exit statements, for example we'll first enter a
BlockStatement, execute any statements inside, and then exit the BlockStatement.

This method will confirm whether we are entering into the statement by returning
true.

### `FlowNode.isExit() : Boolean`

Flow can enter and exit statements, for example we'll first enter a
BlockStatement, execute any statements inside, and then exit the BlockStatement.

This method will confirm whether we are exiting the statement by returning true.

### `FlowNode.getId() : Number | String`

Returns a unique value for this flow, encapsulating whether it represents
entering, exiting or hoisting a statement, and which statement specifically.

This can be used for detecting cycles, or for memoizing dynamic programming
calls.

Usually this will be a positive integer, but where there aren't enough unique
integers to represent every FlowNode then both integers and strings will be
used.

### `FlowNode.getForwardFlows() : [FlowNode]`

This will get all nodes that could be executed next.

Commonly used in combination with `Graph.getStartOfFlow()`.

For example if we're in the conditional of an `IfStatement`, the next executed
statement could be either the consequent or alternate.

The return value is an array of all nodes, which can be an array of a single
value when there are no conditional statements following this FlowNode.
It can be an array of two FlowNodes, where there is a conditional.
It can be an empty array when there are no statements to execute after this one,
for example this statement is the last FlowNode in a Program.

### `FlowNode.getBackwardsFlows() : [FlowNode]`

This will get all nodes that flowed into this node.

Commonly used in combination with `Graph.getEndOfFlow()`.

For example if we're inside an if consequent or alternate, the backwards flows
would be the conditional statement that led up to the execution of this node.

The return value is an array of all nodes, which can be an array of a single
node when there were no conditionals leading up to the current FlowNode, it can
be an array of two FlowNodes when this statement follows a conditional
statement. Finally it can be an empty array when the FlowNode is unreachable,
for example because of throw or return statement.

### `FlowNode.getNode() : ASTNode`

Gets a representation for the node which is similar to the input AST, however,
the inner ASTNodes will be replaced by numerical placeholders. Use `Graph.getNode(id)` to resolve these references.

### `Graph.getNode(id: Number) : ASTNode`

This will return a representation similar to a node in the input AST, however,
it will have had all inner ASTNodes replaced by numerical placeholders. Use the
method recursively to obtain a copy of the original AST.

## Installing dependencies and running tests

    npm it

## Running visualiser

![Image of visualisation tool](docs/flow_animation.gif)

> Use `npx analyse-control ./path/to/script.js` to see a visualisation of the
  control flow graph of a given script

## Known issues

 - Only [ECMAScript 5](https://github.com/estree/estree/blob/master/es5.md) is
   supported

 - Does not model exception flow, it only models flows from an explicit `throws`
   statement through to a `catch` block
