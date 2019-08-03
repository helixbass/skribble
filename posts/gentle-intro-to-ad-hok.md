# A gentle introduction to using `ad-hok` for React hooks

You may have already heard about [React hooks](https://reactjs.org/docs/hooks-intro.html) or started using them
in your React components. Fundamentally, hooks enable you to write (function components)[https://reactjs.org/docs/components-and-props.html#function-and-class-components]
that access features like React state that previously required using a class component.

In this guide we're going to look at an alternative way of structuring your function components using the
[`ad-hok`](https://github.com/helixbass/ad-hok) library. With `ad-hok`, we'll write components in a
[functional composition](https://martinfowler.com/articles/collection-pipeline/)-based style.
Don't worry if you're not already familiar with functional programming, we'll start from the basics and show
you how to think about building up React components out of the small "building block" helpers that `ad-hok` provides.

### How `flow()` works

The common [`Lodash`](https://lodash.com/) utility library actually includes useful helpers for expressing things in a more
functional style which you can import from [`lodash/fp`](https://github.com/lodash/lodash/wiki/FP-Guide).
Let's look at how to convert an example using regular Lodash helpers to use `lodash/fp` instead.

Say we fetch JSON data from an API that describes different dog breeds. We want to print a comma-separated list of all of
the breeds that don't shed e.g. `"Bichon Frise, Dachsund, ..."`. Here's one way we might write this using Lodash:
```js
import {filter, join} from 'lodash'

const dogBreedsData = fetchDogBreedsData()
const dontShed = filter(dogBreedsData, dogBreed => !dogBreed.sheds)
const commaSeparated = join(dontShed, ', ')
console.log(commaSeparated)
```
That works. But notice how it's very "imperative" - at each step, we have to define a new local variable to keep track of
the computation so far. Here it may not be too bad to keep track of what's going on, but especially as logic gets more complex
there are big [readability advantages](https://www.yegor256.com/2015/09/01/redundant-variables-are-evil.html) to minimizing the
number of local variables being used

Basically, it's really hard to keep track in your head of "which variable gets used where" as you're trying to read code and
understand what's going on. So we're going to help ourselves out by using a style where it's more obvious what the dependencies
and flow of data are as you go through the code.

So let's roll up our refactoring sleeves. First, let's extract the part that gets the comma-separated list from the JSON data
into a separate helper function:
```js
import {filter, join} from 'lodash'

const getCommaSeparatedDontShed = jsonData => {
  const dontShed = filter(jsonData, dogBreed => !dogBreed.sheds)
  const commaSeparated = join(dontShed, ', ')
  return commaSeparated
}

const dogBreedsData = fetchDogBreedsData()
const commaSeparatedDontShed = getCommaSeparatedDontShed(dogBreedsData)
console.log(commaSeparated)
```
Ok. We still have a fair amount of local variables going on. If you take that rule of thumb ("minimize local variables") to the
extreme inside our helper function, that would mean zero local variables - what would that look like?

```js
import {filter, join} from 'lodash'

const getCommaSeparatedDontShed = jsonData =>
  join(filter(jsonData, dogBreed => !dogBreed.sheds), ', ')

const dogBreedsData = fetchDogBreedsData()
const commaSeparatedDontShed = getCommaSeparatedDontShed(dogBreedsData)
console.log(commaSeparated)
```
Here, we "inlined" the `filter()` step and then implicitly returned the result of `join()` by using an ES6 arrow function
[expression body](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#Function_body)
(we got rid of the curly braces).

In general, that's a good sign
