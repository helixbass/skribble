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

As far as the check-last-line algorithm, I *think* it'll actually want to ignore overflows on anything except the last line?
(Does this imply that the check-last-line behavior should be opt-in for conditional groups?)
Seems like the current keep-breaking-nested-groups behavior would usually result in things not overflowing (inside of the
explicitly `shouldBreak: true` groups in the conditional group option)?

Ok implemented where all `fits()` checks (except when checking a conditional group option) check that the *whole* command
that was being checked (for fitting) has been fitted at the time a newline is encountered - if it hasn't all been fitted
(ie there's a linebreak *inside* the thing we were checking whether it all fits on a line), then it says "I didn't fit"

This will only be relevant for stuff with conditional groups nested inside of them, since otherwise any hard lines would
have propagated up and we wouldn't be checking for fitting (since we'd already know that it breaks)

And so this should result in outer groups which contain nested conditional groups always breaking when the conditional
group "breaks" (ie anything besides the first conditional group option in flat mode is used, or the first option in flat mode
contains a hard line?) (getting rid of the need to eg do
the manual `breakParent` discussed above - doing it this way should also cover the case when there's not a *hard* line in the
conditional group? Although actually shouldn't that case be covered when checking `fits()`? I think not necessarily - imagine
if the first conditional group option used a manually broken (`shouldBreak: true`) group - then the current `fits()` check
would say "it fits" because eg a `line` in the `shouldBreak: true` group would end the line, but the manual `breakParent`
wouldn't kick in because there's no `willBreak()` - actually wait `willBreak()` checks for `shouldBreak: true` groups (not
just hard lines)! So I guess that implies that the manual `breakParent` is equivalent to this `fits()` algorithm as far as
making sure that any outer group (enclosing a nested conditional group) will break if the conditional group "breaks". But
we'd want to have the algorithm do that for us rather than needing to manually `breakParent`, right? See discussion in #559,
they agree that this is desired behavior consistent with the way things work in general)

The original motivating thing behind this dive was questioning whether `firstBreakingIndex` was the right abstraction.
The above seems to rectify whether conditional groups appear to break(/need to break) to their *parents*, but I assume I
introduced `firstBreakingIndex` related to visible groups, ie whether it appears to have broken to *children*. I'm guessing
that any issues I was having with that would be a `shouldRemeasure` issue? So would be addressed by the idea below of
something like forcing remeasuring on all "top-level" group descendants of a conditional group option

One big case I saw failing then was YAML objects eg this:
```
Block style: !!map
  Clark: Evans
```
was formatting like:
```
Block style:
  !!map
  Clark: Evans
```
Haven't fully wrapped my head around the relevant YAML AST and formatting rules, but I think what's happening is that the
outer thing (`group()`?) that contains a nested conditional group is deciding that it fits (since I guess it's just measuring
until it hits a hard line in the flat version of the conditional group), and then the conditional group itself is
deciding that it doesn't need to remeasure so it's just blindly using its flat version!

Ok fixed the YAML formatting by passing `isConditionalGroupOption: true` also for the first ("flat") conditional group option
(I should have been doing that)

At that point there was a JSX test failing:
```
br_triggers_expression_break =
  <div><br />
  text text text text text text text text text text text {this.props.type} </div>
```
Turned out that there was a bug in the way `fill()` checks if it fits that
got exposed because with this algorithm it was correctly deciding that something with a hard line breaks, which was
resulting in `shouldRemeasure` not being reset by that hard line (remember `shouldRemeasure` is only supposed to be
necessary for conditional groups, in this instance there were no conditional groups involved, `shouldRemeasure` just
happened to be triggering a remeasure of a fill content group that was incorrectly pushed as `MODE_FLAT` (ie considered to
fit as part of its parent)

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
