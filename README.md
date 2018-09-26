# esnext-compose-operator-proposal
Javascript composition operator proposal.


## Contents
- [Reasoning ](#reasoning)
- [Advantages](#advantages)
- [Syntax](#syntax)
- [Usage Examples](#usage-examples)
- [Formal Definition](#formal-definition)
- [Prior Art](#prior-art)
- [Proposal License](#proposal-license)

## Reasoning 
Similar to "pipeline-operator-proposal" yet executes function
composition in the same order/direction javascript already performs it;  
E.g. from right to left:
```
a(b(c(d()))) === a <| b <| c <| d
```

This proposal proposes keeping with javascript's (and many -a- "language"'s) accepted "right-to-left" function composition format.

## Advantages
Some advantages with continuing to do function composition in the 
same order that javascript already performs it (even when adding a new operator) are:
1.  No need to mix and match the order in which function composition occurs/is-defined (in code) (it happens in one direction already unless explicitly defined by user (via `reverse`, `reduce` or other already existing language constructs)
2.  Problems that were already solved with composing functions in the 'right-to-left' direction don't need to be solved again and problems arising from switching directions are never born.
3.  We get to inherit the wealth of knowledge from other programming languages that do function composition/application
  in the same direction, already, (haskell, c++, scala, etc.).

## Syntax
 
 ```
 statement <| statement <| statement?
 ```
 
 where `statement` is anything that yields a function (including a function itself) and last statement is either something that yields a function (including a function itself) or any other value.
 
 ```
 (statement) instanceof Function === true 

 // `statement?`
 [undefined, null, '', 0, false, () => undefined, 'et. al.'].every(x => x === (statement?)) === true
 ```
 
 Essential properties of 'compose' operator:
 
 ```
 compose(a, b, c) === x => a(b(c(x))) &&
 a <| b <| c === x => a(b(c(x))) // true
 
 a <| b <| === a(b()) // `true` // see next example for more on this
 ```
 
 The right-most `statement` in a 'compose' operator statement can optionally be 
 any value (will force resulting composed `function`'s execution if value is not a `function`):
 
 ```
 compose(a, b, c)(1) !== a <| b <| c <| 1 
 
 (a <| b <| c)() === a <| b <| c <| undefined
 
 (a <| b <| c)() === a <| b <| c <| 
 ```
 
 This should also be possible with this proposal:
 ```
     const eight =
         await Promise.resolve(x => x + x) <|
         await Promise.resolve(x => x + x) <|
         await Promise.resolve(x => x + x) <| 1
 ``` 
 In essence anything that yields a function can go on either the left, right, or both, sides of the 'compose' operator.

## Usage Examples
Here are some, more in-depth, example use cases using the "proposed" 'compose' operator.

#### Functional usage example 1

```
// Old way (with no library)
const someOp = namesAndPromisesList => Promise.all(
  namesAndPromisesList.map(([n, p] => p.then(x => [n, x]))
  )

// New way (point-free - No need to decalre incoming variable due to 
//  `<|` operator generating a new function, since we didn't pass in a 
//  'non' function value at the end of the pipe.
//  This new function will take any passed in args and pass them to the last 
//  function in the pipe hence kicking off the pipe execution,
import {map} from 'lodash/fp';  // import 'curried' `map` function 
 
// Transforms `Array<[String, Promise<Any>]>` to `Promise<[String, Any]>
const someOp = Promise.all <| map(([n, p]) => p.then(x => [n, x]))

// Much nicer!  And I didn't need to use `await` or generator functions within pipes, E.g., 
// `a <| await b <| await c` bad composition (io actions/promises shouldn't be 
//  composed with plain functions their values should be extracted and then be used in compositions. 
```  

#### Functional usage example 2:
Execute some transform functions on a 4x4 matrix.
Shows example usage using the old way (es6, es5) 
and the proposed, "composition" operator, way.  
```
const 
  // Create some funcs
  log = console.log.bind(console),
  peek = (...args) => (log(...args), args.pop()),
  id = x => x,        // For depicting our func bodies
  translateMat4 = id, // function body omitted for brevity 
  scaleMat4 =     id, // ""
  rotateMat4 =    id, // ""
  mat4 = (x, y, z, u) => ... // Lets say "produces a matrix of 4 components" (4x4 matrix)
  
  // Define a transformation
  transformMat4Old = m => translate(rotate(peek(skew(scale(m))))),
   
  // Define a transformation
  transformMat4 = translate <| rotate <| peek <| skew <| scale,
  // Note about `transformMat4`: With pipeline 'left-to-right' operator you have to 
  // Think about function composition differently an
  // and no extra thinking is required for wanting to 
  // create a new function from a 'right-to-left' pipeline. 
  
  shallowEqual = (a, b) => ..., // Body ommitted for brevity 
  assertr = b => a => a === b
;
```

```
  // Elsewhere in code
  true === assertr(true) <| 
    shallowEquals(
      transformMat4Old <| mat4(0, 0, 0, 0),
      transformMat4 <| mat4(0, 0, 0, 0)
    ); // `true`
  
  // Old style ...
  //  Via a multitude of ways, but usually either by putting results in variables and then comparing the two
  //  Or by having a boatload of parenthesis (lol we've all encountered 
  //    them at some point our development lives, lol)
  // Here's a bit more real world... (in some test suite)...
  describe ('#someTransform', () => {
    it ('Should do some transform', () => {
      const newMat4 = () => mat4(0.0, 0.0, 0.0, 0.0)
      expect(
        someOtherTransform(
          transformMat4Old(
            newMat4()
          )
        )
      ).toEqual(            // In `jest` performs `deep` 'equals' check
        someOtherTransform(
          peek(
            transformMat4(
              newMat4()
            )
          )
        )
      );
    });
  });
  
  // Notice that inorder to get the code readable it had to broken out into many lines 
  //  due to having jumbled paranthesis confusion when mushed into 4 or less lines
  //  (counting parenthesis etc.).
    
  // Proposed way (can now be written on fewer lines in a more composable format 
  //    (also with less "parenthesis" spaghetti, lol)
    describe ('#someTransform', () => {
      it ('Should do some transform', () => {
        expect().toEqual.apply(
          expect <| someOtherTransform <| transformMat4Old <| newMat4(),
          someOtherTransform <| peek <| transformMat4Old <| newMat4()
        );
      });
    });
```

#### Real world example
Json configuration combinator.

Takes something of the form:
```json
{
  "production": {},
  "development": {"inheritsFrom": "production"},
  "testing": {"inheritsFrom": "production"},
  "staging": {"inheritsFrom": "testing"}
}
```

Merges all the `inheritsFrom` keys of a given key (recursively) and merges
them down into one object then extracts the `keyToExtract` key from the resulting
object:

```
import {assignDeep, apply, compose, jsonClone, reverse} from 'fjl';

const toConfigOptions = ({
        config,
        onKey = 'development',
        inheritsFromKey = 'inheritsFrom',
        keyToExtract = 'options'
    }) => {
    const 
      configsToMerge = [],
      configKeysVisited = new Map()
      ;
    let key = onKey;
    
    // If no config return empty one
    if (!config) { return {}; }
    
    // Get each config inherited from
    while (key && !configKeysVisited.has(key)) {
        let found = config[key];
        if (!found) { key = null; break; }
        configKeysVisited.set(key, found);
        configsToMerge.push(found);
        key = found[inheritsFromKey];
    }
    configsToMerge.push({}); // Object to merge on
    return 
        jsonClone <| x => (x[keyToExtract] || {}) <|
        apply(assignDeep) <| reverse <| configsToMerge
    ;
};

export default toConfigOptions;
```

## Formal Definition
The 'compose' operator...
1.  Consists of a symbol containing two characters `<` and `|` next to each other with no space in between;  I.e., `<|`;
2.  Allows the user to create new functions/compositions by listing 2 or more functions separated by said operator.
3.  Will, in turn, execute a composition if the last entry in said composition is not a function.
4.  Will return a new function that will pass all incoming arguments to the right-most function in composition.
5.  Will execute each function in composition from right to left passing the result of functions on the right to functions at their immediate left.
  
```
const 
  add = a => b => a + b;
  log = console.log.bind(console)
;
   
log <| add(1) <| add(1) <| add(1) <| 2 // "5"
log <| add(':c') <| add(':b') <| 'a' // 'a:b:c'
``` 

6.  All rules for writing statements are allowed within function compositions so long
as such statements return/yield a function (except for last which can yield any values (as per next rule))
7.  All statements within a compose composition must yield a function unless that statement is the right most statement in a 'compose' composition in which case that statement is allowed to yield any value including a function. In the case that the last statement is not a `function` the compose composition will be executed immediately with the results of said statment passed in to resulting composition. 
8.  Left most expression must yield/be a function.

## Prior Art
- fjl/compose - https://functional-jslib.github.io/fjl/module-function.html#.compose 
- RamdaJs/compose - https://ramdajs.com/docs/#compose 
- Plain old javascript function composition.
- Haskell 'compose' operator: http://hackage.haskell.org/package/base-4.11.1.0/docs/Prelude.html#g:12
- Inline example:
```
const 

    compose = (...funcs) => x => 
        funcs.reduce((lastX, fn) => fn(lastX), x),
        
    op = x => x + x
    
;
    
// Proposed built-in
op <| op <| op <| op(1) === 16

// vs. (userland)
compose(op, op, op, op)(1) === 16 

// vs. (paranthesis counting)
op(op(op(op(1)))) === 16
    
```

## Proposal License
BSD-3-Clause
