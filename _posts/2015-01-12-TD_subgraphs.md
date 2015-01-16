---
layout: post
title:  "Timely dataflow: subgraphs"
date:   2015-01-12 16:00:00
categories: dataflow naiad
published: false
---

This is the third post about a re-think of timely dataflow implementations. In the previous post we covered the interface that nested scopes would need to support in order to communicate about progress. The plan for this post is to sketch out the details of what the implementation of a general-purpose `Subgraph` type would need to look like in order to host other `Scope` objects.

## Preliminaries: Timestamps and PathSummaries

Before talking about the `Subgraph` type itself, there are a few important first steps. Firstly, our goal is to define a type which implements the `Scope<T,S>` trait, but which can host subscopes of types implementing `Scope<T2,S2>`. This isn't going to work for arbitrary types `T` and `T2`, `S` and `S2`.

Rather than rationalize the restrictions, I'll just state them, and then we can talk about whether they are good ideas, bad ideas, or whether there is a proper formal basis for the choices.

* `T2` must be a pair `(T, TInner)` for some `TInner: PartialOrd`.
* `S2` must be `Summary<S1, SInner>`, for some `SInner: PathSummary<TInner>`.

The `Summary` type is something we get to define, which is good. We could also have done the same for timestamps, but pairing works out to be what I think we want anyway.

Our implementation of `Summary` is just an enumeration, representing the cases where a path either lies wholly within the scope (the `Local` case) or may leave the scope and re-enter (the `Outer` case). The `Local` case just needs to know what changes in `TInner`, whereas the `Outer` cases needs to change `T` and then supply a new `TInner`. Apparently I've done that last bit with a `SInner`.

{% highlight rust %}
enum Summary<SOuter, SInner> {
    Local(SInner),         // reachable within scope.
    Outer(SOuter, SInner), // unreachable within scope, but
                           // reachable through outer scope.
}
{% endhighlight %}
The important distinction between these cases is that when a path leaves the scope it strips off its `TInner` coordinate. Any `SInner` component (from either a `Local` or an `Outer`) is discarded when `followed_by` an `Outer` path summary, as whatever has happened to `TInner` is forgotten.

It is important that, by doing this, we are certain that `Outer` summaries are always longer than `Local` summaries; this is where the structure of timely dataflow graphs comes in: any path from the output of the scope back to its inputs must involve a vertex that strictly advances the `T` coordinate.

**Open Question**: Did I need a `SInner` in `Outer`, or should it just be `Outer(SOuter, TInner)`?

Looking at the code, I think it would probably be fine, but the idea of adding another generic parameter to `Summary` and changing all that code is terrifying. Viewed differently, perhaps the types `T` and `TInner` are spurious, and we only need `SOuter` and `SInner`, where every instance of a timestamp is really just a path summary applied to a default starting timestamp? Let's do it this way for now.

I'm going to skip describing the `results_in` and `followed_by` methods, which apply a summary to a timestamp and another summary, respectively. They do what you would expect, either extending or discarding a `TInner`/`SInner`, and either maintaining or extending the `T`/`SOuter`.

## The Subgraph type

This is where the brain hurt begins. We're going to look at a super-simplified version of the `Subgraph` type, and it is still going to be a bit confusing. I've elided all sorts of buffers and such, so here goes:

{% highlight rust %}
struct Subgraph<TOuter, SOuter, TInner, SInner>
where TOuter: Timestamp,
      SOuter: PathSummary<TOuter>,
      TInner: Timestamp,
      SInner: PathSummary<TInner>
{
    inputs:         u64,                        // number inputs into the scope
    outputs:        u64,                        // number outputs from the scope

    children:       Vec<Box<Scope<(TOuter, TInner), Summary<SOuter, SInner>>>>,
    edges:          Vec<Vec<Vec<Target>>>,

    ext_summaries:  Vec<Vec<Antichain<SOuter>>>,
    int_summaries:  Vec<Vec<Vec<(Target, Antichain<Summary<SOuter, SInner>>)>>>,

    ext_capability: Vec<MutableAntichain<TOuter>>,
    ext_guarantee:  Vec<MutableAntichain<TOuter>>,
    input_messages: Vec<Rc<RefCell<Vec<((TOuter, TInner), i64)>>>>,

    progcaster:      Progcaster<(TOuter, TInner)>,
}
{% endhighlight %}


{% highlight rust %}
pub enum Target {
    GraphOutput(u64),      // to outer scope
    ScopeInput(u64, u64),  // (subscope, port)
}
{% endhighlight %}
