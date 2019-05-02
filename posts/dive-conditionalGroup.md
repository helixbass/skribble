# A conceptual dive into the weeds of Prettier's conditionalGroup()

I think this will happen:
```
outer(setTimeout(function() {
  ...
}, 100), moreArgs)
```
Fix: propagate breaks through "flat" version of conditional groups

Want to be able to check if *last* line of conditional group fits or not
eg we may not want this:
```
setTimeout(function() {
  b;
}, aaaaaaaaaaaaaaaaaaa
  .bbbbbbbbbbbbbbbbbbbbbbb.cccccccccccccccccccccccccccccccccccccccccccccccc);
```
but instead say "the conditional option where we only break the first arg doesn't fit, so break all the args":
```
setTimeout(
  function() {
    b;
  },
  aaaaaaaaaaaaaaaaaaa.bbbbbbbbbbbbbbbbbbbbbbb
    .cccccccccccccccccccccccccccccccccccccccccccccccc // or however this would actually get formatted
);
```

`shouldRemeasure` isn't sufficient

Happens to work ok for break-first-call-arg conditional group b/c there's only two args and the first
one (function) always has a hardline at the end of the block, which triggers `shouldRemeasure = true` heading
into the second arg

But if it allowed three args, the second arg could have a group that resets `shouldRemeasure` and the third arg
could then have a group that would be `MODE_FLAT` and `shouldRemeasure === false`, so it wouldn't break even if it should

The problem is `shouldRemeasure` as a single "global" flag

`shouldRemeasure` is only necessary for conditional groups (b/c for non-conditional groups, a hardline would propagate
and the groups containing it would always break)

So conceptually what we want is for `shouldRemeasure: true` to be "attached" to all "top-level" (meaning non-nested
within another group, they could be nested inside eg a concat() which is top-level in the conditional group option) groups
inside a conditional group (ie they should always check if they fit or not)

- what about `fill()`? does it use `fits()`? (track uses of `fits()`)

Implementation:

a new mode seems like the best fit conceptually - `MODE_FLAT_REMEASURE`?

otherwise I guess you'd have to add another field to `cmds` to track it?
