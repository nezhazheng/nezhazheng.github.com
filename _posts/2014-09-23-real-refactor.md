---
layout: post
title:  "一次真实的重构过程"
categories: ["DDD"]
---

> 为了避免暴露公司业务，重构过程所涉及的领域模型与业务均为虚拟，但是每一个过程都绝对真实。

在给同事交接业务系统的过程中，发生了一个加一个小需求的插曲，给同事讲解的过程中，我一边在编码，一边在不停的对大概20行代码做连续的重构，结束后，我希望我能记录下给这段过程。希望这段过程可以给大家带来一些启发，其实代码的可读性永远都有提升空间。

而在把代码写美的过程中，其实你就是在不断的在思考业务，消化业务，加深对业务的理解，你就是在不断提炼，优化模型，使Domain Model与业务更匹配，使代码能更自然的表达业务。


####起点（代码开始时候的样子）

#####Movie Model: 大概有如下属性和方法：
{% highlight java %}
public class Movie {
	...
	//开始时间
	private Long timeBegin;
	//结束时间
	private Long timeEnd;
	//电影类型
	private MovieType movieType;

	/**
	 * 当前时间是否在电影开始结束时间区间内，并且电影内容不为空
	 * @param now
	 * @return
	 */
	{% raw %}public boolean startedAndNotEnding(Long now) { {% endraw %}
		return XXX && XXX.
	}

	public boolean isEnd(Long now) {
		return XXX.
	}
	...
}
{% endhighlight %}

#####MovieService: 用来处理与电影相关的业务，下面这个方法用来判断电影是否有效
{% highlight java %}
/**
 * 判电影是否有效，标准有四
 * 1、当前时间是否在电影开始结束时间区间内，并且电影内容不为空（自拍电影没有这条规则）
 * 2、查询区域是否命中此电影
 * 3、如果电影是情色电影，那么此电影不能出现在黑名单中
 * @return
 */
private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
	// 标准1
	return movie.startedAndNotEnding(now)
		// 标准2
		&& movie.matchArea(areaCode)
		// 标准3
		&& (movie.getMovieType() != MovieType.CATEGORY_ADWORD
				|| !movieManager.isSkuInBlacklist(movie.getBatchId().toString(), skuId));
}
{% endhighlight %}


> 需求：
> 校验电影是否有效，增加一个规则：如果电影是情色电影，则必须是XYZ公司的


####第一幕：

**同事首先针对这个需求，修改了我的注释，这个习惯是好的，开始写代码前，先思考业务。**

{% highlight java %}
/**
 * 判电影是否有效，标准有四
 * 1、当前时间是否在电影开始结束时间区间内，并且电影内容不为空（自拍电影没有这条规则）
 * 2、查询区域是否命中此电影
 * 3、如果电影是情色电影，那么此电影不能出现在黑名单中并且必须是XYZ公司的
 * @return
 */
{% endhighlight %}

**我觉得如果是写成下面这样，更有利于以后的维护。**

{% highlight java %}
/**
 * 判电影是否有效，标准有四
 * 1、当前时间是否在电影开始结束时间区间内，并且电影内容不为空（自拍电影没有这条规则）
 * 2、查询区域是否命中此电影
 * 3、如果电影是情色电影，那么此电影不能出现在黑名单中并且必须是XYZ公司的
 * 4、如果电影是情色电影，那么此电影必须是XYZ公司的
 * @return
 */
{% endhighlight %}


####第二幕：

**同事接下来准备做下面的改动，在之前的标准3中，增加新的规则。**

{% highlight java %}
  private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
        // 标准1
        return movie.startedAndNotEnding(now)
                // 标准2
                && movie.matchArea(areaCode)
                // 标准3
                && (movie.getMovieType() != MovieType.CATEGORY_ADWORD
                || !movieManager.isSkuInBlacklist(movie.getBatchId().toString(), skuId)
                && movie.getName().startWith("XYZ"));
    }
{% endhighlight %}

**我对这个改动依旧不是很满意，因为我觉得第一代码里找不着注释里写的标准四了，第二，我觉得标准3不宜读了。**

我做了下面这个改动。

{% highlight java %}
  private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
        // 标准1
        return movie.startedAndNotEnding(now)
                // 标准2
                && movie.matchArea(areaCode)
                // 标准3
                && (movie.getMovieType() != MovieType.CATEGORY_ADWORD
                || !movieManager.isSkuInBlacklist(movie.getBatchId().toString(), skuId)
                // 标准4
                && (movie.getMovieType() != MovieType.CATEGORY_ADWORD
                   || movie.getName().startWith("XYZ");
    }
{% endhighlight %}

####第三幕：

事情看起来不错，代码和注释看上去配对的井井有条。

**但是我仍然不是很满意标准3和标准4的写法，因为我觉得还是需要我思考，不够易读，第一个判断是判断不是什么类型，**
**而我的注释与业务直接的语义都是如果是什么类型，然后XXX。于是我做了下面的改动。**

{% highlight java %}
private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
	// 标准1
	return movie.startedAndNotEnding(now)
		// 标准2
		&& movie.matchArea(areaCode)
		// 标准3
		&& eroticMovieMustNotInBlackList(movie)
	        // 标准4
		&& eroticMovieMustFromXYZ(movie);
}

private boolean eroticMovieMustNotInBlackList(Movie movie) {
	if(movie.getType() != MovieType.EROTIC) {
		return true;
	}
	return !movieManager.isSkuInBlacklist(movie.getBatchId().toString());
}

private boolean eroticMovieMustFromXYZ(Movie movie) {
     if(movie.getType() != MovieType.EROTIC) {
        return true;
     }
     return movie.getId().startWith("XYZ");
}
{% endhighlight %}

**我抽取了两个小方法，并且让方法名与方法逻辑都与业务直接匹配，而且isValid这个方法变得更好读了。**

#### 第四幕：

**我试图让这两个小方法更好读更能直接对上我写的注释，于是**

{% highlight java %}
if(movie.isErotic()) {
     return movie.getId().startWith("XYZ");
}
return true;
{% endhighlight %}

#### 第五幕：

**我突然发现eroticMovieMustFromXYZ这个方法完全没必要出现在Service中，第一是因为Movie这个Model理应能“告诉我”他来自哪，第二是因为这个方法所依赖的东西全部都在Movie中。**

{% highlight java %}
public class Movie {
     private String id;
     private boolean eroticMovieMustFromXYZ() {
        if(this.isErotic()) {
          return this.fromXYZ()
        }
        return true;
     }
     private boolean fromXYZ() {
          return movie.getId().startWith("XYZ"); 
     }
}
{% endhighlight %}

**所有的调用全都变成了对"this"的调用，这种感觉很好，这意味着内聚，没有其他依赖了，并且可读性在进一步加强。
还有调用方也变成了这样。**

{% highlight java %}
 // 标准4
&& movie.eroticMovieMustFromXYZ();
{% endhighlight %}

#### 第六幕：

我们回过头来看最初的地方，它现在是这样的：

{% highlight java %}
  private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
        // 标准1
        return movie.startedAndNotEnding(now)
                // 标准2
                && movie.matchArea(areaCode)
                // 标准3
                && eroticMovieMustNotInBlackList(movie)
               // 标准4
                && movie.eroticMovieMustFromXYZ();
    }
{% endhighlight %}

**我觉得上面这个表达式有点长了，而且那四行注释有点傻气，你们觉得呢？于是：**

{% highlight java %}
  private boolean movieIsValid(Long now, Movie movie, String areaCode, String skuId) {
       boolean rule1 = movie.startedAndNotEnding(now);
       boolean rule2 = movie.matchArea(areaCode)
       boolean rule3 = eroticMovieMustNotInBlackList(movie)
       boolean rule4 = movie.eroticMovieMustFromXYZ();

       return rule1 && rule2 && rule3 && rule4.
    }
{% endhighlight %}

**做完这个改动后，我意识到我需要调整下注释，于是。**

{% highlight java %}
/**
 * 判断电影是否有效，必须同时满足下面4个条件
 * 1、当前时间是否在电影开始结束时间区间内，并且电影内容不为空（自拍电影没有这条规则）
 * 2、查询区域是否命中此电影
 * 3、如果电影是情色电影，那么此电影不能出现在黑名单中并且必须是XYZ公司的
 * 4、如果电影是情色电影，那么此电影必须是XYZ公司的
 * @return
 */
{% endhighlight %}


> 最后，重构不是一门很高深的技术，重构是对美，是对简单地一种追求！
