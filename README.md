# react-docgen [![Build Status](https://travis-ci.org/reactjs/react-docgen.svg?branch=master)](https://travis-ci.org/reactjs/react-docgen)

`react-docgen` is a CLI and toolbox to help extracting information from [React][] components, and generate documentation from it.

It uses [recast][] and [babylon][] to parse the source into an AST and provides methods to process this AST to extract the desired information. The output / return value is a JSON blob / JavaScript object.

It provides a default implementation for React components defined via 
`React.createClass`, [ES2015 class definitions][classes] or functions 
(stateless components). These component definitions must follow certain 
guidelines in order to be analyzable (see below for more info).

## Install

Install the module directly from npm:

```
npm install -g react-docgen
```

## CLI

Installing the module adds a `react-docgen` executable which allows you to convert
a single file, multiple files or an input stream. We are trying to make the
executable as versatile as possible so that it can be integrated into many
workflows.

```
Usage: react-docgen [path]... [options]

path     A component file or directory. If no path is provided it reads from stdin.

Options:
   -o FILE, --out FILE   store extracted information in FILE
   --pretty              pretty print JSON
   -x, --extension       File extensions to consider. Repeat to define multiple extensions. Default:  [js,jsx]
   -i, --ignore          Folders to ignore. Default:  [node_modules,__tests__]
   --resolver RESOLVER   Resolver name (findAllComponentDefinitions, findExportedComponentDefinition) or
      path to a module that exports a resolver.  [findExportedComponentDefinition]

Extract meta information from React components.
If a directory is passed, it is recursively traversed.
```

By default, `react-docgen` will look for the exported component created through 
`React.createClass`, a class definition or a function (stateless component) in 
each file. You can change that behavior with the `--resolver` option, which 
either expects the name of a built-in resolver or a path to JavaScript module 
exporting a resolver function. Have a look below for [more information about 
resolvers](#resolver).

Have a look at `example/` for an example of how to use the result to generate a
markdown version of the documentation.

## API

The tool can be used programmatically to extract component information and customize the extraction process:

```js
var reactDocs = require('react-docgen');
var componentInfo = reactDocs.parse(src);
```

As with the CLI, this will look for the exported component created through `React.createClass` or a class definition in the provided source. The whole process of analyzing the source code is separated into two parts:

- Locating/finding the nodes in the AST which define the component
- Extracting information from those nodes

`parse` accepts more arguments with which this behavior can be customized.

### parse(source \[, resolver \[, handlers\]\])

| Parameter |  Type | Description |
| -------------- | ------ | --------------- |
| source       | string | The source text |
| resolver     | function | A function of the form `(ast: ASTNode, recast: Object) => (NodePath|Array<NodePath>)`. Given an AST and a reference to recast, it returns an (array of) NodePath which represents the component definition. |
| handlers    | Array\<function\> | An array of functions of the form `(documentation: Documentation, definition: NodePath) => void`. Each function is called with a `Documentation` object and a reference to the component definition as returned by `resolver`. Handlers extract relevant information from the definition and augment `documentation`. |

#### resolver

The resolver's task is to extract those parts from the source code which the handlers can analyze. For example, the `findExportedComponentDefinition` resolver inspects the AST to find

```js
var Component = React.createClass(<def>);
module.exports = Component;

// or

class Component extends React.Component {
  // ...
}
module.exports = Component;
```

and returns the ObjectExpression to which `<def>` resolves to, or the class declaration itself.

`findAllComponentDefinitions` works similarly, but finds *all* `React.createClass` calls and class definitions, not only the one that is exported.

This makes it easy, together with the utility methods created to analyze the AST, to introduce new or custom resolver methods. For example, a resolver could look for plain ObjectExpressions with a `render` method.

#### handlers

Handlers do the actual work and extract the desired information from the result the resolver returned. Like the resolver, they try to delegate as much work as possible to the reusable utility functions.

For example, while the `propTypesHandler` expects the prop types definition to be an ObjectExpression and be available as `propTypes` in the component definition, most of the work is actually performed by the `getPropType` utility function.

## Guidelines for default resolvers and handlers

- Modules have to export a single component, and only that component is analyzed.
- When using `React.createClass`, the component definition (the value passed to it) must resolve to an object literal.
- When using classes, the class must either `extend React.Component` *or* define a `render()` method.
- `propTypes` must be an object literal or resolve to an object literal in the same file.
- The `return` statement in `getDefaultProps` must contain an object literal.

## Example

For the following component

```js
var React = require('react');

/**
 * General component description.
 */
var Component = React.createClass({
  propTypes: {
    /**
     * Description of prop "foo".
     */
    foo: React.PropTypes.number,
    /**
     * Description of prop "bar" (a custom validation function).
     */
    bar: function(props, propName, componentName) {
      // ...
    },
    baz: React.PropTypes.oneOfType([
      React.PropTypes.number,
      React.PropTypes.string
    ]),
  },

  getDefaultProps: function() {
    return {
      foo: 42,
      bar: 21
    };
  },

  render: function() {
    // ...
  }
});

module.exports = Component;
```

we are getting this output:

```
{
  "props": {
    "foo": {
      "type": {
        "name": "number"
      },
      "required": false,
      "description": "Description of prop \"foo\".",
      "defaultValue": {
        "value": "42",
        "computed": false
      }
    },
    "bar": {
      "type": {
        "name": "custom"
      },
      "required": false,
      "description": "Description of prop \"bar\" (a custom validation function).",
      "defaultValue": {
        "value": "21",
        "computed": false
      }
    },
    "baz": {
      "type": {
        "name": "union",
        "value": [
          {
            "name": "number"
          },
          {
            "name": "string"
          }
        ]
      },
      "required": false,
      "description": ""
    }
  },
  "description": "General component description."
}
```

## Result data structure

The structure of the JSON blob / JavaScript object is as follows:

```
{
  ["description": string,]
  ["props": {
    "<propName>": {
      "type": {
        "name": "<typeName>",
        ["value": <typeValue>]
        ["raw": string]
      },
      "required": boolean,
      "description": string,
      ["defaultValue": {
        "value": number | string,
        "computed": boolean
      }]
    },
    ...
  },]
  ["composes": <componentNames>]
}
```
(`[...]` means the property may not exist if such information was not found in the component definition)

- `<propName>`:  For each prop that was found, there will be an entry in `props` under the same name.
- `<typeName>`: The name of the type, which is usually corresponds to the function name in `React.PropTypes`. However, for types define with `oneOf`, we use `"enum"`  and for `oneOfType` we use `"union"`. If a custom function is provided or the type cannot be resolved to anything of `React.PropTypes`, we use `"custom"`.
- `<typeValue>`: Some types accept parameters which define the type in more detail (such as `arrayOf`, `instanceOf`, `oneOf`, etc). Those are stored in `<typeValue>`. The data type of `<typeValue>` depends on the type definition.


[react]: http://facebook.github.io/react/
[recast]: https://github.com/benjamn/recast
[babylon]: https://github.com/babel/babel/tree/master/packages/babylon
[classes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes
