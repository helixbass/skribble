# A conceptual dive into the weeds of Prettier's conditionalGroup()

I think this will happen:
```
outer(setTimeout(function() {
  ...
}, 100), moreArgs)
```
Fix: propagate breaks through "flat" version of conditional groups

Ok the reason it doesn't is that there's an explicit `somePrintedArgumentsWillBreak ? breakParent : ""` (for some fascinating related early Prettier history, see [#559](https://github.com/prettier/prettier/pull/559) [#496](https://github.com/prettier/prettier/pull/496))

The problem with this is that `willBreak()` is *static* - it only works here because there are `hardline`s when formatting a block (here, the function body). If those were eg `line`s that did in fact end up breaking, you'd get the formatting above since it wouldn't know statically that it should propagate the break to the parent

So how should it propagate a soft break like that to the parent (across the `conditionalGroup()` "boundary")?

The only way is through measuring/`fits()`, since the parent figures out whether it breaks before its child does

So here `outer()` should say "if I encounter a conditional group whose flat version doesn't *all* fit on one line,
I don't fit" (as opposed to "if I hit a `hardline` before I've run out of space, I fit")

At that point the manual `breakParent` *should* be unnecessary?

Ok trying to get conditional groups to check if their last line fits -
was seeing things like:
```
f(a => {
  b;
},
c);
```
Looking at the doc associated with the conditional group option for this "should group first" case, I was confused as to why
it wasn't always doing this, since the group containing the first arg and comma is marked `break: true` (since the hardline of the block propagated out to this wrapping group) and it's just a `line` after the comma

Turns out it's not actually ever rendering the `shouldGroupFirst` version of that conditional group option!
Why? because it thinks the flat (first conditional group option) version fits!
So by changing the algorithm to check that the last line fits, it was (correctly?) deciding that the flat version
doesn't fit and was actually rendering the `shouldGroupFirst` version, which is broken - in order to achieve that
behavior with the space after the comma, I guess you'd need a different version of that group-with-the-comma that
knows that it's inline so doesn't use a `line`?

-----------------------------------------

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

The way that this happens:
```
map(row => [
  eee
]);
```
is that when it's checking if the conditional group option for "break last arg" for the arguments fits,
the `shouldBreak: true` from the `group()` in the conditional group option "propagates" to the `[...]` group
because of this line (in `fits()`):
```
cmds.push([ind, doc.break ? MODE_BREAK : mode, doc.contents]);
```
ie when `fits()` is checking a `group()` which is `MODE_BREAK` (which only happens inside a conditional group),
it keeps measuring all further nested groups in `MODE_BREAK`!
So what you're really measuring is if the first line fits when breaking every single group you encounter.
And you have no control over whether it should break every single group like this, that's effectively hardcoded
into the algorithm
