# Ref() Class

## Status

Champion(s): *TBC*

Author(s): *Jason Ford*

Stage: 0

## About

- `Ref()` is a minimal wrapper constructor that can be initialized with any type of value, defaulting to `null` if nothing is provided. It is non-primitive by default and so has *pass-by-reference* behavior by default.
- A `Ref` instance is largely transparent; you work with it like you would the value you initialized it with. This includes all syntax variants, like `n++` and so on. It can be thought of as wrapping the value in an `Array(1)` -- yet all operations apply to the value *inside* instead.
- This is effectively a way to promote primitives values to a non-primitive class level, with some nice functionality for non-primitives as well.
- Like `Symbol` and `Object`, the global `Ref` constructor exposes useful methods, outlined below.

## Documentation

### Initializing
Wrap any data type in the `Ref` constructor; defaulting to `null` if nothing is passed.
```js
let refA = new Ref('a');
refA += 'b'; // string within refA is now 'ab'
```

Optionally, a `Symbol` can be passed as the second argument to the constructor, acting as a *custom namespace*, overriding the default global namespace.
```js
let ctx 	= Symbol('some-context');
let refA 	= new Ref('a', ctx);
```

Optionally, an `Object` can be passed as the second argument to the constructor, where namespace and set/get traps can be configured. This can allow something like `Signals` to be constructed.
```js
let refA = new Ref('a', {
	namespace: Symbol('some-context'),
	set( value ){
		// whatever is returned becomes the new value
		// could call other code here, enabling a Signal/Event paradigm
	},
	get( value ){
		// value is the current value, can return it or anything
		// The return from here does not affect the internal value
	}
});
```

Passing a `Ref` instance back into the constructor creates a new `Ref` instance with the same internal value. If that value is a primitive, it is copied. This matches the behavior of `Set`.
```js
let refA2 = new Ref(refA);
// refA2<instance> === refA<instance> => false
// Ref.get(refA) === Ref.get(refA2) => true because contents were returned then compared
```

### Methods
Since a `Ref` instance is intended to be worked with exactly like its value, the constructor is used to access `Ref` specific methods, just like `Object` and its methods.

#### Ref.is(<Ref instance\>, <Ref instance\>)
Regular equality on `Ref` instances won't work, since the internal value would be used instead. This method enables comparing if two `Ref` instances are the same instance or not.
```js
let refA 	= new Ref('a');
let refAb 	= refA;
let refA2	= new Ref('a');
console.log( Ref.is( refA, refAb ), refA === refAb ); // true, true
console.log( Ref.is( refA, refA2 ), refA === refA2 ); // false, true
```

#### Ref.set(<Ref instance\>, <new value\>)
Replaces the internal value of the passed `Ref` instance. Novel if the value is a primitive, as it allows one to effectively update a primitive value wherever it was referenced.

If `new value` is a `Ref`, the value inside is used instead. This is to avoid nested Refs. If the goal was to merge/replace Refs, use `Ref.replace`.

*Method name subject to discussion; using 'get/set' to match Proxy for its trap syntax*

```js
let data1 = [ refA ];
let data2 = { 'text':refA };
Ref.set(refA,'b');
// 'b' === data1[0] === data2.text => true
```

#### Ref.get(<Ref instance\>)
Explicit way to get the internal value of a `Ref`. If the value is primitive, its a copy.

Usually not necessary, the whole point of `Ref` is to assign it to variables, properties, pass as arguments, etc to gain the various advantages. Would only use `Ref.get()` if you want to 'export' the value so it's not a `Ref` anymore.

```js
let refA = new Ref('a');  // Ref<'a'>
let strA = Ref.get(refA); // String<'a'>
```

#### Ref.for(<any\>,<namespace: Symbol\>)
An equivalent to `Symbol.for()` to make it easy to swap anything to a `Ref` instance that already exists or create a new one. For non-primitives the usual object reference is used, like in `Set`/`Map`.

This method can utilize a namespace to control which instance is returned and avoid issues caused by having only a single global registry for everything.

```js
let ctx1	= Symbol('some-context');
let refA 	= new Ref('a', ctx1);
let refA2 	= Ref.for('a', ctx1);
// refA === refA2; no different than refA2 = refA;

let ctx2	= Symbol('some-context');
let refA3 	= Ref.for('a', ctx2);
// refA === refA3 only because equality is checking their values, which are primitives
// Ref.set(refA2,'A') would not affect refA3
```

#### Ref.replace(<Ref instance\>, <Ref instance\>)
All references to the second Ref instance are replaced with the first Ref instance, regardless of values.
```js
let refA 	= new Ref('a');
let refB 	= new Ref('b');
let arrRefs = [ refA, refB ];
let setRefs = new Set([ refA, refB ]);
Ref.replace( refA, refB );
console.log( arrRefs ); // [ <refA>, <refA> ] => [ 'a', 'a' ]
console.log( setRefs ); // { <refA> } => { 'a' }
```

## Use Cases

### Synchronicity
Particularly for primitive values, being able to natively link a value in multiple data structures would avoid workarounds like wrapping the value in an Array, or doing lookups.
```js
let version = Ref('1.1.1');
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

### Pointer-Like/Self-Modifying
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
// again, presumably the variables exist in a very different scope as replaceArr()
```

## Implementations

### Polyfill/transpiler implementations
*TBC -- Not really sure how to do this, let me know!*

### Notes
Presumably `valueOf`, `toString`, `toJSON`, and other internal mechanisms might have to be tweaked to detect `Ref` and get its internal value.

## Q&A

**Q**: **Why locally-transparent syntax, why not myRef.internalValue=newValue or myRef.update(newValue)**

**A**: Such syntax would hide the `Ref` data type behind regular OOP syntax, and since `Ref` is basically a 100% side-effect class, it needs to be clear that the line is about a `Ref` and 'this line has side-effects'.

______________________________
## EXTRA
### HTMLElement + Ref = linkref keyword

Some properties of HTML elements have a coerce-to-string behavior, for example `textContent` and `innerHTML`. If the RHS of the assignment is entirely a `Ref` instance (no operations), the engine could note this and 'replay' the assignment line whenever the `Ref`'s value changes.

This would be opt-in via a new leading keyword `linkref`. This would go a long way to having a native MVM mechanism. Consider the example below:

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