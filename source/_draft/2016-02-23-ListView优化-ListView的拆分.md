---
title: ListView优化：ListView的item拆分
date: 2016-02-23 20:19:18
comments: true
categories: Android
tags:
  - Android
  - ListView
---
这算是开年来的第一篇文章，距离上一篇文章过去了一个多月了。虽然这阶段挺忙的，但还是没挤出时间来写一篇文章，主要还是自己懒！好了废话少说。

# 起因  
由于一次需求的改变，导致listview的item太大，在大屏的手机中一个item超出三分之二的大小，在小屏的手机上就更不用说了。这样做下来的结果就是在滑动的时候有些卡顿，没有之前的顺滑，感觉上就很不爽。

<!--more-->

好吧，先看一下item，让大家感受一下，(原本想自己草草的画一张，可惜没有这个天赋。原谅我可耻了盗了一张设计稿过来)。

![item](http://7xrx8e.com1.z0.glb.clouddn.com/blog-img-listview_item.png)

太长有木有。

# 接着
拆分这个想法是来自前面看过的一篇文章[facebook新闻页ListView优化](http://blog.aaapei.com/article/2015/02/facebookxin-wen-ye-listviewyou-hua)，很像有木有，简直就是量身定做。

# 然后 问题来了
将那篇文章看了好多遍，还是有很多地方没弄清楚，只搞懂了想法和大概的一些思路，具体要怎么去做却还是迷迷糊糊的。主要的问题在于：如果有多个type怎么去组合显示，比如不只是妆品，还有视频什么的之类。到现在我还没想通，其实在那篇文章是有提到的，可惜我还是没想出一个所以然来，可能还是水平不够。希望有大牛来指点一下迷津，打通一下任督二脉。

# 不管了，先开搞
虽然还存在一些问题没想通，但还是没办法停下来等我想明白。就将这个问题暂时保留待什么时候想通了再来进一步的改造。

首先，开始拆分简单一点的，只有一种类型的列表，也就上面图片中显示的那种item。将其拆分成6个，还是来张图比较容易看懂。

![item_with_line](http://7xrx8e.com1.z0.glb.clouddn.com/blog-img-listview_item_with_line.png)

直接上代码，比我在这边罗哩罗嗦来得简洁。

拆分为6个item

```
public final static int ITEM_TYPE = 6;//拆分item的类型

//item的各种类型
private final int TYPE_HEADER      = 0;
private final int TYPE_VIEWPAGER   = 1;
private final int TYPE_CONTENT     = 2;
private final int TYPE_RECYCLEVIEW = 3;
private final int TYPE_COMMENT     = 4;
private final int TYPE_FOOTBAR     = 5;
```

```
@Override
   public int getViewTypeCount() {
       return ITEM_TYPE;
   }

   @Override
	public T getItem(int position) {
    position = position / ITEM_TYPE;
		if (position < 0 || position >= items.size()) {
			return null;
		}
		return items.get(position);
	}

   @Override
   public int getItemViewType(int position) {
       return position % ITEM_TYPE;
   }

   @Override
   public int getCount() {
       return items.size() * ITEM_TYPE;
   }

   @Override
   public View getView(int position, View convertView, ViewGroup parent) {
       switch (getItemViewType(position)) {
           case TYPE_HEADER://头部
                HeardViewHodler heardViewHodler;
                if (convertView == null) {
                    convertView = inflater.inflate(R.layout.item_header, parent, false);
                    heardViewHodler = new HeardViewHodler(convertView, mActivity, mImageLoader);
                    convertView.setTag(heardViewHodler);
                } else {
                    heardViewHodler = (HeardViewHodler) convertView.getTag();
                }
                Good good = getGood(position);
                if (good != null && good.user != null) {
                    heardViewHodler.bindData(good);
                }
                return convertView;
           case TYPE_VIEWPAGER://图片
                ...
           case TYPE_CONTENT://内容
                ...
           case TYPE_RECYCLEVIEW://话题
                ...
           case TYPE_COMMENT://评论
                ...
           case TYPE_FOOTBAR://底部
                ...
           default:
               return convertView;
       }
   }
```

这样就完成的item的拆分，滑动一下试试，果然顺畅了许多。

# 最后 (未完待不知道什么时候有的下篇)
关于多种类型的，使用了一种简单粗暴的方法，这个就不说，免得误人子弟。
