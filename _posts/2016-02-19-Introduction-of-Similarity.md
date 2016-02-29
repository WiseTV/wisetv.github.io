---
author: 李威
comments: true
date: 2016-02-19 9:15:10+00:00
layout: post
title: 一种相似度匹配的方法及实际应用案例
description: Introduction of Similarity
categories:
- DAMS
---

<h2>一、什么是数据挖掘应用</h2>

<p>成功的应用=算法（20%）+建模（40%）+数据质量（40%）</p>
<ul>
<li>数据质量对项目成功是有相当大的决定作用的，好的数据质量甚至不需要用什么复杂算法就能得到很好的结果。</li>
<li>建模，通俗说，就是将数据适配成算法可以处理的格式，是连接算法和数据的桥梁，是主要的业务工作，需要正确的建模方向才有可能得到成功的结果。</li>
<li>算法，涉及数学和计算机等知识。</li>
</ul>

<h2>二、	什么是相似度</h2>
<ul>
<li>针对同一事件的所有新闻比较相似。</li>
<li>两个用户的收视习惯比较相似。</li>
<li>模糊查询时，查询字符串与被查询到的内容，比较相似。</li>
<li>这里“比较相似”就指的是相似度比较高，相似指的是模糊的概念，并不是完全相等。</li>
</ul>

<h2>三、	相似度计算的简单说明</h2>
<ul>
<li>针对同一事件的所有新闻比较相似。
将新闻分词，简单来说（仅仅是简单描述，事实上新闻相似度的算法处理有很多需要值得研究的细节），两篇新闻同时包含的词较多，那么这两篇新闻相似度相对就较高。
</li>
<li>两个用户的收视习惯比较相似。
两个用户都收看了的视频较多，那么这两个用户的收视习惯相似度较高。
</li>
<li>模糊查询时，查询字符串与被查询到的内容，比较相似。
如两个字符串相同的字的个数较多，字的顺序也较为一致，那么这两个字符串的相似度较高。
</li>
</ul>

<h2>四、	相似度计算方法的规范化</h2>
<ol>
<li>确定要比较的两个实体（实体可以是新闻、收视用户、字符串等）。
</li>
<li>根据业务目标，结合业务经验，提取出实体的特征（比如新闻分词，用户收视的视频、字符串中包含的字），这些特征集合就是特征向量。
</li>
<li>对要比较的两个实体的特征向量进行计算，用算法计算出相似度。
</li>
</ol>

<h2>五、	相似度是向量间的距离</h2>
<h3>欧式距离</h3>
<ul>
<li>
<p>二维平面上两点a(x1,y1)与b(x2,y2)间的欧氏距离</p>
<img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s1.png'></img>
</li>
<li>
<p>三维空间两点a(x1,y1,z1)与b(x2,y2,z2)间的欧氏距离</p>
<img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s2.png'></img>
</li>
<li>
<p>两个n维向量a(x11,x12,…,x1n)与 b(x21,x22,…,x2n)间的欧氏距离</p>
<img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s3.png'></img>
</ul>

<h3>余弦距离</h3>
<ul>
<li>
<p>二维平面上两点a(x1,y1)与b(x2,y2)间的余弦距离,其中分母表示两个向量b和c的长度，分子表示两个向量的内积。</p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s4.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s5.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s6.png'></img></p>
</li>
<li>
<p>举一个具体的例子，假如新闻X和新闻Y对应向量分别是：
x1, x2, ..., x6400和
y1, y2, ..., y6400	
</p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s7.png'></img></p>
当两条新闻向量夹角余弦等于1时，这两条新闻完全重复（用这个办法可以删除爬虫所收集网页中的重复网页）；当夹角的余弦值接近于1时，两条新闻相似（可以用作文本分类）；夹角的余弦越小，两条新闻越不相关。</p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s8.png'></img></p>
</li>
</ul>

<h3>余弦距离和欧氏距离的对比</h3>
<ul>
<li>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps5s9.png'></img></p>
<p从上图可以看出，余弦距离使用两个向量夹角的余弦值作为衡量两个个体间差异的大小。相比欧氏距离，余弦距离更加注重两个向量在方向上的差异。</p>
</li>
</ul>

<h2>六、	实例</h2>
<h3>1、根据直播节目名称查询对应的回看节目</h3>
<ul>
<li>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s1.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s2.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s3.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s4.png'></img></p>
</li>
</ul>

<h3>2、版权剧下线操作中遇到的片名匹配问题</h3>
<ul>
<li>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s5.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s6.png'></img></p>
<p><img src ='http://tjiptv-dams.github.io/images/jssl/20160219/ps6s7.png'></img></p>
<p><<疯丫头>>第1季，字符为：<<，疯，丫，头，>>，第，1，季
疯丫头第一季，字符为：疯，丫，头，第，一，季</p>
<p>距离为：5/ sqrt(8)* sqrt(6) =2.449*2.828=0.722</p>
</li>
</ul>

<h3>3、推荐系统中相似度问题</h3>
<ul>
<li>
<p>用户1，视频1，视频2，视频3，视频4</p>
<p>用户2，视频2，视频3，视频5，视频6</p>
<p>用户3，视频2，视频3，视频5，视频6，视频7</p>
</li>
<li>
<p>用户1和用户2的相似度2/sqrt(4)* sqrt(4)=0.5</p>
<p>用户1和用户3的相似度2/sqrt(4)* sqrt(5)=2/2.472=0.447</p>
</li>
</ul>


