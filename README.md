# ParallelJavaScript



## Goals

The goal of this proposal is to enable data-parallelism in web applications. Browser applications and in particular EcmaScript often need to leverage all available computing resources to provide the best possible user experience. Today web applications do not take full advantage of ever increasing parallel client hardware due to the lack of appropriate programming models. This proposal puts the parallel computing power of client’s hardware into the hands of the web developer while staying within the safe and secure boundaries of the familiar EcmaScript programming paradigm. It gently extends EcmaScript with simple deterministic data-parallel constructs that enable runtime translation to a low-level hardware abstraction layer. By leveraging multiple CPU cores and vector instructions, Parallel EcmaScript achieves significant speedup over sequential EcmaScript.

### Design
-----

The design of Parallel EcmaScript (aka RiverTrail) adds methods with parallel semantics to Array and Typed Object, see typed objects, prototypes. These methods take as an argument an elemental function which typically returns a single data element. The methods are designed to enable concurrent execution of the elemental function.

We have chosen a set of parallel constructs that we feel is minimal and upon which other data parallel constructs can be built. For example sum would be implemented using the reducePar construct while prefix sum would be implemented using the scanPar construct. We anticipate useful libraries and infrastructure being built upon these constructs. This approach enables a “do few things well” implementation strategy while ensuring the composability needed to build other abstractions.

#### Arrays and Typed Objects
-----

Array types are extended instead of introducing a new data type as we did in previous proposals. The Array types will be used for one dimensional objects. Some interfaces, in particular WebGL and canvas, require the use of specific Typed Object formats like single precision float (float32), unsigned 8 bit Integer values (uint8) or structs. Treating collection of data as multi-dimension data structures is also useful. To support these use we have added the same functionality to the Typed Object types adapting the interfaces to reflect the increased robustness of Typed Object.

#### Elemental Functions
-----

Similar in spirit to the use of kernel functions and callback functions our approach makes use of elemental functions written in EcmaScript. Any EcmaScript function can be used as long as it has an appropriate signature and is temporally immutable, i.e., the function is side effect free and does not mutate global state. The system will determine if the elemental function can be optimized to take advantage of data parallel hardware. If the system can prove that the function is not temporally immutable it will throw an error. If the optimization is semantically possible but the system is unable to do the optimization it is free to execute it sequentially. Providing an elemental function that can be optimized is the responsibility of the developer though there are a few hints that will serve the programmer well. Recursion is an example of a typical EcmaScript construct that is likely to defeat optimization. Sequences of arithmetic operations on Arrays or Typed Objects that hold dense homogeneous data, e.g. Array objects that contain only floating point numbers, are good candidates for optimization.

#### Temporal Immutability
------

Temporal immutability means that whenever it is possible for two or more computations to run concurrently all objects visible to both computations are immutable until the computations complete. Since the main thread is dormant during parallel execution it is free to mutate objects without regard to temporal immutability. Since all current EcmaScript programs are single threaded they are temporally immutable and unaffected by this extension. Temporal immutability is weaker than declaring an object immutable since the object can be mutated by the main thread or if the object is local to a parallel thread.

#### Prototype Methods
-----

Array and Typed Object come with the following four data parallel methods that map Arrays to Arrays and Typed Object to Typed Object: mapPar, scanPar, filterPar, and scatterPar. When combined with elemental functions each of these methods create a freshly minted object. In addition to these we provide the method buildPar which uses an elemental function to create freshly minted Arrays and Typed Object. Array and Typed Object also includes a sixth data parallel method, reducePar, which typically produces a single value. We believe that with these few methods one can create a very robust and complete data parallel library that covers a large number of applications.

###### Example

This simple example creates a three element Array myPA. It then uses the prototype method mapPar and the elemental function val ⇒ val + 1 to create a freshly minted Array myPlusPA with each element in myPA incremented by 1.

```javascript
myPA = [1, 2, 3];                        // [1, 2, 3]
myPlusPA = myPA.mapPar(val => val + 1);  // [2, 3, 4]
```

## Terminology

- *Frame* indicates the iteration space.
- *Depth* indicaates how many dimensions to iterate over. As such it is how many dimensions are to be used in the frame.
- *Grain* refers to the value at each element in the frame - it is what is returned by an elemental function. An element is the grain found when the depth and the number of dimensions are the same. For example if one has a 4 dimensional structure and maps at the depth of 4 then the grain is the same as the element and the frame has 4 dimensions. If I map at the depth of 3 then the grain is a 1 dimensional vector of elements and the frame has 3 dimensions. Since Arrays are always one dimensional one can only iterate at the depth of 1, the frame has 1 dimension and the grain is the same as the elements. The terms grainType and elementType indicate corresponding types of a grain or an element.

## API


mapPar
-----

__Array__

```javascript
myArray.mapPar(elementalFunction, thisArg = undefined)
```

__Typed Object__

```javascript
myTO.mapPar(depth = 1, elementalFunction, thisArg = undefined)
```

##### Arguments
- *depth (optional)*: the number of dimensions to iterate over (as before), default is 1, the outermost dimension.
- *elementalFunction*: described below
- *thisArg*: If a *thisArg* parameter is provided, it will be used as the this value for each invocation of *elementalFunction*. If it is not provided, *undefined* is used instead.

##### Elemental Function
```javascript
function (element, index, source, outCursor) 
```
- *element*: The element from the source.
- *index*: The index in source where element is located as well as where the result will be placed.
- *source*: The source holding the elements.
- *outCursor*: Output cursor where results can be placed. If a non-undefined value is returned outCursor will be overwritten by the returned value. If *outCursor* is not specified the result of the function will be placed in mapPar’s result at *index*.

##### Returns

A freshly minted Array or Typed Object. Elements are the results of applying the elemental function to the elements in the original Array or Typed Object block coerced to the type grain of the source.

##### Throws

- *TypeError* when elementalFunction is not a function.
- *Error* when required by Typed Object conversion semantics.

##### Discussion

One functionally correct implementation of ***mapPar*** using Array would be to use the sequential map.

Example: an identity function
```javascript
result = pa.mapPar(function(val){return val;});
```
====
fromPar
----
__Array__

```javascript
Array.fromPar(source, elementalFunction=undefined, thisArg=undefined)
```

__Typed Object__

```javascript
TypedObject.fromPar(source, elementalFunction=undefined, thisArg=undefined)
```

##### Arguments

- *source*: The source values
- *elementalFunction*: if defined is called as described below, if undefined the source values are simple converted to Array or TypedObject elements.
- *thisArg*: If defined it is used as the this inside the elementalFunction

##### Elemental Function

```javascript
function (element, index, source, outCursor)
```
- *element*: The element from the source.
- *index*: The index in source where element is located as well as where the result will be placed.
- *source*: The source holding the elements.
- *outCursor*: Output cursor where results can be placed. If a non-undefined value is returned outCursor will be overwritten by the returned value. If outCursor is not specified the result of the function will be placed in fromPar’s result at index.

##### Returns

A freshly minted Array or Type Object. Elements are the results of applying the elemental function to the elements in the source and converting them according to the Array or Typed Object. If the elemental function is not provided the source values are converted into the appropriate destination type.

##### Throws

- *TypeError* when elementalFunction is not a function or undefined.
- *Error* when required by Typed Object conversion semantics.

##### Discussion

Unlike the sequential version of from the elemental function takes an index, the source, and an outcursor.

###### Example: Convert [1,2,3] to a Int32Array
```javascript
result = Int32Array.fromPar([1,2,3]);
result = Int32Array.fromPar([1,2,3], function(val){return val;});
```
====
reducePar
----
__Array__

```javascript
myArray.reducePar(elementalFunction)
```

__Typed Object__

```javascript
myArray.reducePar(elementalFunction)
```

##### Arguments

##### Elemental Function

```javascript
function (a, b)
```

- a, b are arguments to be reduced and returned

##### Returns

The final value, if the source Array or Typed Object has only 1 element then that element is returned.

##### Throws

- *TypeError*: when elementalFunction is not a function.
- *RangeError*: if the source Array or Typed Object is empty.

##### Discussion

reducePar is free to group calls to the elemental function and reorder the calls. For an elemental function that is associative the final result will be the same as reducing from left to right. Modular addition of integers is an example of an associative function and in this case the sum of an Array will always be the same regardless of the order that reducePar calls the addition operator. On the other hand, averaging is an example of a non-associative function. The expression Average(Average(2, 3), 9) produces the value 5 2/3 while the expression Average(2, Average(3, 9)) produces the value 4. reducePar is permitted to choose whichever call ordering it finds convenient.

reducePar is only required to return a result consistent with some call ordering and is not required to choose the same call ordering on subsequent calls. Furthermore, reducePar does not magically resolve problems related to overflow and the well documented fact that some floating point numbers are not represented exactly in EcmaScript and the underlying hardware so floating point addition and multiplication are not truly associative.

Typically the programmer will only call reducePar with associative functions but there is nothing preventing them doing otherwise. Calling reducePar with a non-associative function will lead to a result that is guaranteed only to be consistent with some ordering of applying the elemental function on some ordering of the arguments.

reducePar always works on the top level dimensions.

======
scanPar
-----
__Array__
```javascript
myArray.scanPar(elementalFunction)
```

___Typed Object___

```javascript
myTO.scanPar(elementalFunction)
```

##### Arguments
*elementalFunction* described below

##### Elemental Function

```javascript
function (a, b) 
```
- a, b - arguments to be reduced and result returned
The arguments will have the same grainType as the source elements. The returned result is converted immediately to the grainType of source elements.

##### Returns

scanPar returns a freshly minted Array or Typed Object whose i-th element is the result of using the elemental function to reduce the source elements between 0 and i, inclusively. Elements returned from elemental functions are converted to the type of the elements in the spec before being stored in the result.

##### Throws

- TypeError when elementalFunction is not a function.
- Error when required by Typed Object conversion semantics.

###### Example: Prefix Sum
```javascript
pa.scanPar(function(a, b){return a+b;})
```

##### Discussion

The construct implements what is known as an inclusive scan which means that the value of the i-th result is the same as what would be produced by [0..i].reducePar(elementalFunction). Notice that the first element of the result is the same as the first element in the original Array. An exclusive scan can be implemented by shifting right by one the results of an inclusive scan dropping the rightmost value, and inserting the identity at location 0. Similar to reducePar, scanPar can reorder the calls to the elemental functions. Ignoring overflow and floating point anomalies, this cannot be detected if the elemental function is associative and commutative; in which case using an elemental function such as addition to create a partial sum will produce the same result regardless of the order in which the elemental function is called. However using a non-associative or non-commutative function can produce different results due to the ordering that scanPar calls the elemental function. While scanPar will produce a result consistent with a legal ordering the ordering and the result may differ for each call to scanPar.

Typically the programmer will only call scanPar with associative and commutative functions but there is nothing preventing them doing otherwise. Calling scanPar with a non-associative and/or non-commutative function will lead to a result that is guaranteed only to be consistent with some ordering of applying the elemental function. One issue that has come up is when the coercion to grainType is done. The elementalFunction might see arguments from the source mixed up with intermediate results from previous invocation of the elementalFunction if the intermediate values are not converted immediately upon return from the elementalFunction. To avoid this the returned values are converted immediately to the grainType of the source element.

=====
scatterPar
-----
__Array__
```javascript
myArray.scatterPar(indices, defaultValue, conflictFunction, length)
```
__Typed Object__
```javascript
myTO.scatterPar(indices, defaultValue, conflictFunction, length)
```

##### Arguments

- *indices*: array of indices in the resulting array
- *defaultValue* (optional): argument indicating the value of elements not set by scatter. When not present, the default value is undefined
- *conflictFunction* (optional): function to resolve conflicts, details below.
- *length* (optional): argument indicating the length of the resulting array. If absent, the length is the same as the length of the source.

##### Returns

A freshly minted Array A where each element A[i] is defined as one of:

- A[indices[i]] = myArray[i], when indices[i] appears only once in indices
- A[indices[i]] = grainType(conflictFunction(valA, valB ) when multiple elements are scattered to the same location. The return value is immediately converted to grainType.

##### Throws

- *RangeError* when the length of indices does not match the length argument or, if length is not given, the length of the source Array or Typed Object.
- *TypeError* when conflictFunction is neither undefined nor a function.
- *RangeError* when a conflict occurs but no conflict function has been supplied by the programmer.
- *TypeError* when indices contains a value that cannot be interpreted as a number, e.g., +-Infinity and NaN.
- *RangeError* when indices contains an index smaller 0 or greater than the result array’s length.
- *Error* when required by Typed Object conversion semantics. Note that omitting a default value while using a type specification may lead to coercion errors if the default cannot be coerced to the specified type.

###### Example: an identity function
```javascript
result = pa.scatterPar(indices); where indices is a Array in which element === index
```

##### Handling conflicts with the Conflict Function

A conflict occurs when multiple elements are scattered to the same location. It results in a call to conflictFunction, which is an optional third argument to “scatterPar”. If conflictFunction is undefined, scatterPar throws an exception when a conflict occurs. Note that the order in which conflicts are resolved is left unspecified. Therefore, to ensure deterministic behavior, the conflict functions needs to be deterministic and associative.

##### Discussion

It is important to note here that for scatterPar, the arguments to the conflict resolution function are the already coerced values. The rationale is two-fold:

Conceptually, the conflict resolution function resolves conflicting stores to the result object. As only coerced values are stored, these are also the parties in the conflict.
Practically, this semantics enables a simpler and more efficient implementation, where results may be written to the final result buffer even though a conflict may still arise.
The value returned is converted to the grainType found in the source. The created “block” object uses a data storage format that complies with the type description ArrayType(typeSpec, length) where length is the value of the length argument or, if that is omitted, the value of the length property of the source array myArray.

```javascript
conflictFunction(valA, valB)
```

##### Arguments to conflict function

- valA, valB the two values that conflict

##### Returns

Value to place in result[indices[index]] . This value will be converted according to grain type found in the source.


###### Example: Resolve conflict with the larger value
```javascript
function chooseMax(valA, valB){
      return (valA>valB)?valA:valB;
 }
pa = [0,1,2,3,4,5];
result  = pa.scatterPar([0,3,1,4,2,5]);  		   // <0,2,4,1,3,5>
result2 = pa.scatterPar([0,0,1,1,2,2], 42, chooseMax);     // <1,3,5,42,42,42>
result3 = pa.scatterPar([0,0,1,1,2,2], 42, chooseMax, 3);  // <1,3,4>
```

=====
filterPar
-----
__Array__
```javascript
myArray.filterPar(elementalFunction)
```

__Typed Object__

```javascript
myTO.filterPar(elementalFunction)
```

##### Arguments
elementalFunction described below.

##### Elemental Function
```javascript
function (element, index, source) 
```
- *element*: The element from the source Array or Typed Object.
- *index*: The index in source where element is located.
- *source*: The source Array or Typed Object holding the elements.
- If the result of the function is truthy then the corresponding element will be placed in filterPar‘s result. Elements in the result will be in the same order as in the source.

##### Returns

A freshly minted Array holding the source elements located at myArray[i] for which elementalFunction returns a truthy value. The order of the elements in the result is the same as the order of the elements in the source.

##### Throws

- *TypeError*: when elementalFunction is not a function.
- *Error*: If required by the Typed Object conversion semantics.

##### Examples

###### Identity function
```javascript
var pa = [1,2,3,4,5,6,7]; 
var result = pa.filterPar(function(ignore){return true;});
```

###### Filter out every other element
```javascript
var pa = [1,2,3,4,5,6,7]; 
var result = pa.filterPar(function(element, index) { return index%2?false:true;} );
```

##### Discussion

The values are converted to the grain type of the source. The result uses a data storage format that complies with the grain type of the source.

======
buildPar
------
buildPar is a method of ArrayType, see typed objects. It is intended to be used to create new Arrays and Typed Objects in parallel.

__Array__
```javascript
ArrayType.buildPar(iterationSpace, elementalFunction)
```

__Typed Object__
```javascript
ArrayType.buildPar(iterationSpace, elementalFunction)
```

##### Arguments

- *iterationSpace*: the shape, if a scalar the length of a 1 dimensional array, otherwise an array of scalars specifying the shape of the result.
- *elementalFunction*: described below

##### Elemental Function
```javascript
function (index, outCursor)  
function (index, [outCursor1, outCursor2, …, outCursorN]) 
```
- *index*: The index in source where element is located as well as where the result will be placed.
- *outCursor* (optional): outCursor specifies a cursor where results can be placed. If a non-undefined value is returned outCursor will be overwritten by the returned value.
- *[outCursor1, outCursor2, … outCursorN]* (optional): An array of output cursors where results can be placed. If a non-undefined value is returned outCursor1 will be overwritten by the returned value. If a non-undefined value is returned outCursor1 will be set to the returned value.

##### Returns

If zero or one outCursor is specified: A freshly minted Array or Typed Object.

Otherwise: an array holding the arrays associated with the outCursors.

##### Throws

- *TypeError* when elementalFunction is not a function.
- *Error* when required by Typed Object conversion semantics.

##### Discussion

Typed Objects work by moving the shape and grain type specification into the type. For example
```javascript
var T = new ArrayType([20,40], uint32); 
var result = T.buildPar((i, j)=>i+j);
```
=====
flatten
-----
__Array__

Not available.

__Typed Object__
```javascript
myTO.flatten()
```
##### Arguments

None

##### Throws

*RangeError* when myArray is one dimensional.

##### Returns

Typed Object whose outermost two dimensions have been collapsed into one and where the outermost two ArrayType constructors of the descriptor for the underlying storage format have been combined.

###### Example
```javascript
T = new ArrayType(new ArrayType(uint8, 2), 2);
pa = new T ([[1,2], [3,4]])                  // <<1,2>,<3,4>>
flatPA = pa.flatten()                           // <1,2,3,4>
flatPA.elementType === ArrayType(uint8,4)
T3D = new ArrayType(T, 3);
pa3D = new T3D ( [[[1,2][3,4]], [[11,12][13,14]], [[11,22][23,24]]] );
// <<<1,2>,<3,4>>, <<11,12>,<13,14>>, <<11,22>,<23,24>>>
pa2D = pa3D.flatten(); // <<1,2>,<3,4>,<11,12>,<13,14>,<11,22>,<23,24>>
pa1D = pa2D.flatten(); // <1,2,3,4,11,12,13,14,11,22,23,24>
```
======
partition
------
__Array__

Not available

__Typed Object__
```javascript
myTO.partition(size)
```

##### Arguments

- *size*: the size of each element of the newly created dimension; the outermost dimension of myArray needs to be divisible by size.

##### Returns

A freshly minted block where the outermost dimension has been partitioned into elements of size size.

##### Example
```javascript
T = new ArrayType(uint8, 4);
pa = new T([1,2,3,4])  // <1,2,3,4>
pa2D = pa.partition(2)                    // <<1,2>,<3,4>>
```
##### Discussion

While one could implement both flatten and partition using the other constructs we call them out here to make it easy for the compiler to recognize flatten or partition and make optimizations easier.

##### Throws

*RangeError* when outermost dimension is not divisible by size.
