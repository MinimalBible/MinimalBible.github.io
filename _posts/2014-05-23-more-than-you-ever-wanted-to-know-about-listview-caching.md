---
layout: post
title: "More than you ever wanted to know about ListView caching"
modified: 2014-05-23 19:05:13 -0400
tags: [listview, viewholder, holderview, caching]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---
More than you ever wanted to know about ListView caching
========================================================
Unless you wanted to know a *lot*
---------------------------------

So, I ran into some interesting issues with `ListView` caching in trying to get everything implemented. I'm going to document them here, because I imagine I'm not the only one to struggle with these, and it could save people a lot of time and frustration later. Hope this is helpful!

Why does HolderView work anyway?
--------------------------------

One of the first questions I had about HolderView was why it actually worked. I mean, if classes are not static any more, they're being garbage collected when they go out of scope. And if they're being garbage collected, I'll have to keep on re-creating them, and I'm back to having the issues the [ViewHolder and HolderView]({% post_url 2014-05-23-viewholder-vs-holderview.md %}) were designed to solve in the first place.

Well, to understand why `HolderView` works you need to know a bit about how the `ListView` operates in the backend. Long story short, when you scroll through elements, the `ListView` recycles things from the top of the screen down to the bottom. Think of it like a [climbing wall treadmill](http://www.hammacher.com/Product/Default.aspx?sku=12219). This way, you don't have to store every single element of the list, and you don't have to create them either.

So, the `getView()` method is where the actual view inflation of a `ListView` happens. The `ListView` supplies us with one of the `View`s it's already had created for us, so we can get right to work. Now, the `ListView` will display whatever we return from `getView()`. So there's nothing against us returning a completely different view, unrelated to the one the `ListView` gave us originally. And that's why this works.

Because the `HolderView` returns it's own custom `View` object, rather than using the `View` that the `ListView` gave us, we can guarantee attributes like the `bind()` method will actually exist when cast to/from `View`/`ItemView`. This way, when the views are recycled by the `ListView`, it's actually giving us the custom `HolderView` views, rather than generic views. All we have to do is re-`bind()` the `HolderView`, and we're good to go. Fancy, no?

Switching to ViewHolder from HolderView
---------------------------------------
Now, let's say that for whatever reason, you want to switch to a `ViewHolder` pattern from a `HolderView`. Not too challenging, but make sure you're thorough about it. [When I tried to do this](https://github.com/MinimalBible/MinimalBible/commit/d664f12d0825201f64c755b2b6ecee26e2169e6b#diff-46af121991fccb227334e34062a21659L50) I wasn't so thorough, and so instead of caching only references to the `ViewHolder`, I just cached the `HolderView` instead. It led to the same bugs I was seeing prior to the switch (of course) and so I was a bit frustrated. I also started getting some NullPointerException errors, which was really strange. Understanding these errors is incredibly technical, so stick with me.

Part of the reason for using `HolderView` at all is memory management. The standard `ViewHolder` pattern uses static classes, meaning they just stay in memory until the actual application is killed. The standard `HolderView` is not a static class, which means that when it goes out of scope, it is destroyed.

Well, in the implementation linked to above, I stored the `HolderView` object into the cache in a kind of hybrid approach. I kept the memory management of `HolderView` while using the `getTag()` style of `ViewHolder` caching. And to be honest, it wasn't until trying to write this post (4 days after the fact) that I understand why it was broken. **Since the `HolderView` gets destroyed when it goes out of scope** (for example, when you scroll past the view), **the tag of the View will be null when you scroll anywhere.** Thus, I had to modify the code so that I check both the view and tag, rather than just the view:

**BookListAdapter.java**
{% highlight java %}
// Note that convertView==null will short-circuit the getTag() check
if (convertView == null || convertView.getTag() == null)
{% endhighlight %}

View recycling leaves views partially initialized
	-------------------------------------------------

Now, this last piece is a very nasty issue I had seen [ever since adding](https://github.com/MinimalBible/MinimalBible/commit/b04d6c67ae0324cfaa12c3d17b2815fe08936658) the initial `ProgressWheel` (took 3 days to get it fixed). What was happening was that when I showed the `ProgressWheel`, individual rows in the `ListView` would have the wheel displayed when I had never clicked them!

Well, there wasn't ever really an a-HA! moment in trying to fix this, but by the time I had understood enough of how the `ListView` actually works to debug some of the issues above, this one started to make sense.

What happens is that the views get re-cycled when you scroll down, but the bind method never *explicitly* set the state, or inflated the layout. Thus, the view that was given already had the `ProgressWheel` showing even if the item hadn't actually clicked on it. After the fix above, everything started working again.

I hope that answers any questions you may have about the `ListView`. I think the component is challenging to use, and if you don't really understand the optimizations it does, things get very strange and confusing very quickly. But now you know enough of how everything is laid out in memory to be efficient yourself! Go forward and write great code!
