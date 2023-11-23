---
{"dg-publish":true,"permalink":"/zettelkasten/javascript-asynchronous-programming/","tags":["javascript","typescript","programming-language","async-programming","async/await","promise"]}
---

# Introduction

We'll start by explaining and walking our way through JavaScript's synchronous execution model, and then end up making an argument for why some operations need to be asynchronous. From there we'll segue way into asynchronous programming, go through some examples using the `setTimeout` function, .

The section about callback-side asynchronous programming in JS will be mostly a history because after promises were implemented into the language, most Web APIs and libraries that support asynchronous operations are written around promises.

- Get room number
- Get professor’s ID
    - Get professor’s name & subject
- Get students

# Prerequisites

To get the most out of this document, you should have a high-level understanding of the following JavaScript concepts:

- Functions as first-class objects, i.e., functions can be assigned to variables, they can be passed as arguments to other functions, they can be stored in data structures such as arrays and objects, and they can be returned from other functions. If you’ve used functions as first-class objects in Perl, Python, Ruby, etc., then you’re be good to go.
- [Function expressions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/function) and [arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions). If you’ve used lambdas or anonymous functions in Perl,Python, Ruby, etc., then you’re good to go. In JavaScript, arrow functions are a more limited and concise version of the traditional function expressions. On this document, we prefer the latter for its cleaner syntax.
- The call stack (not unique to JS), the callback queue, the micro-tasks queue, [web APIs](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Client-side_web_APIs/Introduction), and the event loop. [Philip Roberts’s What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ) should be enough.

# Synchronous Programming

JavaScript is a **single-threaded** programming language with a **synchronous** model of execution. In simple terms, this means that it executes one instruction at a time and in order. Let’s try to understand *single-threadedness* and synchronicity separately.

## Single-threadedness

For example, let’s say we’ve four functions `foo`, `bar`, `qux`, and `wan`:

```jsx
foo();
bar();
qux();
wan();
```

When the JavaScript engine is executing the code, all the functions are pushed into a single ********call stack******** and only the function at the top of the call stack can execute until it’s popped from the stack when it returns, either explicitly or implicitly. In other words, only one function can execute at a given time in JavaScript’s ********************************************main thread******************************************** of execution. After one function finishes executing, the next one is executed, and this continues until the call stack is empty, i.e., all code is executed.

```jsx
MAIN THREAD: foo() -> bar() -> qux() -> wan()
```

If JavaScript was a **multi-threaded** programming language, different functions could be executed in multiple threads at the same time.

```jsx
THREAD 1: foo()
THREAD 2: bar() -> qux()
THREAD 3: wan()
```

## Synchronicity

Synchronous means instructions are executed one after the other in order. For example, using the previous with functions `foo`, `bar`, `qux`, and `wan`:

```jsx
foo();
bar();
qux();
wan();
```

Line 1 executes first:

```jsx
foo(); // <= This line is executing...
bar();
qux();
wan();
```

Line 2 executes first:

```jsx
foo(); 
bar(); // <= This line is executing...
qux();
wan();
```

Line 3 executes first:

```jsx
foo(); 
bar();
qux(); // <= This line is executing...
wan();
```

Line 4 executes first:

```jsx
foo(); 
bar();
qux();
wan(); // <= This line is executing...
```

In a synchronous execution model, if there’s a piece of code that might take a long time to execute, then everything comes to a halt until the code being executed is finished. This behavior is known as ****************blocking****************, i.e., the executing piece of code is blocking instructions after it whether or not depend on it.

In order to illustrate this behavior, we’ll see the following piece of code where we declare a function `displayDate`, which loops 100 million times and during each iteration it creates a date object, and it logs it at the end of the loop. The use of the delay is to simulate a time-consuming and the effect it has on the code that follows it.

```jsx
function displayDate() {
	let date;
  for (let i = 0; i < 100000000; i++) {
	  date = new Date();
	}
  
  console.log(date);
}

console.log('START');
displayDate();
console.log('END');
```

Copy the code, paste it in your browser’s console, and hit `Enter`. When you run the code, you’ll notice it will output `START`, and then it will take a few seconds to output anything else when `displayDate` is executing. The reason for this is that looping 100 million times is a time-consuming task, and when `displayDate` runs it’s halting everything else, i.e., it’s blocking the main thread of execution.

Synchronous is useful in most cases, i.e., we want instructions to execute in the order we declare them. However there are instances where such behavior is unwanted. For example, imagine if `console.log('END')` was handling some user interface unrelated to the `displayDate` function. With synchronous execution and JavaScript being single-threaded, the whole UI will come to a halt until the function `displayDate` finishes executing. For instance, the browse wouldn’t be able to handle things such as user input, like scrolling or clicking a button, until the function completes. This would be inefficient but more importantly it’d make the entire experience frustrating to users.

In addition to time-consuming tasks, there are other situations where synchronous behavior is unwanted, with networking and I/O operations among them:

- Retrieving data from an API.
- Reading files from a file system

Retrieving data from an API usually involves sending a network request to a server and waiting for a response. The waiting time can be a couple of seconds or minutes, and this might vary depending on the internet speed. If there are functions that depend on the data to be returned from the API, in a synchronous execution model they will need to wait for the response from the server before they can run. Likewise, functions that don’t depend on the response from API won’t execute until the response is received, which is wasteful, inefficient, and frustrating.

Now we’ll … that returns an array of planet names and to simulate a call to an API we’re using a delay just like before:

```jsx
// assume this function makes a network request and returns an object
// with an array of planets, e.g., { planets: ['mercury', ...] }
function getPlanets() {
	// delay
	for (let i = 0; i < 100000000; i++) { new Date(); }

  // data from API
  const planets = [
		'mercury', 'venus', 'earth', 'mars', 
    'jupiter', 'saturn', 'uranus', 'neptune'
  ];

  console.log('Data from API');
  return planets;
}

function displayPlanets(response) {
	console.log("Planets: ", response);
}

console.log('Button 1');

const response = getPlanets();
displayPlanets(response);

console.log('Button 2');
```

When we run this code, we get the following output:

```markdown
Button 1
Data from API
Planets:  (8) ['mercury', 'venus', 'earth', 'mars', 'jupiter', 'saturn', 'uranus', 'neptune']
Button 2
```

The message `Button 1` is logged, then the function `getPlanets` starts executing and logs the message `Data from API`, followed by returning an object with data. When `getPlanets` finishes, `displayPlanets` takes the response and logs it to the console. Finally the message `Button 2`, which has nothing to do with the simulated API response, is logged.

As you experienced when you ran the code, this synchronous behavior is undesirable: `console.log('Button 2')` is independent of the data returned by `getPlanets` and thus it shouldn’t have to wait it finishes executing. Like mentioned before, in a real-world scenario, this could be handling events such as user input, button clicks, UI rendering, etc. In a synchronous execution model, these events will halt until the time-consuming task finishes executing, which will result in a bad experience for the users. 

We know only `displayPlanets` depends on the data `getPlanets` returns so what if we could put `getPlanets` aside (i.e., it executes in the background and then `displayPlanets` is called when `getPlanets` finishes executing) and continue with the execution of the rest of the code? The output would look like this:

```markdown
Button 1
Button 2
Data from API
Planets:  (8) ['mercury', 'venus', 'earth', 'mars', 'jupiter', 'saturn', 'uranus', 'neptune']
```

It turns out we can do this, which is the basis of ************************asynchronous programming************************.

# Asynchronous Programming

In **************asynchronous************** code, operations that are time-consuming and that happens at unpredictable times are put in the background in order to to prevent blocking any code in the main thread which continues executing normally. In other words, asynchronous operations don’t pause execution of other functions in the call stack.

## setTimeout

Before ES6, a popular way of doing asynchronous programming in JS was by using the `[setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout)` function, which is an asynchronous function. `setTimeout` is one of the web APIs functions and it’s specifically a method of the `window` object, which it’s globally available to JS code. Its syntax is as follows:

```jsx
setTimeout(functionRef, delay, param1, ..., paramN)
```

The argument `functionRef` is the function to be executed after the timer expires, `delay` is the number of milliseconds after which `functionRef` is executed, and `param1, ..., paramN` are arguments that are passed to `functionRef` when it’s executed. Only `functionRef` is a required parameter; if `delay` is omitted, the value 0 is used which means execute `functionRef` immediately in the next event cycle.

Let’s illustrate how `setTimeout` works with this example:

```jsx
console.log('START');

setTimeout(() => {
    console.log('first');
}, 1000);

setTimeout(() => {
    console.log('second');
}, 500);

console.log('END');
```

If we run this code, we get this output:

```html
START
END
second
first
```

We know `setTimeout` is an asynchronous function, and thus it places its code block in the background, which is run in the next event cycle. That’s why the first two lines in the output are `START` and `END`, despite the way the code is structured; unlike asynchronous code, synchronous code is executed immediately by the JS engine.

Why do we get `second` logged before `first` though? The code blocks for `first` and `second` in the `setTimeout` invocations are placed in the background with a timer of `1000` milliseconds and `500` milliseconds respectively; obviously the `500` ms timer finishes earlier than the `1000` ms timer, which means the code block `() => { console.log('second') }` executes before  `() => { console.log('first') }` in the next event cycle, which explains why `second` is logged before `first`.

## Making Our Code Asynchronous with setTimeout

Now that we know how `setTimeout` works, and how we can use it to make our code asynchronous, we will rewrite our code to be asynchronous with it.

```jsx
function getPlanets() {
  setTimeout(() => {
		for (let i = 0; i < 100000000; i++) { new Date(); }

	  const planets = [
			'mercury', 'venus', 'earth', 'mars', 
	    'jupiter', 'saturn', 'uranus', 'neptune'
	  ];

	  console.log('Data from API');
	  return planets;
	}, 0);
}

function displayPlanets(response) {
	console.log("Planets: ", response);
}

console.log('Button 1');

const response = getPlanets();
displayPlanets(response);

console.log('Button 2');
```

When we run this code, we get:

```markdown
Button 1
Planets:  undefined
Button 2
Data from API
```

If you look at the output, there are a few things to notice:

- The message `Data from API` has been logged after `Button 2`, even though it was called before it.
- The `getPlanets` function was meant to return an array of planets, yet `response` is `undefined`.

These two things mean our code is behaving asynchronously, i.e., it’s not longer waiting for the time-consuming `getPlanets` function to execute the other statements. This is great, but in our quest for asynchronous code `getPlanets` lost the ability to return values. 

If you think about it, it makes sense the variable `response` is `undefined`. When JS engine executes the line `const response = getPlanets();`, it doesn’t know `getPlanets` is a function that will take time to produce a value so it simply moves to the next line of code; `getPlanets` doesn’t return anything on time, thus `response` is `undefined`. Once we go asynchronous, we cannot use asynchronous functions for their return values. However it’s usually the case we want to do use that value (e.g., display it like what `getDisplay` does) so how do we get a hold of it? The only option we’ve is to pass a function (to the asynchronous function) that will use the value in some way. In other word, we need to pass a ****************callback****************.

<aside>
💡 **Takeaway:** An asynchronous function cannot be used for its return value. Instead we must pass a callback that will use the value the asynchronous function produces.

</aside>

# Building Up to Callbacks

JavaScript treats functions just as any other object, e.g., they can be passed around, returned from other functions, etc. In JavaScript, a function that is passed as an argument to another function is known as a ****callback****. In general, there are two main uses for callbacks:

- For transforming or filtering out values. In this example, we transform each number by squaring it and returned the transformed array.

```jsx
const nums = [1, 2, 3, 4, 5];
const squaredNums = nums.map((num) => num * num);
const oddNums = nums.map
```

- For delaying execution until a particular time. In this example, execution is delayed until a user clicks a button, at which point the user is alerted with `Button clicked!`.

```html
<button id="click-me">Click me!</button>

<script>  
    let button = document.querySelector('#click-me');
    button.addEventListener('click', () => { alert('Button clicked!'); });
</script>
```

Moving forward, we’ll use the term ********callback******** as used in the second case (i.e., delayed execution until some time), which is what we’re interested in. Instead of delayed execution of a function until a user clicks a button, we’re interested on delaying execution of a function until we receive data from an API, for example. If we want `displayPlanets` to execute only when `getPlanets` finishes, then we must pass it as a callback. Before `getPlanets` returns, we will invoke `displayPlanets` with the value it’d have returned otherwise.

## Using Callbacks

Here we modify the function `getPlanets` to accept a callback. We can name the callback anything we want (since it’s simply a function), however a convention is to use `callback` or `cb`. We use the former.

```jsx
function getPlanets(callback) {
  setTimeout(() => {
		for (let i = 0; i < 100000000; i++) { new Date(); }

	  const planets = [
			'mercury', 'venus', 'earth', 'mars', 
	    'jupiter', 'saturn', 'uranus', 'neptune'
	  ];

	  console.log('Data from API');
	  callback(planets); // <== calling the callback with data
	}, 0);
}

function displayPlanets(response) {
	console.log("Planets: ", response);
}

console.log('Button 1');

getPlanets(displayPlanets);

console.log('Button 2');
```

If we run the above code, we get the following output:

```html
Button 1
Button 2
Data from API
Planets:  (8) ['mercury', 'venus', 'earth', 'mars', 'jupiter', 'saturn', 'uranus', 'neptune']
```

*Yay!* We got the output we expected: 

- Both `Button 1` and `Button 2` are logged in the right order and aren’t blocked by `getPlanets`.
- We pass `displayPlanets` to `getPlanets`, which the latter executes with the fictitious API response.
- The function `getPlanets` is asynchronous and thus it returns no useful value. Thus assigning it to a variable is no use to us. That’s we removed the assignment to `response`.

We used the named function `displayPlanets` because it was already declared in our code. However it’s convention to simply pass the callback as an arrow function or a function expression. 

We can rewrite the above snippet as follows:

```jsx
function getPlanets(callback) {
  setTimeout(() => {
		for (let i = 0; i < 100000000; i++) { new Date(); }

	  const planets = [
			'mercury', 'venus', 'earth', 'mars', 
	    'jupiter', 'saturn', 'uranus', 'neptune'
	  ];

	  console.log('Data from API');
	  callback(planets);
	}, 0);
}

console.log('Button 1');

getPlanets((planets) => {
	console.log("Planets: ", planets);
});

console.log('Button 2');
```

We simply declared the callback right in `getPlanets`'s arguments list when call `getPlanets` (remember, functions are first-class objects in JS). That’s all, and we get the same output as above if we run this code. Try it!

# Callbacks

By now we’ve an understanding of synchronous vs asynchronous code in JavaScript and how callbacks are used to deal with asynchronous code. From here onward, we’ll use more a real-world example where we use callbacks, find out the issues with them, and why we needed a more modern alternative that addressed the issues with callbacks.

## Setup

We’re tasked with creating an information card for lecturers that will give conferences in their respective areas of expertise. The information card will look as follows:

```markdown
Professor: Edsger Dijkstra
Subject: Algorithm Design
Room: C2
Attendees: 12
```

 We’ll be requesting the needed information from an API:

- An object representing a professor, which contains the properties `name` and `id`.
- Using the professor’s `id` property, we can request both the professor’s area of expertise and the room he will be conducting the lecture at.
- An array containing the names of the attendees.

## Callback-based API

We’ll be using four asynchronous functions which mimic the behavior we get when we call an API:

- `getLecturer(callback)` accepts a callback to which it passes an object `{ id: ..., name: ...}`.
- `getRoom(id, callback)` accepts a lecturer’s `id` and a callback to which it passes a room number.
- `getSubject(id, callback)` accepts a lecturer’s `id` and a callback to which it passes the lecturer’s subject.
- `getAttendees(callback)` accepts a callback to which it passes an array of attendees.

Each function simulates an API call: we receive responses from each asynchronous function within 1-5 seconds, which we’re doing to simulate a deterministic delay from a network request. Once we’ve all the data required data we’ll create the specified info card.

We won’t provide the source code for the functions right here but you can find it at the end of this document. The idea being we don’t need to know how they’re implemented, all we need to know is they’re callback-based asynchronous functions, they pass arguments to the callback, and that’s all we care about.

## Working with Callbacks

Let’s start with `getRoom` and pass a callback that logs the argument `getRoom` passes to it:

```jsx
getLecturer(lectObj => {
    console.log(lectObj);
});
```

When we execute this code, we should get a room number:

```jsx
{ id: 3, name: 'Barbara Liskov' }
```

However we’re not only interested in a lecturer, we’re also interested on the lecturer’s subject, lecturer’s assigned room, and the attendees. We already have access to the object representing the lecturer, and we can use the lecturer’s `id` to request the subject and the assigned room. In order to do this, we need to nest each callback and by doing so we will make the received data available to the innermost callback. Thus we end up with the following:

```jsx
getLecturer(lectObj => {
    const { id, name } = lectObj;
    getSubject(id, subject => {
        getRoom(id, room => {
            getAttendees(attendees => {
                console.log(name, subject, room, attendees);
            });
        });
    });
});
```

The callback we pass to `getLecturer` makes both `id` and `name` available to the callback we pass to `getSubject`, which in turns makes `id`, `name`, and `subject` available to `getRoom`, and so on. Inside the innermost callback, we’ve access to the argument passed to it as well as all the arguments passed to the outer callbacks. This is known as [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style) and it’s a common style to make arguments available down the callback chain.

If we run the code, we might get something as follows:

```html
Alan Turing Theory of Computation C1 [
  'Nina Morse',
  'Raphael Zuniga',
  'Shayna Newton',
  'Elise Huber',
  'Riley Le',
  'Tommy Payne',
  'Brooks Cortez',
  'Mya Morton',
  'Kole Sandoval',
  'Darwin Simpson',
  'Tristin Choi',
  'Isaiah Rowe'
]
```

As the log shows, we got the lecturer’s name, subject, assigned room, and a list of attendees. With the data available it couldn’t be easier to create the info card and logging it to the console:

```jsx
getLecturer(lectObj => {
    const { id, name } = lectObj;
    getSubject(id, subject => {
        getRoom(id, room => {
            getAttendees(attendees => {
                let infoCard = '';
                infoCard += `Lecturer: ${name}\n`;
                infoCard += `Subject: ${subject}\n`;
                infoCard += `Room: ${room}\n`;
                infoCard += `Attendees: ${attendees.length}`;
                console.log(infoCard);
            });
        });
    });
});
```

If we run the code, we might get the following info card:

```html
Lecturer: Barbara Liskov
Subject: Distributed Computing
Lecturer: D2
Lecturer: 5
```

To make our code tidier, let’s outsource creating and displaying the info card to a function:

```jsx
function displayInfoCard({lecturer, subject, room, attendees}) {
    let infoCard = '';
    infoCard += `Lecturer: ${lecturer}\n`;
    infoCard += `Subject: ${subject}\n`;
    infoCard += `Room: ${room}\n`;
    infoCard += `Attendees: ${attendees.length}`;
    console.log(infoCard);
}

getLecturer(lecturer => {
    const { id, name } = lecturer;
    getSubject(id, subject => {
        getRoom(id, room => {
            getAttendees(attendees => {
                displayInfoCard({name, subject, room, attendees});
            });
        });
    });
});
```

We’ve created the expected info card but we cannot ignore that as we pass callbacks, the more indented and the harder to reason about our code gets. This structure that results from nesting callbacks is known as **callback hell**, and that’s something we want to avoid if possible because not only is the code hard to read and understand but also slow. 

The reason why our code is slow has to do with the ordering of the events or which one depends on the other. For example, in order to invoke `displayInfoCard` we definitely have to wait for the functions `getLecturer`, `getSubject`, `getRoom`, and `getAttendees` to get all the data, however it’s not the case that `getSubject`, `getRoom`, and `getAttendees` have to wait for each other since they don’t depend on one another. With the current setup, `getRoom` and `getAttendees` are needlessly waiting for `getSubject`, and the amount of time each of them waits compounds. 

In order to do away with this inefficiency, what we want is to start `getSubject`, `getRoom`, and `getAttendees` start independently, e.g., they don’t depend on one another and we don’t care which one finishes first. Instead what we care about is they’ve finished by the time we call `displayInfoCard` since it depends on them having finished. Unfortunately JavaScript doesn’t have a way of managing callbacks like this and it’s the programmer’s job to take care of any setup to achieve it.

<aside>
💡 **Takeaway:** Asynchronous programming with callbacks in JS has two main problems: First, callback hell, and second, JS doesn’t provide an easy way to start several independent requests at the same time in order to improve performance, which makes asynchronous programming with callbacks a frustrating experience for the programmer.

</aside>

## Multiple Callbacks at the Same Time

With asynchronous programming with callbacks, JS doesn’t provide us with a way to start multiple requests at the same time. However that didn’t dissuade programmers from finding ways of achieving better performance, even to the detriment of the readability of their code. This section describes a way to do this:

- Create a global variable that holds the data that will be passed to the callback that depends on it.
- Implement a logic mechanism to check if this global variable have all the data after all the independent asynchronous functions complete. If all the data is available, then call the dependent asynchronous function with that data. Remember those independent asynchronous functions will finish at different time, which is why we need this logic mechanism.
- Run each independent asynchronous function separately, and inside the callback make sure to invoked the logic mechanism after updating the global variable with its data.

Let’s commence! We start by creating a global variable (e.g., `data`) that will hold the data from `getRoom`, `getProfessor`, and `getStudents`.

```jsx
let data = {};
```

These functions will be running at the same time but there’s no guarantee on which one finishes first, thus we must implement some logic that takes care of checking if `data` has the data `getMessage` needs and only then call `getMessage`. 

```jsx
let data = {};

function checkDataCompleteness() {
    if (
        data.hasOwnProperty('lecturer') &&
        data.hasOwnProperty('subject') &&
        data.hasOwnProperty('room') &&
        data.hasOwnProperty('attendees')
    ) {
        displayInfoCard(lecturer);
    }
}
```

By implementing this logic, we make sure `displayInfoCard` gets called only when the combined data from `getLecturer`, `getSubject`,  `getRoom`, and `getAttendees` is available. However, we’re not done yet: we must call `getLecturer`, `getSubject`, `getRoom`, and `getAttendees` separately, and implement some logic that updates `data` with their respective values, whenever they’re available, and call `checkObjectCompleteness`. It’s worth noting that we cannot call neither `getSubject` nor `getRoom` without an `id`, thus they both depend on `getLecturer`; however we can call `getSubject` and `getRoom` at the same time.

```jsx
let data = {};

function checkDataCompleteness() {
    if (
        data.hasOwnProperty('lecturer') &&
        data.hasOwnProperty('subject') &&
        data.hasOwnProperty('room') &&
        data.hasOwnProperty('attendees')
    ) {
        displayInfoCard(data);
    }
}

getLecturer(lecturer => {
    data['lecturer'] = lecturer.name;

    getSubject(lecturer.id, subject => {
        data['subject'] = subject; 
        checkDataCompleteness();
    });

    getRoom(lecturer.id, room => {
        data['room'] = room; 
        checkDataCompleteness();
    });
    
    checkDataCompleteness();
});

getAttendees(attendees => {
    data['attendees'] = attendees;
    checkDataCompleteness();
});
```

Ok, our code is now complete and it should be more performant than its more callback hell-ish version. If we run this code, we might get the following:

```html
Lecturer: John von Neumann
Subject: Computer Architecture
Room: E1
Attendees: 5
```

Our code might perform better than the deeply nested callbacks sequence, however at what cost? 

- We now must keep a global variable `data`, which is accessible to all our code.
- We needed to implement some logic mechanism with `checkDataCompleteness` in order to know when `displayInfoCard` should be called.
- Each function has to update `data` and call `checkDataCompleteness`, which is repetitive and error prone.

We need to do too much book-keeping for something that should be fairly simple. With this, we’ve established that asynchronous programming using callbacks doesn’t scale very well. For a long time, there was no way to get around callback hell and nasty code in JavaScript; programmers did it this way because they had no choice, until promises were introduced to the language.

Promises solve the main two issues we dealed with here:

- They get rid of callback hell.
- It allows programmer to start multiple requests at the same time

The next section introduces the concept of promises.

<aside>
💡 **Takeaway:** Making multiple independent requests when working with callback-based asynchronous code in JS is tedious, frustrating and error prone experience because the language doesn’t provide any constructs to make this easier.  However promise-based asynchronous JS takes care of this.

</aside>

# Promises

**Promises** are the foundation of asynchronous programming in modern JavaScript, and they help programmers write cleaner asynchronous code. A promise is an object that represents a value which will eventually be available: right now or a bit later in the future.

A `Promise` object can be in one of these states:

- *fulfilled*, which means the operation was completed successfully. (e.g., network request got the data).
- *rejected*, which means the operation failed. (e.g., network request encountered an error).
- *pending*, which is the promise’s initial state; it’s neither fulfilled nor rejected. (e.g., network request isn’t neither resolved nor rejected).

The eventual state of a *pending* promise can either be *fulfilled* with a value or *rejected* with a reason (or error). A promise that has been either ****fulfilled**** or ********rejected******** is known to be *******settled*******. 

We interact with promises by passing a callback to its `then` method, which takes up to two arguments: the first argument is a callback function for the fulfilled case of the promise, and the second argument is a callback function for the rejected case. Conventionally, both callbacks are referred to as `onSuccess`, and `onFailure` respectively. Each `.then()` returns a newly generated promise object, which can optionally be used for chaining.

## Promise-based API

We’ll be using the same set of functions from the ****************Callback**************** section, however their underlying implementation is now based on promises:

- `getLecturer()` returns a promise that resolves to the object `{id: ..., name: ...}`.
- `getRoom(id)` accepts a lecturer’s `id` and returns a promise that resolves to a room number.
- `getSubject(id)` accepts a lecturer’s `id` and returns a promise that resolves to a subject.
- `getAttendees()` returns a promise that resolves to an array of attendees.

## Working with Promises

Let’s start with `getLecturer`:

```jsx
getLecturer()
.then((lecturer) => {
  console.log(lecturer); 
});
```

You’ll notice it’s similar to the way we did it previously, however instead of passing a callback to `getLecturer` we pass it to `then`. We don’t know when `getLecturer` will finish, however whenever it finishes or resolves, *then* the callback we pass to `then` is called.

Let’s now grab the remaining data:

```jsx
getLecturer()
.then(lecturer => {
  console.log(lecturer.name);
  
  getRoom(lecturer.id)
  .then(room => {
    console.log(room);
  });

  getSubject(lecturer.id)
  .then(subject => {
    console.log(subject);
  });

  getAttendees()
  .then(attendees => {
    console.log(attendees);
  });
});
```

THIS NEEDS MORE EXPLAINING.

## Promise.all

The `Promise.all()` method fulfills when **all** the promises fulfill. It accepts an array of promises, and returns a new promise that resolves into an array of the fulfillment values in the same order.

```jsx
getLecturer()
.then(lecturer => Promise.all([
    lecturer.name,
    getSubject(lecturer.id), 
    getRoom(lecturer.id),
    getAttendees(),
  ])
)
.then(res => {
  console.log(res)
});
```

THIS NEEDS MORE EXPLAINING.

# Async/Await

`async`/`await` isn’t so much a new way to do asynchronous programming in JS; instead

This new syntax removes awkward functions, provides a clear linear style code which looks synchronous, explicit asynchronous points, and native try/catch error handling (we’ll touch on this later).

The simplest explanation of how this works is that `await` takes a promise, waits for it's value to be available, and then returns that value.

## Setup

We’ll be using the same implementation of the functions we used in the Promises section. They’ve a promise-based API, and that’s what async/await. Remember async/await isn’t a new mechanism for handling asynchronous code in JavaScript; it’s simply syntactic sugar for promises that makes their consumption more synchronous-like.

## Working with Async/Await

Both `async` and `await` work in conjunction:

- `await` **********awaits********** the resolution of a promise and a promise only.
- `async` tells the JS engine that a function is asynchronous and the programmer may use `await` inside.

You can only use `await` inside an `async` function, and using `await` without `async` will simply throw a syntax error.

Let’s see an example with `async`/`await`:

```jsx
async function logLecturer() {
  const lecturer = await getLecturer();
  console.log(lecturer.name);
}

logLecturer();
```

Inside `logLecturer`, we await `getLecturer` and then log the lecturer’s name. This is equivalent to

```jsx
function logLecturer() {
	getLecturer()
  .then(lecturer => console.log(lecturer.name));
}

logLecturer();
```

When working with `async`/`await`, a common convention is to separate the functions into those that return values and those with side effects, thus we could refactor the above code as follows:

```jsx
async function lecturerName() {
	const lecturer = await getLecturer();
  return lecturer.name;
}

async function logLecturer() {
  const name = await lecturerName();
  console.log(name);
}

logLecturer();
```

Note how it reads like synchronous code, despite being asynchronous. Another thing worth noting is that `async`/`await` is infectious: If a functions awaits an async function, then that function itself must be async.

## Refactoring Info Card with Async/Await

At a first pass, this is how our info card code using await/async might look like:

```jsx
async function getInfoCard() {
  const { lecturer, id } = await getLecturer();
  const subject = await getSubject(id);
  const room = await getRoom(id);
  const attendees = await getAttendees();
  return { lecturer, subject, room, attendees };
}

async function infoCard() {
  const info = await getInfoCard();
  displayInfoCard(info);
}

infoCard();
```

However we can improve this! The functions `getSubject`, `getRoom`, and `getAttendees` are independent from each other and we can handle all of them at once with `Promise. all`:

```jsx
async function getInfoCard() {
  const { name, id } = await getLecturer();
  const [subject, room, attendees] = await Promise.all([
		getSubject(id), getRoom(id), getAttendees(),
	]);
  
	return { name, subject, room, attendees };
}

async function infoCard() {
  const info = await getInfoCard();
  displayInfoCard(info);
}

infoCard();
```

Running the code we might get the following:

```markdown
Lecturer: Margaret Hamilton
Subject: Software Engineering
Room: D1
Attendees: 5
```

# Handling Error with Promises

All the functions in the **Promises** section always resolve to a value, and thus we never needed to deal with a potential error. However in the real world things are less than ideal.

In this section we’ll use the same functions, however they’ve been updated to return promises that either resolve or reject. There’s 50/50 chance the promise will resolve or reject, which means we must take care of both resolution and rejection.

## A More Complete .then()

When we introduced the `.then()` method, we weren’t completely honest about the method. In reality, the method takes up to two callbacks as arguments:

- the first argument is a callback function for the fulfilled case of the promise.
- the second argument is a callback function for the rejected case.

Let’s start with `getRoom`:

```jsx
getRoom()
.then(
    (room) => { console.log(room)  }, // callback to handle resolution
    (error) => { console.log(error) } // callback to handle rejection
);
```

If we run this code and the promise resolves, we might get:

```markdown
E1
```

If we run this code and the promise rejects, we get the following:

```markdown
Error: No room
```

## .catch()

If we only want to handle rejection only, then we can pass `undefined` as the first argument to `.then()`.

```jsx
getRoom()
.then(
    undefined,
    (error) => { console.log(error) }
);
```

This is a common pattern so there’s the `.catch` that works just like `.then()` but only deals with rejection. Thus the previous can be rewritten as follows:

```jsx
getRoom()
.catch((error) => {
    console.log(error)
});
```

## .then() and .catch()

The `.catch()` method is chainable just like `.then()`, thus you’ll usually see a series of chained `.then()` for the resolved cases, followed by a `.catch()` for the rejected cases. For example:

```jsx
getRoom()
.then((room) => {
   console.log(room)
})
.catch((error) => {
    console.log(error)
});
```

This is the same as the following but it arguably reads better:

```jsx
getRoom()
.then(
    (room) => { console.log(room)  },
    (error) => { console.log(error) }
);
```

We’ve now talked about promise’s resolution and rejection so we can describe the methods `Promise.allSettled`, `Promise.race`. and `Promise`. However before this, let’s talk about two simple and straightforward methods:

- `Promise.resolve()` which returns a `Promise` object that resolves into a given value.
- `Promise.reject()` which returns a `Promise` object that is rejected with a given reason or error.

## Promise.allSettled

The `Promise.allSettled` method returns a promise that fulfills when all the promises settle (i.e., fulfill or reject). It accepts an array of promises, and returns an array of objects that describe the outcome of each promise.

# Handling Error with Async/Await

We handle errors with in `async`/ `await` the same we handle error with synchronous code in JS. Namely, using `try/catch`. If the promise we await is rejected, it throws an error that we can catch and then handle. 

```jsx
async function doStuff () {
    try {
        const foo = await Promise.reject('No room');
    } catch (error) {
        console.log(error);
    }
    console.log('Error handled!');
};

doStuff();
```

If we run this code, we get the following output:

```markdown
No room
Error handled!
```

With `async/await`, there’s no need for `Promise.catch` statement or providing a `onFailure` callback to `then`.

# Creating a Promise-based API

More often than not, you’ll be consuming promise-based API rather than creating promises yourself. However it’s still valuable to know how to implement them.

A promise is created by instantiating the `Promise` class that takes a callback as an argument. In turn, that callback takes two functions as arguments:

- `resolve`, which takes a single value and you should invoke when the promise should be resolved. The argument you pass to it will be the data the promise should return when it succeeds.
- `reject`, which takes a single value and you should invoked when the promise should be rejected. The argument you pass to it will be a reason the promise should return when it fails. You can pass any value, however it’s conventional to pass an instance of `new Error`.

```jsx
new Promise((resolve, reject) => {
	// call resolve if promise succeeds
  // call reject if promise fails
}); 
```

## A Bit More About the `then` Method

We already discussed we call the `then` method on a promise in order to handle the success and possible failure. We also know `then` takes two functions as arguments: the first callback is invoked when the promise is fulfilled and the second callback is invoked when the promise rejects.

What we didn’t know is that `then` always returns a promise, meaning whatever is returned from of its callback arguments is wrapped in a promise. However it depends on the return value:

1. If the callback returns a number, string, etc., `then` will automatically transform this number, string, etc. to a fulfilled promise with a value of this number, string, etc.
    
    ```jsx
    Promise.resolve(2)
    .then((result) => {
        console.log(`result is ${result}`);
    		// then transforms this into a fulfilled promise
        return result * result;
    })
    .then((result) => {
        console.log(`result squared is ${result}`)
    });
    ```
    
2. If the callback returns a promise, `then` simply returns the promise as-is.
    
    ```jsx
    Promise.resolve('hello')
    .then((message) => {
        console.log(`Message: ${message}`);
    		// then returns this promise
        return Promise.resolve(Math.floor(Math.random() * 100));
    })
    .then((luckyNumber) => {
        console.log(`Lucky number (0-99): ${luckyNumber}`);
    });
    ```
    
3. If the callback returns `undefined`, `then` returns a fulfilled promise with a value set to `undefined`. All functions in JS without an explicit `return` statement return `undefined`.
    
    ```jsx
    Promise.resolve('hi')
    .then((greeting) => {
        console.log(greeting);
    		// then returns a promise with value set to undefined
    })
    .then((result) => {
        console.log(result);
    });
    ```
    
4. If a callback throws an error, `then` returns a rejected promise with value set to that error.
    
    ```jsx
    Promise.resolve('foo')
    .then((result) => {
        console.log(result);
        throw 'Something bad happened';
        console.log('This is not reached');
    		// then returns a promise set to the thrown error
    })
    .catch((error) => {
        console.log(error)
    });
    ```
    

Keep in mind that errors travel down the promise chain and they're handled by:

- the first `then` statement with a second callback.
- the first `catch` statement.

💡 **Takeaway:** The `then` method always returns a promise, which underlies the concept of **promise chaining**.

# A More Real-World Example Using the Fetch API

Blah

# References

- [https://esdiscuss.org/topic/does-async-await-solve-a-real-problem](https://esdiscuss.org/topic/does-async-await-solve-a-real-problem)
- [https://thomashunter.name/posts/2015-06-30-the-long-road-to-asyncawait-in-javascript](https://thomashunter.name/posts/2015-06-30-the-long-road-to-asyncawait-in-javascript)
- [https://thomashunter.name/posts/2013-04-27-the-javascript-event-loop-presentation](https://thomashunter.name/posts/2013-04-27-the-javascript-event-loop-presentation)
- [https://wttech.blog/blog/2021/a-brief-history-of-asynchronous-js/](https://wttech.blog/blog/2021/a-brief-history-of-asynchronous-js/)
- [https://developer.okta.com/blog/2019/01/16/history-and-future-of-async-javascript](https://developer.okta.com/blog/2019/01/16/history-and-future-of-async-javascript)
- [https://thomashunter.name/posts/2015-06-30-the-long-road-to-asyncawait-in-javascript](https://thomashunter.name/posts/2015-06-30-the-long-road-to-asyncawait-in-javascript)

[Asynchronous Programming in JS](https://www.notion.so/Asynchronous-Programming-in-JS-cf0ac3db41314ec08bd07d80f8e4d747?pvs=21)

[Async in JS ](https://www.notion.so/Async-in-JS-1dd8a9299b5e4e228c97e07e7910477a?pvs=21)