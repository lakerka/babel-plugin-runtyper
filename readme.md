# Runtyper
[![Build Status](https://travis-ci.org/vitalets/babel-plugin-runtyper.svg?branch=master)](https://travis-ci.org/vitalets/babel-plugin-runtyper)
[![npm](https://img.shields.io/npm/v/babel-plugin-runtyper.svg)](https://www.npmjs.com/package/babel-plugin-runtyper)
[![license](https://img.shields.io/npm/l/babel-plugin-runtyper.svg)](https://www.npmjs.com/package/babel-plugin-runtyper)
[![Twitter](https://img.shields.io/twitter/url/https/github.com/vitalets/babel-plugin-runtyper.svg?style=social)](https://twitter.com/intent/tweet?url=https%3A%2F%2Fgithub.com%2Fvitalets%2Fbabel-plugin-runtyper&text=Runtime%20type%20checker%20for%20JavaScript%20without%20annotations%3A&hashtags=javascript%2Ctypechecking%2Cbabelplugin)

> [Babel](https://babeljs.io) plugin for runtime type-checking in JavaScript

Runtyper warns you about silent type-mismatch operations in your JavaScript code.
Include it into your *development builds* and it will notify your in runtime when it detects weirdness. No type-annotations needed.

## Example
Imagine you have comparison like `x === y` and in runtime values are `x = 1`, `y = "1"`.
When executed you will silently get unexpected `false` result due to missing type conversion.  
After applying Runtyper it will show warning in console as soon as such situation occur:

![Strict compare warning example](https://cloud.githubusercontent.com/assets/1473072/24467786/8b531758-14be-11e7-80da-32de20e04d38.png)

or you can configure it to throw errors:

![Strict compare error example](https://cloud.githubusercontent.com/assets/1473072/24371480/926108e8-1333-11e7-8e17-0223ed0c21ad.png)


## Install
1. Ensure you have [babel installed](https://babeljs.io/docs/setup/)
2. Run in terminal:
  ```bash
  npm install --save-dev babel-plugin-runtyper
  ```

## Usage 
1. No changes to your existing codebase needed!
2. Add `babel-plugin-runtyper` to babel config in *development builds*.  
    For example, directly in terminal: 
    ```bash
    babel in.js -o out.js --plugins=babel-plugin-runtyper --source-maps
    ```
    or in `package.json` scripts:
    ```json
    "scripts": {
      "babel-dev": "babel in.js -o out.js --plugins=babel-plugin-runtyper --source-maps"
    }
    ``` 
    or with [webpack babel-loader](https://webpack.js.org/loaders/babel-loader/):
    ```js
    {
        test: /\.js$/,
        exclude: /(node_modules)/,
        loader: 'babel-loader',
        query: {
            plugins: [
                ['babel-plugin-runtyper', {enabled: process.env.NODE_ENV !== 'production'}]
            ]
        }
    }
    ```
    > Please note to run webpack as `NODE_ENV='production' webpack -p` (see [#2537](https://github.com/webpack/webpack/issues/2537))

3. Enable source-maps to see original place of error:
  * In Chrome set `Enable JavaScript source maps` in devtools settings
  * In Firefox please follow [this instruction](https://developer.mozilla.org/en-US/docs/Tools/Web_Console/Console_messages#Source_maps)
  * In Node.js use [source-map-support](https://github.com/evanw/node-source-map-support) package:
      ```js
      require('source-map-support').install();
      ```

> Tip: checkout [examples](examples) directory in this repo to see browser and Node.js demos


## How it works
Basically it wraps all type-important operations into function that additionally performs type checking.   
For example, before: 
```js
if (x === y) { ... }
```
After (simplified):
```js
if (strictEqual(x, y)) { ... }

function strictEqual(a, b) {
  if (typeof a !== typeof b) {
    console.warn('Strict compare of different types: ' + (typeof a) + ' === ' + (typeof b));
  }
  return a === b;
}
```

## Supported operators
* **Strict equality (`===, !==`)**  
  Protects you from:
  ```js
  1 === '1'        // false
  1 === [1]        // false
  1 === new Date() // false
  ...
  ```

* **Addition (`+`)**  
  Protects you from:
  ```js
  '1' + null      // '1null'
  '1' + undefined // '1undefined'
  '1' + NaN       // '1NaN'
  1 + NaN         // NaN
  1 + {}          // '1[object Object]'
  ...
  ```
  
* **Arithmetic (`-, *, /, %`)**  
  Protects you from:
  ```js
  1 - '1px'     // NaN
  1 - undefined // NaN
  1 * null      // 0
  1 * {}        // NaN
  ...
  ```
    
* **Relational (`>, >=, <, <=`)**  
  Protects you from:
  ```js
  2 < '11'        // false (but '1' < '11' is true)
  1 < null        // false
  [1] < {}        // true
  2 < [11, null]  // false (but 2 < [11] is true)
  ...
  ```

## Configuration
To configure plugin pass it to babel as array:
```js
  plugins: [
      ['babel-plugin-runtyper', options]
  ]
```
**Options**

| Name                         | Default  | Values                                   | Description                                        |
|------------------------------|----------|------------------------------------------|----------------------------------------------------|
| `enabled`                    | `true`   | `true`, `false`                          | Is plugin enabled                                  |
| `warnLevel`                  | `"warn"` | `"info"`, `"warn"`, `"error"`, `"break"` | How warnings are displayed                         |
| `implicitAddStringNumber`    | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x + y` where `x, y` are `(string, number)` |
| `implicitEqualNull`        | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x === y` where `x` or `y` is `null`        |
| `implicitEqualUndefined`   | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x === y` where `x` or `y` is `undefined`   |
| `explicitAddEmptyString`     | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x + ""` where `x` is not `string`          |
| `explicitEqualTrue`        | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x === true` where `x` is not `boolean`     |
| `explicitEqualFalse`       | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x === false` where `x` is not `boolean`    |
| `implicitEqualCustomTypes` | `"deny"` | `"allow"`, `"deny"`                      | Rule for `x === y` where `x instanceof Custom1` and `y instanceof Custom2` |

**Warning levels description**
 
 * `info` - notification via `console.info` without stacktrace
 * `warn` - notification via `console.warn` with stacktrace
 * `error` - notification via `console.error` with stacktrace
 * `break` - notification via throwing error and breaking execution

## Run on existing project
You can try Runtyper on your existing project with the softest configuration to see how it is going in runtime.  
Config:
```js
{
    enabled: true,
    implicitAddStringNumber: 'allow',
    implicitEqualNull: 'allow',
    implicitEqualUndefined: 'allow',
    explicitAddEmptyString: 'allow',
    explicitEqualTrue: 'allow',
    explicitEqualFalse: 'allow',
    implicitEqualCustomTypes: 'allow'
}
```
Maybe you will not get any warnings, maybe you will get something like this:
```
Error: Strict equal of different types: -1 (number) === "" (string)
Error: Strict equal of different types: 2 (number) === "" (string)
Error: Strict equal of different types: 56.9364 (number) === "" (string)
Error: Strict equal of different types: -0.0869 (number) === "" (string)
Error: Numeric operation with non-numeric value: null / 60 (number)
Error: Numeric operation with non-numeric value: "2017-03-29T00:00:00... (Date) / 1000 (number)
Error: Numeric operation with non-numeric value: "2017-03-29T00:00:00... (Date) / 1000 (number)
...
```

## Compare to static tools
Static code analysis is also the way to perform type checking in your application. 
For example, there is Facebook's [Flow](https://flowtype.org) project.
It can be used together with Runtyper to detect errors on both pre-runtime and runtime stages.

Yet, static tools require extra efforts for:
* Writing type-annotations (may be annoying)
* Integration with third-party libraries (as their API should be also annotated)
* Processing external events from user / server (different and changing formats)
* Involving new members (who is not familiar with typed JavaScript)

Let's take an example from [Flow's get started page](https://flowtype.org/en/docs/getting-started/):
```js
// @flow
function square(n) {
  return n * n; // Error!
}

square("2");
```

But if `square()` is used to handle user's input from textfield - error will not be found: 
```js
// @flow
function square(n) {
  return n * n; // no Error, until you annotate `event.target.value`
}

window.document.getElementById('username').addEventListener('change', function (event) {
  square(event.target.value);
});
```

Runtyper allows to catch such cases in runtime:

![Textfield error](https://cloud.githubusercontent.com/assets/1473072/24371601/f8d10ab0-1333-11e7-8baf-6b6501accd29.png)

So consider both approaches to make your applications more robust and reliable.

## FAQ
1. **Why I get error for [template literals](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Template_literals) like `${name}${index}`?**  
   Likely you are using [babel-preset-es2015](https://babeljs.io/docs/plugins/preset-es2015/) that transforms template literals into concatenation via `+`.
   And you get `(string) + (number)`. You can fix it in several ways:
    * set plugin option `implicitAddStringNumber: "allow"`
    * add explicit conversion: `${name}${String(index)}`
    * consider using [babel-preset-env](https://babeljs.io/docs/plugins/preset-env/) as template literals are widely supported natively

2. **Why explicit comparings like `x === null` or `x === undefined` are not warned?**  
  Because when you explicitly write `(variable) === null` you assume that variable *can be* `null`. 
  Another thing is comparing two variables `x === y`. Here it depends on your app desing: if `x` and `y` are *not nullable*,
  you may want to get warnings, otherwise you can set plugin options `implicitCompareNull` and `implicitCompareUndefined` to `"allow"`.

3. **Are non-strict equal operators `==` and `!=` being checked?**  
  Nope. Just quote Douglas Crockford from [JavaScript, the Good Parts](http://oreilly.com/catalog/9780596517748/):
    > JavaScript has two sets of equality operators: `===` and `!==`, and their evil twins `==` and `!=`. 
      The good ones work the way you would expect. If the two operands are of the same type and have the same value, 
      then `===` produces `true` and `!==` produces `false`. The evil twins do the right thing when the operands
      are of the same type, but if they are of different types, they attempt to coerce the values. 
      The rules by which they do that are *complicated* and *unmemorable*.  
      These are some of the interesting cases:

      ```js 
      '' == '0'           // false
      0 == ''             // true
      0 == '0'            // true

      false == 'false'    // false
      false == '0'        // true

      false == undefined  // false
      false == null       // false
      null == undefined   // true

      ' \t\r\n ' == 0     // true
      ```

    >  **The lack of transitivity is alarming. My advice is to never use `==` and `!=`. Instead, always use `===` and `!==`.**
    
    You can set ESLint [eqeqeq rule](http://eslint.org/docs/rules/eqeqeq) and forget about `==` once and for all.
    
> In case of other questions or ideas please feel free to [file new issue](https://github.com/vitalets/babel-plugin-runtyper/issues/new).

## License
MIT @ [Vitaliy Potapov](https://github.com/vitalets)
