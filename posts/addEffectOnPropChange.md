# Handy React hooks helper: `useEffectOnPropChange()`

### The old way: "prop-diffing" in `componentDidUpdate()`

In the days before React hooks, a typical pattern would be to compare the previous and new values of a certain prop in a
lifecycle method like `componentDidUpdate()` and conditionally "do something" if that prop had changed values in some
particular way

For example:
```
class Watcher extends React.Component {
  ...
  componentDidUpdate(prevProps) {
    if (this.props.foo !== prevProps.foo && this.props.foo === 'bar') {
      console.log("foo just changed to 'bar'")
    }
  }
}
```
This type of "prop-diffing" always felt pretty boilerplate-y/hard to read - you're sort of intermixing the mechanics of
inspecting for a certain type of prop change having taken place with the action that should happen if that condition is met

### The new standard: `useEffect()`

These days, using class component lifecycle methods can typically be replaced by React hooks, in this case primarily
[`useEffect()`](https://reactjs.org/docs/hooks-effect.html)

The React hooks docs show us the [basic pattern for getting a reference to the previous props](https://reactjs.org/docs/hooks-faq.html#how-to-get-the-previous-props-or-state):
```
import {useRef, useEffect} from 'React'

function usePrevious(value) {
  const ref = useRef()
  useEffect(() => {
    ref.current = value
  })
  return ref.current
}

const Compare = props => {
  const prevProps = usePrevious(props) || {}
  return <h1>Previous foo: {prevProps.foo}, current foo: {props.foo}</h1>
}
```
You can see that the `usePrevious()` helper is using the `useRef()` hook to track the previous value of something (in this case,
the props passed to the `<Compare>` component)

So with `usePrevious()` as a building block, we can create an abstraction that allows us to say "when some particular prop
changes, do something":
```
import {find} from 'lodash'

const useEffectOnPropChange = (changeProps, callback) => props => {
  const prevProps = usePrevious(props) || {}
  useEffect(() => {
    const changed = find(
      ensureArray(changeProps),
      changeProp => prevProps[changeProp] !== props[changeProp]
    )
    if (!changed) return null
    return callback(props, prevProps)
  })
}

const Watcher = props => {
  useEffectOnPropChange('foo', ({foo}, prevProps) => {
    if (foo === 'bar' && prevProps.foo === 'baz') {
      console.log("foo just changed to 'bar' from 'baz'")
    }
  })(props)
  ...
}
```
Here `ensureArray()` is a helper that allows the first argument to `useEffectOnPropChange()` to either be an array of prop
names or a single prop name:
```
import {isArray} from 'lodash'

const ensureArray = maybeArray =>
  isArray(maybeArray) ? maybeArray : [maybeArray]

...
// both of these calling styles work:
useEffectOnPropChange('foo', ...)(props)
useEffectOnPropChange(['foo', 'bar'], ...)(props)
```
So with `useEffectOnPropChange()` we've given ourselves a more declarative way to indicate that something should happen
whenever a certain prop changes value

This is a lot nicer than manually "prop-diffing" inside `componentDidUpdate()`, but I think there's room to make it even more
declarative. Basically now `useEffectOnPropChange()` is declarative with respect to *which prop we're watching* but not with
respect to *under which conditions we care* - often times (like in the example above), our condition is actually more specific
than just "the `foo` prop changed", it's actually eg "the `foo` prop went from being null to non-null" or "the new value of the
`foo` prop is `'bar'`"

I can picture a couple different ways of making those cases more declarative:
```
// Maybe these common cases deserve their own name:
useEffectOnPropBecomingTruthy('foo', ...)(props)
useEffectOnPropBecomingFalsy('foo', ...)(props)

// Here's one calling interface for specifying the prop change condition separately from the action:
useEffectOnPropChange({foo: (currentFoo, prevFoo) => currentFoo === 'bar' && prevFoo === 'baz'}, ...)(props)

// Or similarly:
useEffectOnPropChangeWhen((props, prevProps) => props.foo === 'bar' && prevProps.foo === 'baz')('foo', ...)(props)
```
These might all be nice ways to express a condition in different situations - the goal is to feel empowered to create the
nice abstractions that you (and others) can then reach for when you encounter a situation where they'd come in handy!

For example, here's how I might implement `useEffectOnPropBecomingTruthy()`:

First, I think we're going to need a general way to "inject the condition" into `useEffectOnPropChange()`. Looking at the
ideas for different calling interfaces above, I think it'll make sense to make `useEffectOnPropChangeWhen()` our underlying
abstraction that all the rest (including `useEffectOnPropChange()`) are built on top of

```
const useEffectOnPropChangeWhen = (condition = () => true) => (changeProps, callback) => props => {
  const prevProps = usePrevious(props) || {}
  useEffect(() => {
    const changed = find(
      ensureArray(changeProps),
      changeProp => prevProps[changeProp] !== props[changeProp]
    )
    if (!changed) return null
    if (!condition(props, prevProps)) return null
    return callback(props, prevProps)
  })
}
```
We basically just added another "level of currying" to our previous `useEffectOnPropChange()` implementation in order to
accept the additional `condition`

So now we can reimplement `useEffectOnPropChange()` in terms of `useEffectOnPropChangeWhen()`:
```
const useEffectOnPropChange = (changeProps, callback) =>
  useEffectOnPropChangeWhen()(changeProps, callback)
  
// or just:
const useEffectOnPropChange = useEffectOnPropChangeWhen()
```

And now we can implement variations like `useEffectOnPropBecomingTruthy()` also using `useEffectOnPropChangeWhen()`:
```
const useEffectOnPropBecomingTruthy = (changeProps, callback) => {
  const arrayOfChangeProps = ensureArray(changeProps)
  const condition = (props, prevProps) =>
    find(arrayOfChangeProps, changeProp => props[changeProp] && !prevProps[changeProp])
  return useEffectOnPropChangeWhen(condition)(changeProps, callback)
}
```

### Bonus: use `ad-hok`

These are all nicely declarative ["custom hooks"](https://reactjs.org/docs/hooks-custom.html). But I typically use
[`ad-hok`](https://github.com/helixbass/ad-hok) to allow composing hooks like this more fluidly

It's trivial to convert these custom hooks into `ad-hok`-style helpers:

```
const addEffectOnPropChange = (changeProps, callback) => {
  const curriedUseEffectOnPropChange = useEffectOnPropChange(changeProps, callback)
  return props => {
    curriedUseEffectOnPropChange(props)
    return props
  }
}
```
or written in a more functional style:
```
import {flow} from 'lodash/fp'

const addEffectOnPropChange = flow(
  useEffectOnPropChange,
  tap
)
```
where `tap()` is a [standard](https://ramdajs.com/0.21.0/docs/#tap) [functional](https://apidock.com/ruby/Object/tap) [helper](http://raganwald.com/JQuery-Combinators/):
```
const tap = callback => val => {
  callback(val)
  return val
}
```
The above is elegantly simple but might be hard to visualize, so here's a version where the flow of arguments is more explicit:
```
const addEffectOnPropChange = flow(
  (changeProps, callback) => useEffectOnPropChange(changeProps, callback),
  curriedUseEffectOnPropChange => {
    return props => {
      curriedUseEffectOnPropChange(props)
      return props
    }
  }
)
```

Then we can use `addEffectOnPropChange()` in an `ad-hok`-style component:
```
const Watcher = flow(
  addEffectOnPropChange('foo', ({foo}, prevProps) => {
    if (foo === 'bar' && prevProps.foo === 'baz') {
      console.log("foo just changed to 'bar' from 'baz'")
    }
  }),
  ...
)
```

### Practical use case: tracking current language

In the current project I'm working on, we're using [`i18next`](https://www.i18next.com/)/[`react-i18next`](https://react.i18next.com/)
for internationalization support (translations)

It seems as though `i18next` expects to be the "source of truth" for what the currently selected language is. But that felt
somewhat at odds with our Redux store being the source of truth for most similar state. And at some point I felt like I was
seeing wonky/flaky updating of `i18next`'s version of what the currently selected language is

So at that point I decided to move the current language "source of truth" into our Redux store and have `i18next`'s idea of
the current language be informed by our Redux store

There may be a more idiomatic way of "broadcasting" a value from your Redux store, but the way I achieved this was to create
a component (that gets mounted at the root level of our application) whose sole purpose is to listen to the Redux store for
changes to the current language and inform `i18next` of that change:
```
import {connect} from 'react-redux'
import {flowMax, renderNothing} from 'ad-hok'
import i18n from 'i18next'
import PropTypes from 'prop-types'

import {languageSelector} from 'redux-local/selectors'
import addEffectOnPropChange from 'util/addEffectOnPropChange'

const enhance = connect(languageSelector)

const WatchLanguage = flowMax(
  ...
  addEffectOnPropChange('currentLanguage', ({currentLanguage}) => {
    i18n.changeLanguage(currentLanguage)
  }),
  renderNothing()
)

WatchLanguage.propTypes = {
  currentLanguage: PropTypes.string.isRequired,
}

export default enhance(WatchLanguage)
```

`addEffectOnPropChange()` makes it super easy to fire off the call to `i18n.changeLanguage()` whenever our Redux store's
`currentLanguage` changes
