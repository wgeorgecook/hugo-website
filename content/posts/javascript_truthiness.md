+++
title = "Javascript_truthiness"
date = "2022-09-23T10:21:08-07:00"
author = "William George Cook"
tags = ["javascript", "bugs", "work"]
keywords = ["javascript", "comparisons", "equality", "types"]
description = "Javascript's Weakness Is It's Biggest Strength"
showFullContent = false
readingTime = false
hideComments = false
+++

## Comparisons in Javascript
While my title may say "Full Stack", I'm really a backend dev who is pretty flexible and willing to hack around the front end when necessary. This can sometimes lead to me falling for some Javascript gotchas (see also [Python None]{{< ref "python_none" >}}). This week in Adventures With Javascript, I got to scratch my head on a bug where a form was falling back to a default value even when a valid input was entered. 


### A Flexible Type System
One of Javascripts biggest flaws is actually one of it's biggest strengths. Since Javascript does not require strict typing, it's able to compare two different types and Javascript will do it's best to coerce them to a shared type. For instance, you can compare strings and integers:
```
const a = 1;
const b = '1';
console.log(a == b);

> true
```

You can compare strings and booleans:
```
const a = false;
const b = '';
console.log(a == b);

> true
```

This flexibility makes it a lot less daunting for new developers. And if you're designing and api but haven't settled on a schema for the data,this system makes prototyping very rapid. However, there can be unintended drawbacks. 

### Type Coercion 
You'll notice in the second example above that an empty string is coerced to a falsy boolean. The same thing happens with the number 0. 
```
const a = false;
const b = 0;
console.log(a == b);

> true
```

Which can cause some issues if you're doing some form validation where 0 is either a valid input, or the user provided input is transformed to 0 for the backend to process. In this example I was coercing a 12 hour AM/PM based user input into a 24 hour based number to send to the backend. 


```
const twelveToTwentyFour = (timeInt, amOrPm) => {
    if (amOrPm == 'PM') {
      // Add 12 to the provided timeInt but don't send 12 PM as 24
      return (timeInt === 12) ? 12 : timeInt + 12;
    } else {
      // Ensure we don't send 12 AM as 12
      return (timeInt === 12) ? 0 : timeInt;
    }
}
```

### Bug Hunting
If the user passes in 12AM to this function, we transform it to 0 before we send it to the backend. However, this caused issues when we try to validate the form and fill in default values. 

```
 const start = twelveToTwentyFour(12, 'AM') || 8;
 console.log(start);

 > 8
```

12AM is a valid time, so why are we getting the default value set on the form? Type coercion comes to help in here because we're doing the `or` operation here to validate the input. 

```
const a = 0;
const b = 8;
console.log(a || b);

> 8
```

You can verify that 0 here is being interpreted as a falsy value but comparing against a falsy boolean:
```
const a = 0;
const b = false;
console.log(a == b);

> true
```

### Bug Squishing
So here we are! We identified the bug. But how do we fix it? Luckily, Javascript has a way to do _strict_ type checking. Substitute that double equals (`==`) for a triple equals (`===`) and you'll get more expected results. 

```
const a = 0;
const b = false;
console.log(a === b);

> false
```

We need to change our original valiation a little. Prior to user input, our start value is `undefined`. Using a strict equality check on `undefined` we can determine if we need to set this to the user provided input, or default it to 8.

```
const parsed = twelveToTwentyFour(12, 'AM')
const start = (parsed === undefined) ? 8 : parsed;
console.log(start);

> 0
```

Calling the function without any arguments validates this for us. 
```
const parsed = twelveToTwentyFour()
const start = (parsed === undefined) ? 8 : parsed;
console.log(start);

> 8
```

## How Many Equals?
If you're like me and used to programming in strictly typed systems, some Javascript niceties like the triple equals can slip your mind. Sometimes you're dealing with component libraries and the return values from those components don't match what your API require. However, unless you have a really good reason to use a loose comparison, use the strict equals operator and avoid spending half a day dealing with problems like this. If you want to explore how the loose equality operator handles these type comparisons, [this table](https://dorey.github.io/JavaScript-Equality-Table/) shows exactly what comparisons are truthy and which are not. 
