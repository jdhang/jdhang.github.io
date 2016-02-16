---
layout: post
comments: true
title: "Discovery - Circular Error in Object Assignments"
date: 2015-03-06
categories: javascript
tags:
  javascript
  code
  beginner
---
As I was progressing through the book [Eloquent Javascript][ejs], which I highly
recommend for any beginner programmer or anyone that wants to start learning
javascript, I got to one of Chapter 4 - Data Structures: Objects and Arrays
exercises and got stuck. The exercise called for the creation of a function
called `arrayToList` that took in an array and output a new data structure called a list.

<!--break-->

A list is an object with a value property and a rest property, which is another list object.
Say the input array was `[1, 2, 3]`, the output would be `{value: 1, rest:
{value: 2, rest: {value: 3, rest: null}}}`. Notice the last `list` object has a
`rest` property of `null`.

My initial solution looped through the array backwards creating a new `list`
object with the `value` equal to the array value at the looping index and the
`rest` equal to a `temp` variable containing the previous `list` object. The
function would then return the `list` object.

My code looked like the following:

``` javascript
function arrayToList(array) {
  var list = {};
  var temp = {};
  for (var i = (array.length - 1); i >= 0; i--) {
    if (i === (array.length - 1)) {
      list = {value: array[i], rest: null};
    } else {
      list.value = array[i];
      list.rest = temp;
    }
    temp = list;
  }
  return list;
}
```

Now to me this made sense because I was assigning the known properties of `list`
with the dot notation. This unfortunately gave me unexpected results when I ran
the code. The output was the following:

``` shell
> console.log(arrayToList([1, 2, 3]));
{value: 1, rest: [Circular]}
```

This was very frustrating to me and I could not figure out what the problem was.
I tried to google the answer but came up with nothing. Although part of the
problem was that I wasn't sure what to ask entirely. After an hour of hair
pulling I decided to just assign the `list` object outright.

I changed this

``` javascript
list.value = array[i];
list.rest = temp;
```

to:

``` javascript
list = {value: array[i], rest: temp}
```

And when I ran the code again I got the result I wanted.

``` shell
> console.log(arrayToList([1, 2, 3]));
{value: 1, rest: {value: 2, rest: {value: 3, rest: null}}}
```

Now this did not initially make sense to me, why does the explicit reassignment
but the dot notation of individual properties raise an error. After some
reasoning I realized the the problem was not the dot notation of assignments for
`list` but the assignment of the `temp` variable used to store the previous
`list` object!

`temp = list;` intuitively makes sense to me, however in Javascript `temp` is
now referencing the `list` object, not storing a new object with the same values
and properties as `list`, which is what I would think.

So this is why I was getting the circular error because temp would keep
referring to `list` as I was reassigning values within the `list` object. If I
changed the assignment of `temp` to create a "clone" of `list`, basically create
a new object with the same values as `list`, then my code with the dot notation
assignments would work.

``` javascript
temp = list;
// becomes
temp = {value: list.value, rest: list.rest}
```

And it did!

In the end I felt that reassigning the `list` object with this code:

``` javascript
list = {value: array[i], rest: temp}
```

seemed better than explicitly reassigning the properties. In addition to that I
felt the need to set `temp` to the `list` properties to be safe as well.

``` javascript
temp = {value: list.value, rest: list.rest}
```

**To summarize:**

A variable assigned to an object will refer to the same instance of an object
for the life of the program. If the variable is suppose to be a clone of the
instance of the object, make sure you create a new object with the same values
in order to perfect a circular error, a.k.a. clone it!.

This might have been way too long of a post for something that might be trivial
to some javascript developers but I thought it was good to write about it.

Hopefully other beginner developers don't run into the same mistake I did! Happy
Coding!

[ejs]: http://eloquentjavascript.net
