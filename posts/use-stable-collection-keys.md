When rendering a list of things in React, we use `key` to help React keep track of which item is which (ttem _identity_) over time.
We try and find things that uniquely identify each item in the list and use that for the `key` value. But sometimes it can be hard to
find a suitable unique `key` value for all the items in the list

## The downside of using array index for `key`

One option that always seems appealing is to just use the array index of each item as the `key` - at any given moment, the array index of
an item should uniquely identify it relative to all of the other items, right? That is true, but the problem becomes clear if you look at how
your list of items may change over time:

Let's say we have a todo list app and we're trying to render the current list of todo's
