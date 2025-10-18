# Ref()

## Status

Champion(s): *TBC*  
Author(s): *Jason Ford*  
Stage: 0

## Problems to Address

- The inability to pass primitives by reference (without wrapping)
```js
let a = 1;
let b = a; // copy of a
```

- The inability for a primitive binding to be *explicitly* changed from a different scope
```js
function f(n){
	// has no general/closure access to a, only the copy n
	// even if this definition was last, variable could be in an inaccessible scope
}
{
	let a = 1;
	f(a);
	// above passes a copy and f's code has no good way to change original x
	// f would have to return the new value and overwrite x here
}
```
In both cases above, you can easily work around the problem by using `a=[1]` and changing its content instead, but that approach would add 'silly' syntax overhead to all downstream code.

##### More Specific Examples of Problems
- Wasteful data duplication of primitives across object instances (common fields like units, category, etc)
- Costly loops used to mass-update values across object collections (might require conversion to iterable first)
- Sending entire objects downstream just so primitive properties can be updated (overloading arguments, exposing sensitive objects, etc)
- Hand-made events/functions to detect and propagate changes to primitives and/or replacements of non-primitives (libraries/frameworks try to address this)

## Proposal

- `Ref()`; a minimal *wrapper* constructor that is functionally transparent, ensuring *pass-by-reference* for any data type it's initialized with.
- It can be thought of as a 'single-value container' like wrapping something in `[](1)` — yet all operations would apply to the value *inside* instead.
- You work with a `Ref` instance exactly like you would the value given to it. Syntax variants like `n++`/`n+='z'` are technically shorthands; when expanded `Ref` mechanisms are used to mutate the value of a `Ref` instance.
- `Ref` is not just for primitives, its functionalities are useful for non-primitives as well.
- `valueOf`, `toString`, `toJSON`, and others would be forwarded to the internal value of the Ref instance.
- Like `Symbol`/`Object`/`Reflect`, the global `Ref` API exposes useful methods, outlined below.

## Why
Currently, without a pass-by-reference for primitives, **the programmer has no way to explicitly state their desired behavior — pass-by-value or pass-by-reference**. Maybe it can be deduced from the logic (code that reaches back and updates other places; they wrapped it in [], etc), but that is speculative.

With `Ref`, the programmer can explicitly state from the onset, "I want this to pass downstream as a reference", and at any point can use `Ref.get()` or `Ref(Ref)` to convey when they want to unlink and have a new instance.

**Ref allows a currently *ambiguous* programmer design choice to become a *clear* programmer design choice.**

---------

## Details

### Initializing
Wrap any data type in the `Ref` constructor; defaulting to `null` if nothing is passed.
```js
let refA = new Ref('a');
refA += 'b'; // string within refA is now 'ab'
```

### Arguments

The first argument will always be the value to wrap. The second argument can be:

#### Symbol
Optional; a `Symbol` acting as a *namespace* binding, overriding the default global namespace binding. This is largely for controlling lookups (see further down) not forming groups.
```js
let ctx 	= Symbol('some-context');
let refA 	= new Ref('a', ctx);
```

#### Config Object
Optional; an `Object`, where *namespace* and *set/get* traps can be configured. This can allow something like `Signals/Proxies` to be constructed at value-level instead of object-level.
```js
let refA = new Ref('a', {
	namespace: Symbol('some-context'),
	set( newValue ){
		// whatever is returned becomes the new value
		// could call other code here, enabling a Signal/Event paradigm
	},
	get( currentValue ){
		// value is the current value, can return it or anything
		// The return from here does *not* affect the internal value
	}
});
```

#### Ref Instance
Passing a `Ref` instance back into the constructor creates a new `Ref` instance with the same internal value. If that value is a primitive, it is copied. This matches the behavior of `Set` (new instance, copied p. contents, same np. contents).
```js
let refA2 = new Ref(refA);
// refA2<instance> === refA<instance> => false
// Ref.get(refA) === Ref.get(refA2) => true because contents were returned then compared
```

---------

## Methods
Since a `Ref` instance is intended to be worked with exactly like its value, the global `Ref` API is used to access specific methods, just like `Object`, `Symbol`, `Reflect`, etc.

### Ref.set(<Ref instance\>, <new value\>)
Replaces the internal value of the passed `Ref` instance; can replace any value with any value (no type-matching requirement). See [these specific examples](#pointer-like-self-modify) for what becomes possible with `Ref.set()`.

If `<new value>` is a `Ref`, the value inside is used instead. This is to avoid nested Refs, which I believe would be too error-prone to support. If the goal was to merge/replace Refs, use `Ref.replace`.

```js
let data1 = [ refA ];
let data2 = { 'text':refA };
Ref.set(refA,'b');
// 'b' === data1[0] === data2.text => true
```

### Ref.get(<Ref instance\>)
Returns the current internal value of a `Ref`; a copy if primitive, otherwise a reference to the non-primitive.

Main use is to 'export' the value so it's not a `Ref` anymore.

```js
let refA = new Ref('a');  // Ref<'a'>
let strA = Ref.get(refA); // String<'a'>
```

### Ref.is(<Ref instance\>, <Ref instance\>)
Regular equality on `Ref` instances won't work, since the internal values would be equated not the instances. This method enables comparing if two `Ref` instances are the same instance or not.
```js
let refA 	= new Ref('a');
let refAb 	= refA;
let refA2	= new Ref('a');
console.log( Ref.is( refA, refAb ), refA === refAb ); // true, true
console.log( Ref.is( refA, refA2 ), refA === refA2 ); // false, true
```

### Ref.namespace(<Ref instance\>,<namespace: Symbol\>)
Set/overwrite the namespace a Ref instance is associated with.

Throws an error if the combination already exists and is not itself (confusion scenario).

### Ref.replace(<Ref instance\>, <Ref instance\>)
All references to the first are replaced with the second, regardless of values. This may require Refs to be implemented in engines as 'double-pointers'.
```js
let refOld	= new Ref('o');
let refNew 	= new Ref('n');
let arrRefs = [ refOld, refNew ]; // [ <refOld>, <refNew> ]
let setRefs = new Set([ refOld, refNew ]); // { <refOld>, <refNew> }
Ref.replace( refOld, refNew );
console.log( arrRefs ); // [ <refOld>, <refNew> ]
console.log( setRefs ); // { <refOld> } * a retroactive side-effect; TBD *
```

### Ref.for(<any\>,<namespace: Symbol\>)
An equivalent to `Symbol.for()` to make it easy to swap anything to a `Ref` instance that already exists or create a new one. For non-primitives the usual object reference is used, like in `Set`/`Map`.

Pass a `Symbol` namespace specifier as the second argument to create/return a matching `Ref` instance. This avoids issues caused by having only a single global registry.

```js
let ctx1	= Symbol('some-context-1');
let refA 	= new Ref('a', ctx1);
let refA2 	= Ref.for('a', ctx1);
// refA === refA2; no different than refA2 = refA;

let ctx2	= Symbol('some-context-2');
let refA3 	= Ref.for('a', ctx2);
// refA === refA3 only because equality is checking their values, which are primitives
// Ref.set(refA2,'A') would not affect refA3
```

### ~~Ref.copy(<Ref instance\>)~~
A `copy` or `clone` method is not provided on purpose, as it would be misleading. Since the value can be a primitive or non-primitive, it could imply making a copy of a non-primitive, which is a non-trivial action.

Instead, pass the Ref instance back into the Ref constructor (`new Ref(<Ref>)`) as outlined [here](#passing-ref-as-value).

---------

## Refactoring Dangers
Using `Ref` requires some foresight, mainly to avoid the following scenario:

1. The binding `x` currently stores a regular primitive at *point A*
1. At *point A*, you wrap `x` with `Ref` so you can use `Ref.set()` downstream at *point Z*
1. Somewhere between *point A to Z*, the code was passing the value (as-copy) to other bindings and mutating them
1. Those passes are now as-reference and each mutation along the way affects all downstream paths, leading to an unclear present value and possible errors

Above is not a dealbreaker, as changing objects in an existing codebase is a similar story of danger and careful refactoring. The solution would be to use `Ref.get()` to export the value as a non-Ref in each upstream place where a copy seems to be wanted, likely wherever passing occurs.

---------

## Use Cases

### Shortcuts to Primitives in Objects
It's easy to have complex data structures with deeply nested values. `Ref` allows you to use a 'shortcut' mapping paradigm for those deep values to expose a convenient get/set API for downstream use, without needing accessor keys. This allows the data structure to change without having to worry about updating the relevant accessor paths throughout the codebase.
```js
function someObjMaker(){
	let v3 			= Ref(0)
	let v4 			= Ref('abc')
	let v5 			= Ref([0,1,2])
	let deepObject 	= {
						k1:{
							k2:{
								k3: v3,
							},
							k4: v4,
						},
						k5: v5
					};
	return { deepObject, shortcuts: { k3, k4, k5 } };
}
let O = someObjMaker();
Ref.set(O.shortcuts.k3,1); // old inflexible way => O.k1.k2.k3 = '1';
Ref.set(O.shortcuts.k4,'ABC');
Ref.set(O.shortcuts.k5,[3,4,5]);
```
Recall that since `Ref` is pass-by-reference, updating one binding updates all bindings to that value (like changing the contents of a `[]`).

### Synchronicity
Particularly for primitive values, being able to natively link a value in multiple data structures would avoid workarounds like wrapping the value in an Array, or doing lookups.
```js
let version = new Ref('1.1.1');
let display = { version, title, ... };
let package = { version, dependencies, ... };

Ref.set(version,'1.1.2');
```

### Data Deduplication
It is common to create or end up with many copies of a primitive value in memory, especially when working with APIs that return DB-like records. I’m not sure if browsers do some kind of de-duplication behind the scenes, but `Ref.for` would give the programmer an easy way to do that.
```js
const records = [
	{ category:'ABC123', productId:123 },
	{ category:'ABC123', productId:456 },
	{ category:'ABC123', productId:789 },
	...
];

// when receiving data from APIs, there tends to be repeated values. Use Ref.for() to swap to single instance
records.forEach((record) => record.category = Ref.for(record.category) );
```

### Pointer-Like - Self Modify
Languages with pointers have a mechanical separation between reference and value that allows for repointing and single-sourcing. Javascript only has a clumsy way to do this (Array(1)) that incurs extra markup (a[0], a.get(0), etc). A `Ref` class would add some attractive pointer-like features, example below:
```js
// pointer-like - for primitives

let refText		= new Ref('some text');
let refId 		= new Ref(12345);
let myObj 		= { refText, refId }; // usually, this would get copies of the value

function format( caseMethodName, refText, refId ){
	// presumably this function is far away from the vars above, different scope entirely
	let recased 	= refText[caseMethodName]();
	let formatted 	= '#'+(refId.toString().padStart(10,'0'));
	Ref.set( refText, recased );
	Ref.set( refId, formatted );
	// by using Ref, properties on myObj were also 'updated' without needing to do anything!
}

format( 'toUpperCase', refText, refId );
// didn't have to return/re-assign/pass myObj in order to 'update' myObj!

console.log( refText, refId ); // "SOME TEXT", "#0000012345"
```

### Pointer-Like - Swap Object References
Giving `Ref` a non-primitive mainly gives you the ability to replace all bindings to that non-primitive with an entirely different value. This obsoletes the trick of emptying then re-filling an object just to preserve its bindings. This is possible because `Ref` is itself a 'single-value container' and all operations apply to the value inside at `get`/`set` time.
```js
// pointer-like - for objects

let myList 		= new Ref([1]);
let myListAgain = myList;

function replaceArr( someList ){
	let newListEntirely = [2]; // redundant to be a Ref() due to next line
	Ref.set( someList, newListEntirely );
}

replaceArr( myList );
// myList and myListAgain are now refs to newListEntirely ([2]) without having to return newListEntirely and re-assign myList/myListAgain to it
// presumably the variables exist in a very different scope as replaceArr()
```

---------

## Polyfill/Transpiler
*TBC -- Not really sure how to do this, let me know!*

---------

## Q&A
**Q**: **Why transparent syntax, why not `myRef.internalValue = newValue` or `myRef.update( newValue )`**  
**A**: Such syntax would hide the `Ref` nature behind regular OOP syntax, and since `Ref` is basically 100% side-effect, it needs to be clear when the line of code is about a `Ref` as a way to say 'there will be side-effects'.

---------

## Special Treatments

Since `Ref` is a 'single-value container', it may be suitable for some special treatments, like how `Symbols` was deemed suitable to be used as keys in `Map`.

### HTMLElement

Consider the `HTMLElement` properties `textContent` and `innerHTML`; they have a coerce-to-string behavior on assignment. The special treatment would occur when a `Ref` is assigned -- the engine would still get and coerce the Ref's internal value to string and apply that to the actual DOM, but it would bind the assigned `Ref` instance to the property as-is, rather than the coerced-to-string copy that it currently does. This effectively means `myElm.textContent = myRef; Ref.is(myElm.textContent, myRef) => true;`.

### Why
To create a 'live' connection between a value and the element's property without extra wiring, as a way to have a more elegant native MVM mechanism. There are 2 possible designs:

**Innate Behavior of HTMLElement + Ref**  
Opt-in by default; the connection happens automatically when assigning a `Ref` to an HTMLElement's properties. When the Ref's internal value changes, the HTMLElement's property is re-generated. You opt-out by using `Ref.get()` when doing the initial assignment.

**Keyword `linkref`**  
Opt-in by adding a new leading keyword `linkref` at the start of the assignment line.

**Example**  
Consider the examples below demonstrating the live connection. The only difference is whether `linkref` is considered necessary to 'turn on' this behavior:

```js
// ** when code changes the value **
let counterDisplay 	= new Ref('');
let counter 		= new Ref(10, {
						set: function(value){
							Ref.set( counterDisplay, '#'+value );
							return value;
						}
					});

linkref elementCountdown.textContent = counterDisplay;
// linkref tells engine to link this assignment line to the Ref on the RHS (RHS must be a Ref to use linkref keyword)

counter++;
// the set handler is triggered and it updates the display version of this counter
// when counterDisplay's internal value changes due to Ref.set, the engine 'replays' any linkref assignment lines associated with it -- so textContent ends up updated as well.
// This avoids having to give the set trap a reference to elementCountdown or some ui-updating function
```
```js
// ** when ui/DOM changes the value **
let inputName = new Ref('');

linkref inputElement.value = inputName;
// user enters a value into the input element...

console.log( inputName );
// will print whatever inputElement.value is; no need for inputElement.oninput to update inputName
// engine essentially flipped the LHS/RHS of the linkref line and replayed it when element's value changed
```