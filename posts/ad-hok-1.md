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
  )
  
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
