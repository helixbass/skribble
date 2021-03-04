# Multi-step timing animations in React Native with Reanimated v2

Let's take a look at how to use [Reanimated v2](https://docs.swmansion.com/react-native-reanimated/)
to create multi-step "sequence" timing animations in React Native

Let's create a pseudo-"transition" where a circle expands to fill up the screen and then we slide in
some content over it. Here's the finished result:

[screencap]

So conceptually, we want to:
1. Scale up the circle large enough so that it fills the entire screen
1. 

This is roughly the sequence we want, but we may want to play around with things like [easings](https://easings.net/)
and how much time there is between steps

"Out of the box", Reanimated gives us a building block with the [`withTiming()`](https://docs.swmansion.com/react-native-reanimated/docs/api/withTiming/)
helper, but it doesn't really give us any higher-level abstractions for coordinating multiple timing steps

If you've done timing-sequence animations for the web before and used [GSAP](https://greensock.com/gsap/),
you probably know that its "timeline" abstraction is extremely helpful for specifying the relative timings
in a flexible and intuitive way

So let's create our own similar abstraction to allow us to easily set up and adjust the steps of our animation
