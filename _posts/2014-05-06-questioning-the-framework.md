---
layout: post
title: "Questioning the Framework"
modified: 2014-05-07 22:52:43 -0400
tags: [framework, dependency injection, di, ioc, inversion of control, jsr330, android annotations, butterknife, dagger]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---
*(insert joke about thinking outside the box here)*
------------------------------------------------
 
So, I've been trying to refactor some of the code design recently. Most of this is centered around the [DownloadManager](https://github.com/MinimalBible/MinimalBible/blob/adc3bbf69cb9326b561d8b22a20fd58162970683/MinimalBible/src/org/bspeice/minimalbible/activities/downloader/DownloadManager.java) class, but the [BookListFragment](https://github.com/MinimalBible/MinimalBible/blob/adc3bbf69cb9326b561d8b22a20fd58162970683/MinimalBible/src/org/bspeice/minimalbible/activities/downloader/BookListFragment.java) is involved as well. Before we get too much farther though, let's explain the use case.
 
The `BookListFragment` is responsible for displaying a `ListView` of books available for download, whether Bibles, Dictionaries, Commentaries, etc. There is one fragment per category of books. However, trying to refresh the list of available books is a very expensive operation. So, there is a `DownloadManager` to make downloading easy, and provide a caching mechanism. This class should be a Singleton to ensure that we re-use the list of books.
 
And that's the basic use case. Time for some code.
 
**BookListAdapter.java**
{% highlight java %}
@Override
public View getView(int position, View convertView, ViewGroup parent) {
    BookItemView itemView;
    if (convertView == null) {
        itemView = BookItemView_.build(this.ctx);
    } else {
        itemView = (BookItemView) convertView;
    }
    itemView.bind(getItem(position));
    return itemView;
}
{% endhighlight %}
 
**BookItemView.java**
{% highlight java %}
@EViewGroup(R.layout.list_download_items)
public class BookItemView extends RelativeLayout {
    @ViewById (R.id.img_download_icon) ImageView downloadIcon;
    @ViewById(R.id.txt_download_item_name) TextView itemName;
    @ViewById(R.id.img_download_index_downloaded) ImageView isIndexedDownloaded;
    @ViewById(R.id.img_download_item_downloaded) ImageView isDownloaded;
    public BookItemView (Context ctx) { super(ctx); }
    public void bind(Book b) {
        itemName.setText(b.getName());
    }
}
{% endhighlight %}
 
Here's what's going on: The `BookItemView` is responsible for displaying an individual element in the list. [AndroidAnnotations](http://androidannotations.org/) is responsible for injecting the `LayoutInflater`, and individual views (the `@ViewById`). From there, the `BookListAdapter` can then build individual elements of the list using `BookItemView_.build()`.
 
This works mostly fine, but here's my issue - the `BookItemView` is small enough, I feel it should be an inner class. I can't do that with AndroidAnnotations. Also, the astute reader will notice that I'm building a `BookView_`, not a `BookView`. That's AndroidAnnotations generating the class for me. Which is exactly what it should be doing, but in my limited experience I've been having to learn the semantics of when/not to use the generated class. Plus when you have a lot of generated classes all over the place, I think it will get complicated.
 
Onward to the `DownloadManager`. I haven't committed code for this yet, but I'm hoping to illustrate my point. The class needs to be a singleton, and will eventually need an EventBus inject. I imagine it would look something like this:
 
**DownloadManager.java**
{% highlight java %}
@Singleton
public class DownloadManager {
    @Inject EventBus downloadBus;
    // Extra methods to actually do stuff here
}
{% endhighlight %} 

Now, the first question is, where do I get `@Inject` from? I'm wanting to use [Dagger](http://square.github.io/dagger/), as it doesn't force me to use `Class_` naming, and is [JSR-330](https://jcp.org/en/jsr/detail?id=330) compliant (which mostly means I think the `@Bean` naming used by AndroidAnnotations looks weird).
 
The second question is: where does `@Singleton` come from? That's a little bit ambiguous. Both Dagger and AndroidAnnotations provide an annotation, but one forces me to use the `Class_` naming *(AndroidAnnotations)*, one doesn't. One forces me to build an `ObjectGraph` at runtime *(Dagger)*, one doesn't.  This can get very confusing.
 
Realizing I was about to run into this confusion, I ported the existing AndroidAnnotations code to use [ButterKnife](http://jakewharton.github.io/butterknife/). ButterKnife is a library used to support view injection (but does a bit more). It's not as feature-filled as AndroidAnnotations, but doesn't require that I use `Class_`.
 
You can check out the code over [here](https://github.com/MinimalBible/MinimalBible/tree/b49facb2fee9b0cbced5ee9d0c75dbf9afdb0cc2). The interesting bits that use ButterKnife are [here](https://github.com/MinimalBible/MinimalBible/blob/b49facb2fee9b0cbced5ee9d0c75dbf9afdb0cc2/MinimalBible/src/org/bspeice/minimalbible/activities/downloader/BookListAdapter.java). ButterKnife requires me to write a bit more boilerplate code, but doesn't force the class naming rules on me. It's a bit of a tradeoff.
 
So that's a quick overview of the problem. All that to say, **I'm questioning what I should use as a framework**. I like the functionality AA provides, but it comes with a readability/understandability penalty. Coming from Python, I'm a big fan of [making things explicit](http://legacy.python.org/dev/peps/pep-0020/), but AA does provide a lot. ButterKnife/Dagger are both incredibly well-designed, and very focused. Which means they leave me to do some of the legwork myself.
 
I've also been investigating [RoboGuice](https://github.com/roboguice/roboguice) as well, and I'm going to write another blog post comparing everything soon!
