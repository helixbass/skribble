# Avoid the clunky parts of React hooks with ad-hok: early return

Frequently it's natural to think of our components as "bailing out early" when rendering, e.g. based on a loading state

With React hooks, we're forced to depart from the intuitive way of expressing this logic in order to avoid violating the
[rules of hooks](https://reactjs.org/docs/hooks-rules.html)

For example, this is forbidden by the rules of hooks:
```js
const TodaysMatches = () => {
  const {data, loading} = useMatchesQuery()
  
  if (loading) {
    return <LoadingSpinner />
  }
  
  const {t} = useTranslation()
  
  const todayMatches = useMemo(() =>
    sortBy('startTime', data.matches.filter(match => isToday(match))
  , [data])
  
  return (
    <div>
      <h1>{t('todaysMatches')}</h1>
      <ul>
        {todayMatches.map(todayMatch => <Match match={todayMatch} key={todayMatch.id} />)}
      </ul>
    </div>
  )
}
```

Why? Because we're not allowed to call any hooks after the "early return" of `<LoadingSpinner />`

So then we're left in the uncomfortable situation of having to restructure our code to something *less* intuitive
while making sure it still behaves the way we want it to

One option is to "push down" the bail-out to after all of the hooks have been called:

```js
const TodaysMatches = () => {
  const {data, loading} = useMatchesQuery()
  
  const {t} = useTranslation()
  
  const todayMatches = useMemo(() =>
    sortBy('startTime', (data ? data.matches : []).filter(match => isToday(match))
  , [data])

  if (loading) {
    return <LoadingSpinner />
  }

  return (
    <div>
      <h1>{t('todaysMatches')}</h1>
      <ul>
        {todayMatches.map(todayMatch => <Match match={todayMatch} key={todayMatch.id} />)}
      </ul>
    </div>
  )
}
```
We had to guard against `data.matches` not being present, since we're no longer sure that we're on the "happy path"
when calculating today's matches

That isn't tragic, but there's mental overhead (as the programmer or the reader) when you have to track multiple
states simultaneously through a single code path ("How would this behave when we're in "loading" state? How would it
behave when we're in "loaded" state?")

This should feel artificial to you, dear reader. We're being forced to structure our code to appease the *technology*,
which React itself helped us realize is a [bad idea](https://www.youtube.com/watch?v=x7cQ3mrcKaY). A total smell.

Whereas in ad-hok, we're able to write the logic the way we wanted to, using a nice declarative style:

```js
const TodaysMatches = flowMax(
  addMatchesQuery(),
  branch(({loading}) => loading, returns(() => <LoadingSpinner />)),
  addProps(({data: {matches}}) => ({
    todayMatches: sortBy('startTime', matches.filter(match => isToday(match)),
  }), ['data.matches']),
  addTranslationHelpers,
  ({todayMatches, t}) => (
    <div>
      <h1>{t('todaysMatches')}</h1>
      <ul>
        {todayMatches.map(todayMatch => <Match match={todayMatch} key={todayMatch.id} />)}
      </ul>
    </div>
  )
)

// these "thin wrapper" helpers would typically already be defined elsewhere:

const addMatchesQuery = () =>
  addProps(() => {
    const {data, loading} = useMatchesQuery()
    return {data, loading}
  })
  
const addTranslationHelpers = addProps(() => {
  const {t} = useTranslation()
  return {t}
})
```
Not only did we get to "bail out early" like we wanted to, we're also now in a position to easily abstract out
that common pattern:
```js
const TodaysMatches = flowMax(
  addMatchesQuery(),
  branchIfLoading,
  addProps(({data: {matches}}) => ({
    todayMatches: sortBy('startTime', matches.filter(match => isToday(match)),
  }), ['data.matches']),
  addTranslationHelpers,
  ({todayMatches, t}) => (
    <div>
      <h1>{t('todaysMatches')}</h1>
      <ul>
        {todayMatches.map(todayMatch => <Match match={todayMatch} key={todayMatch.id} />)}
      </ul>
    </div>
  )
)

const branchIfLoading = 
  branch(({loading}) => loading, returns(() => <LoadingSpinner />))
```
And perhaps even bake the "bailing out" into our query helpers by default:

```js
const TodaysMatches = flowMax(
  addMatchesQuery(),
  addProps(({data: {matches}}) => ({
    todayMatches: sortBy('startTime', matches.filter(match => isToday(match)),
  }), ['data.matches']),
  addTranslationHelpers,
  ({todayMatches, t}) => (
    <div>
      <h1>{t('todaysMatches')}</h1>
      <ul>
        {todayMatches.map(todayMatch => <Match match={todayMatch} key={todayMatch.id} />)}
      </ul>
    </div>
  )
)

const addMatchesQuery = ({shouldBranchIfLoading = true} = {}) => flowMax(
  addProps(() => {
    const {data, loading} = useMatchesQuery()
    return {data, loading}
  }),
  shouldBranchIfLoading ? branchIfLoading : props => props
)
```

Those, dear reader, are *good smells*



<!--
But both of these are inconvenient for the programmer and the reader - our mental model is that the "loading" path only
depends on the `loading` state, but we're not being allowed to directly express that and "move on" to the core "happy path"
logic.
-->
