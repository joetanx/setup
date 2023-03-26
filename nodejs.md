## Some notes on how exports and module.exports work in Node.js

Full sample code:

```js
const function1 = function() {
  console.log('i am a function')
}
exports.messenger = {
  showMessage: function(msg) {
    console.log(msg)
  }
}
exports.messenger.showMessage('this is a message')
console.log(module)
var value1 = 50
exports.data1 = value1
console.log(module)
var value2 = 100
exports.data2 = value2
console.log(module)
module.exports = {value2,function1}
console.log(module)
console.log(exports)
module.exports.function1('test')
module.anything = {value1}
module.whatever = {value2}
console.log(module)
console.log(module.anything.value1)
console.log(module.whatever.value2)
```

Full sample output:

```js
this is a message
Module {
  id: '.',
  path: '/root',
  exports: { messenger: { showMessage: [Function: showMessage] } },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
Module {
  id: '.',
  path: '/root',
  exports: { messenger: { showMessage: [Function: showMessage] }, data1: 50 },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
Module {
  id: '.',
  path: '/root',
  exports: {
    messenger: { showMessage: [Function: showMessage] },
    data1: 50,
    data2: 100
  },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
Module {
  id: '.',
  path: '/root',
  exports: { value2: 100, function1: [Function: function1] },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
{
  messenger: { showMessage: [Function: showMessage] },
  data1: 50,
  data2: 100
}
i am a function
Module {
  id: '.',
  path: '/root',
  exports: { value2: 100, function1: [Function: function1] },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ],
  anything: { value1: 50 },
  whatever: { value2: 100 }
}
50
100
```

## Tracing code versus output

### Part 1

Code:
- Function `function1` is declared, but not used yet
- Function `showMessage` is exported and used with `exports.messenger.showMessage('this is a message')`

```js
const function1 = function() {
  console.log('i am a function')
}
exports.messenger = {
  showMessage: function(msg) {
    console.log(msg)
  },
}
exports.messenger.showMessage('this is a message')
console.log(module)
```

Output:
- Calling `exports.messenger.showMessage('this is a message')` prints out the message `this is a message`
- Printing `module` shows the exported `messenger` object with `showMessage` function

```js
this is a message
Module {
  id: '.',
  path: '/root',
  exports: { messenger: { showMessage: [Function: showMessage] } },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
```

### Part 2

Code:
- Exports a variable `data1` and prints the content of `module`
- Then, exports a variable `data2` and prints the content of `module` after

```js
var value1 = 50
exports.data1 = value1
console.log(module)
var value2 = 100
exports.data2 = value2
console.log(module)
```

Output:
- ☝️ Node.js has a sequential behaviour: `data1` and `data2` is added to `exports` one after another

```js
Module {
  id: '.',
  path: '/root',
  exports: { messenger: { showMessage: [Function: showMessage] }, data1: 50 },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
Module {
  id: '.',
  path: '/root',
  exports: {
    messenger: { showMessage: [Function: showMessage] },
    data1: 50,
    data2: 100
  },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
```

### Part 3

Code:
- Assigns `value2` and `function1` declared earlier to `module.exports`
- Prints `module` and `exports`
- Calls `function1` with `test` string as argument

```js
module.exports = {value2,function1}
console.log(module)
console.log(exports)
module.exports.function1('test')
```

Output:
- Printing `module` shows the updated content from `module.exports`, but printing `exports` still have the same content from earlier
- ☝️ When uninitialized, `module.exports` points to `exports` and hence, has the content from `exports`
  - When `module.exports` is assigned explicitly, it no longer points to `exports` → `module.exports` and `exports` are then separate objects with their own contents
- Function `function1` can be called with `module.exports.function1(...)` and the argument `test` is ignored since the function doesn't use it

```js
Module {
  id: '.',
  path: '/root',
  exports: { value2: 100, function1: [Function: function1] },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ]
}
{
  messenger: { showMessage: [Function: showMessage] },
  data1: 50,
  data2: 100
}
i am a function
```

### Part 4

Code:
- Assigns more modules `anything` and `whatever` with values `value1` and `value2` respectively
- Prints out content of `module` and values from modules `anything` and `whatever`

```js
module.anything = {value1}
module.whatever = {value2}
console.log(module)
console.log(module.anything.value1)
console.log(module.whatever.value2)
```

Output:
- Printing `module` shows content of modules `exports`, `anything` and `whatever`
- Values from modules `anything` and `whatever` can be referenced using `module.anything.value1` and `module.anything.value2`

```js
Module {
  id: '.',
  path: '/root',
  exports: { value2: 100, function1: [Function: function1] },
  filename: '/root/testexport.js',
  loaded: false,
  children: [],
  paths: [ '/root/node_modules', '/node_modules' ],
  anything: { value1: 50 },
  whatever: { value2: 100 }
}
50
100
```

## Another example

#### display.js

```js
const messenger = function (message) {
  console.log(message)
}
const prefix = 'My name is:'
module.exports = {prefix,messenger}
```

#### name.js

```js
module.exports = function (firstName,lastName) {
  this.firstName = firstName
  this.lastName = lastName
  this.fullName = function () { 
    return this.firstName + ' ' + this.lastName
  }
}
```

#### app.js

Export function as a class

```js
const showMessage = require('./display.js')
const getName = require('./name.js')
const myName = new getName('James', 'Bond')
showMessage.messenger(showMessage.prefix)
showMessage.messenger(myName.fullName())
```

#### Output

```console
My name is:
James Bond
```
