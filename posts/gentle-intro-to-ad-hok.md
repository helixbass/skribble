# A gentle introduction to `ad-hok`

You may have already heard about [React hooks](https://reactjs.org/docs/hooks-intro.html) or started using them
in your React components. Fundamentally, hooks enable you to write (function components)[https://reactjs.org/docs/components-and-props.html#function-and-class-components]
that access features like React state that previously required using a class component.

In this guide we're going to look at an alternative way of structuring your function components using the
[`ad-hok`](https://github.com/helixbass/ad-hok) library. With `ad-hok`, we'll write components in a
[functional composition](https://martinfowler.com/articles/collection-pipeline/)-based style.
Don't worry if you're not already familiar with functional programming, we'll start from the basics and show
you how to think about building up React components out of the small "building block" helpers that `ad-hok` provides.

### How `flow()` works

The common [`lodash`](https://lodash.com/) utility library includes useful helpers for expressing things in a more
functional style which you can import from [`lodash/fp`](https://github.com/lodash/lodash/wiki/FP-Guide).
Let's look at how to convert an example using regular Lodash helpers to use `lodash/fp` instead.

##### Example: Dog breeds

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
the computation so far. Here it may not be too bad to understand what's going on, but especially as logic gets more complex
there are big [readability advantages](https://www.yegor256.com/2015/09/01/redundant-variables-are-evil.html) to minimizing the
number of local variables being used.

Basically, it's really hard to keep track in your head of "which variable gets used where" as you're trying to read code. So we're going to help ourselves out by using a style where it's more obvious what the dependencies
and flow of data are as you go through the code.

##### Start refactoring

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
console.log(commaSeparatedDontShed)
```
Ok. We still have a fair amount of local variables going on. If you take that rule of thumb ("minimize local variables") to the
extreme inside our helper function, that would mean zero local variables - what would that look like?

```js
import {filter, join} from 'lodash'

const getCommaSeparatedDontShed = jsonData =>
  join(filter(jsonData, dogBreed => !dogBreed.sheds), ', ')
```
Here, we "inlined" the `filter()` step and then implicitly returned the result of `join()` by using an ES6 arrow function
[expression body](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#Function_body)
(we got rid of the curly braces).

In general, that's a good sign - in functional programming, describing computations using expressions === :heart:. But here,
that wasn't necessarily an improvement. We'd like to be able to read the code left-to-right and top-to-bottom, but here we sort of have to read
"inside out" to understand what's going on - first, the nested `filter()`, then the outer `join()`.

So let's go back to the previous version. And I'm going to introduce the "alternative" versions of `filter()` and `join()` from `lodash/fp`:
```js
import {filter, join} from 'lodash/fp'

const getCommaSeparatedDontShed = jsonData => {
  const dontShed = filter(dogBreed => !dogBreed.sheds)(jsonData)
  const commaSeparated = join(', ')(dontShed)
  return commaSeparated
}
```
Ok, let's break down what changed by looking at the `filter()` call:
```js
// lodash version
import {filter} from 'lodash'
filter(jsonData, dogBreed => !dogBreed.sheds)

// lodash/fp version
import {filter} from 'lodash/fp'
filter(dogBreed => !dogBreed.sheds)(jsonData)
```
You can see that the `lodash/fp` version sort of takes the same arguments as the `lodash` version but (a) in reverse order -
the "callback argument" `dogBreed => !dogBreed.sheds` comes first (b) across two separate function calls. This is the gist of what the [`lodash/fp` Guide](https://github.com/lodash/lodash/wiki/FP-Guide) means by "immutable auto-curried iteratee-first data-last methods".

So what's so great about that? We just changed the order of the arguments and took more function calls to get the same result.
Well, notice how now there's a pattern in `getCommaSeparatedDontShed()` where the result of each previous step is the only
argument to the outer function call of the next step:
1. `jsonData` (the input to `getCommaSeparatedDontShed()`) is the outer argument to `filter()`
1. `dontShed` is the outer argument to `join()`
1. We return the final value `commaSeparated`

What you want to start to picture is a "pipeline" where we feed in our input value (`jsonData`), and then each step of the pipeline transforms the value that it receives into a new value that it hands off to the next step in the pipeline. And then
finally the last step in our pipeline "hands off" its transformed value as the return value (`commaSeparated`) of the whole pipeline function (`getCommaSeparatedDontShed()`).

What `lodash/fp` gives us is a helper for managing that pipeline for us: `flow()`. Take a look:
```
import {filter, join, flow} from 'lodash/fp'

const getCommaSeparatedDontShed = flow(
  filter(dogBreed => !dogBreed.sheds),
  join(', ')
)
```
All `flow()` is doing is connecting the steps of the pipeline for you. If it's not clear to you what's going on here, let's start
with a simpler example that illustrates the way it's composing functions:
```js
const addOneThenDouble = flow(
  x => x + 1,
  y => y * 2
)

// 1. `x` gets the value 3, so the first function returns 4
// 2. `y` gets the value 4, so the second function return 8
// 3. The return value of the whole flow() is the return value of the last step, so 8
addOneThenDouble(3) // 8
```
First of all, we can see that calling `flow()` returns a function (which we're calling `addOneThenDouble`). And the arguments to
`flow()` are also functions - each step in the pipeline is a function that accepts an input argument and returns a value.

So that's also what's going on in our `getCommaSeparatedDontShed()` - at each pipeline step, we have a function returned by the
`lodash/fp` helpers `filter()` and `join()`.

The cool part about `flow()` is that lots of typical logic tends to follow this pattern of handing off the result of one step
to the next step. So we can actually also express the other part of our code as a `flow()` as well:
```js
import {filter, join, flow} from 'lodash/fp'

const getCommaSeparatedDontShed = flow(
  filter(dogBreed => !dogBreed.sheds),
  join(', ')
)

flow(
  () => fetchDogBreedsData(),
  dogBreedsData => getCommaSeparatedDontShed(dogBreedsData),
  commaSeparatedDontShed => console.log(commaSeparatedDontShed)
)()
```
And when we see a pattern like `x => func(x)`, that can typically be rewritten as just `func` (since there's no need to define a wrapper function that just forwards the argument `x` on to `func()`). So applying that rule, we could simplify to:
```js
import {filter, join, flow} from 'lodash/fp'

const getCommaSeparatedDontShed = flow(
  filter(dogBreed => !dogBreed.sheds),
  join(', ')
)

flow(
  fetchDogBreedsData,
  getCommaSeparatedDontShed,
  console.log
)()
```
This may look "odd" to you right now, but notice how we've gotten rid of all the local variables - if you can follow how `flow()`
works, it's now quite straightforward to trace the flow of data and execution when reading the code. You might be able to
picture how valuable that can be as a way to organize your code when you're dealing with more complex pieces of logic.

Don't worry if this doesn't "click" right away. Let it sink in and really try and wrap your head around the execution flow (i.e.
when each individual function gets called and what it does). "Thinking functionally" doesn't happen overnight but is an
highly recommended quality as a programmer. For further reading:
- ["Dipping a toe into functional JS with lodash/fp"](https://simonsmith.io/dipping-a-toe-into-functional-js-with-lodash-fp) is a more thorough (yet concise) introduction to `lodash/fp` and related concepts
- [James Sinclair's blog](https://jrsinclair.com/) does a great job of explaining even rather advanced functional programming
topics in simple terms to an audience familiar with JS
