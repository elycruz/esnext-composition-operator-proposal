# esnext-composition-operator-proposal
Javascript composition operator proposal (W.I.P.).

## Contents
- Reasoning
- Usage Examples
- Formal definition of proposal
- Resources

## Reasoning 
Similar to "pipeline-operator-proposal" yet executes function
composition in the same direction javascript already performs it;  E.g. from right to left:
```
a( b( c )) === a <| b <| c
```

This proposal proposes keeping with javascript's (and many a language's) already 
implemented/accepted composition direction for "functions".

**Note**: 
This proposal only deals with "functions".  For promises
 operators were already introduced:
 ```
 async/await
 Promise.prototype.then
 Promise.prototype.catch
 Promise.all
 etc.
 ```
 and for generators operators were also introduced `for of {}` comprehensions (A.K.A. Array Comprehensions).
For all other complex use cases of promises and functions consider the following snippet and different programming 
paradigms (like functional programming for instance) for guidance on dealing with promises "without" having to add 
a new language feature to help take care of them (no pun/jab intended at javascript engine implementers/folks etc.).
 
```
// Old way (with no library)
const someOp = namesAndPromisesList => Promise.all(
  namesAndPromisesList.map(([n, p] => p.then(x => [n, x]))
  )

// New way (point-free - No need to decalre incoming variable due to `<|` operator generating a 
//  new function, since we didn't pass in a 'non' function value at the end of the pipe.
//  This new function will take any passed in args and pass them to the last 
//  function in the pipe hence kicking off the pipe execution,
import {map} from 'lodash/fp';  // import 'curried' `map` function 
 
// Transforms `Array<[String, Promise<Any>]>` to `Promise<[String, Any]>
const someOp = Promise.all <| map(([n, p]) => p.then(x => [n, x]))

// Much nicer!  And I didn't need to use `await` or generator functions within pipes, E.g., 
// `a <| await b <| await c` bad composition (io actions/promises shouldn't be 
//  composed with plain functions their values should be extracted and then be used in compositions. 
```  

Note, there are also advantages with continuing to do function composition in the 
same direction javascript already does it (even when adding a new operator):
1.  No need to mix and match the way function composition occurs/is-defined (in code) (it happens in one direction
  unless explicitly defined by user (by means already existing in the language (reduce/reverse) not some new language
  feature)).
2.  Problems that were already solved with composing functions in the 'right-to-left' direction don't 
  need to be solved again.
3.  Inheriting the wealth of knowledge from other programming languages that do function composition/application
  in the same way/direction (haskell, scala, etc.).

## Examples
Here are some examples showcasing the usages of the "proposed" composition operator.

#### Composing functions into a new one:
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
  //  Or by having a boatload of parenthesis (lol we've all encountered them at some point our development lives, lol)
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
  //  due to having jumbled paranthesis confusion when mushed into 4 or less lines(counting parenthesis etc.).
    
  // Proposed way (can now be written on fewer lines in a more composable format (also with less "parenthesis" spaghetti, lol)
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
    return jsonClone <| 
      x => x[keyToExtract] || {} <|
      apply(assignDeep) <|
      reverse <|
      configsToMerge
    ;
};

export default toConfigOptions;
```

## Definitions
For our purposes:
- `function composition` - The act of composing functions together. 
- `pipe composition` - The result of function composition with composition operator.  

## Formal definition of the function composition operator
The composition operator...
0.  Consists of a symbol containing two characters `<` and `|` next to each other with no space in between;  I.e., `<|`.
1.  Allows the user to create new functions/compositions by listing 2 or more functions separated by said operator.
2.  Will, in turn, execute a composition if the last entry in said composition is not a function.
3.  Will return a function/composition that will pass all incoming arguments to right-most function in pipe's function list.
4.  Will return the left-most's function's result at end of execution.  
```
const 
  add = a => b => a + b;
  log = console.log.bind(console)
;
   
log <| add(1) <| add(1) <| add(1) <| 2 // "5"
``` 

5.  All rules allowed within params list should be allowed within pipes (though unless expression is not the last
in expression list (list separated by `<|`) the resulting function should throw an error since it composition operator
can only compose functions together with an optional, final, value at the end of pipe definition).
