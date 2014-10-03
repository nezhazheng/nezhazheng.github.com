---
layout: post
title:  "Guava Style"
---

> [Guava]是这么一套能让你写出**人类更容易接受**的Java代码的程序库。

###Optional

####Traditional Java Style 

{% highlight java %}
// Scene 1
void func(String value) {
	if(value == null)
		// do something…
	else 
		<span >// ...</span>
}

// Scene 2
String value = a;
if(value == null) {
	value = b;
}
{% endhighlight %}

####Guava Style

{% highlight java %}
import static guava.Optional;

// Scene 1
void func(Optional<String> value) {
	if(value.isPresent())
		// do something…
	else 
		// ...
}

// Scene 2
value = fromNullable(a).or(b);
{% endhighlight %}

###Objects

####Traditional Java Style

{% highlight java %}
if(first == null){
	return second;
}
return first;
{% endhighlight %}

####Guava Style

{% highlight java %}
Objects.firstNonNull(first, second) // return second if first is null .
{% endhighlight %}

###Strings

####Strings

{% highlight java %}
emptyToNull(String)
nullToEmpty(String)
isNullOrEmpty(String)
{% endhighlight %}

####Joiner

{% highlight java %}
Joiner joiner = Joiner.on("; ").skipNulls();
return joiner.join("Harry", null, "Ron", "Hermione");
{% endhighlight %}

####Splitter

{% highlight java %}
Splitter.on(',')
       .trimResults()
       .omitEmptyStrings()
       .split("foo,bar,,   qux");
{% endhighlight %}


###Collections

####Fluent API

使用Guava的最大原因就是因为它提供了一堆的让人用起来很爽的流式API，说直白点就是把你的代码读得像文章一样通顺，这种流式API也是一种DSL的表现形式。

有了Guava之后下面的这种shit可以滚粗Java世界了。

ps:强烈建议在用流式API的时候通过import static SomeClass.* 的方式引入DSLBuilder，这种类似FluentIterable.from的写法读起来很别扭。

{% highlight java %}
for(int i = 0; i < array.length; i++){
	if(array[0] blablabla..){
		continue;
	}
	do something..
	new Wrap(array);
}
{% endhighlight %}

####Guava Style

{% highlight java %}
from(Iterable).filter(Predicate).transform(Function).toList();

// Predicate接口就像是一个条件，返回true or false.
// and not or 是Predicate中得API
Predicate<TypeA> isOK(listElement){
	return and(not(or(isClassField(), isFinalField())), fieldExist(target));
}

Function<TypeB> wrapToWrapper(listElement){}
{% endhighlight %}

###[New collection types](https://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained)

####Multiset MultiMap

Multiset可以put重复的value,Multimap可以put重复的key.

{% highlight java %}
HashMultiset<String> set = HashMultiset.create();
set.add("a", 3);
set.size(); // got 3
//can do 
set.add("b");
set.add("b");

//forget Map<String,List<Integer>> map = new HashMap()<String,List<Integer>> ;
//let's do this
HashMultimap<String,Integer> map = HashMultimap.<String,Integer>create();
map.get("a") // will got a Set,and the set's size is 0 
map.put("a", 1);
map.put("a", 2);
{% endhighlight %}

#####BiMap

The keyword is inverse,you can inverse your map by map.inverse(),then value will be the key.

{% highlight java %}
BiMap<Integer, String> biMap = ImmutableBiMap.<Integer,String>builder().put(2, "2").put(3,"3").build();
assertThat(biMap.get(2),is("2"));
assertThat(biMap.inverse().get("3"),is(3));
{% endhighlight %}

#####[Utility Classes](https://code.google.com/p/guava-libraries/wiki/CollectionUtilitiesExplained)

{% highlight java %}
private List<String> includes = of();
private List<String> excludes = of();
private Iterable<MountPredicate> toMountPredicates() {
    return unmodifiableIterable(concat(
                transform(includes, toInclude()),
                transform(excludes, toExclude())));
    }
{% endhighlight %}

除此之外，Guava还有其他一些工具类，例如对事件进行处理的EventBus，反射的流式API等等。鉴于此篇博文并非是API翻译文档，读者可以在[guava]的官网上里看到详细文档。

[guava]: (https://code.google.com/p/guava-libraries/)




