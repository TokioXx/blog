---
title: 获取平衡：响应式文字布局
categories:
    - frontend
    - translate
tag:
    - css
    - best practice
---

[原文链接](https://24ways.org/2016/responsive-display-text/?utm_source=CSS-Weekly&utm_campaign=Issue-243&utm_medium=email "Get the Balance Right: Responsive Display Text" )

当我们设置文字时，最好根据预定义的屏幕尺寸去设置大小。如果使用'rem'，字体的大小将根据页面默认大小和用户设置来缩放。你可以在相同的缩放条件下选择更大的字体来到达同样的目的。

无论如何，显示在屏幕的相应大小的文字，是先被看到，才能读出来。换句话，用图片的角度来看。你很可能会根据视口的大小去缩放情境图片、封面等。用相同的方式去显示图片，对相应的屏幕或者窗口固定文字的形状和大小。

#### 介绍视口单位

CSS3带来了一组相对于视口固定的单位。无论你在任何地方想要使用长度单位比如px, em或者百分比时，都可以使用视口单位来代替。总共有4种视口单位，值为1表示视口高或宽的1%。

- vw - 视口宽度
- vh - 视口高度
- vmin - 视口宽高的较小值
- vmax - 视口宽高的较大值

你可以将标题大小按屏幕比例缩放，而不是用一系列的媒体查询。下面的语句将标题字体大小设置为屏幕宽度的13%。

```css
h1 {
    font-size: 13 vw;
}
```

对应下列的宽度，渲染的字体大小为:

<table><caption>Rendered font size (px)</caption>
<thead><tr><th>Viewport width</th>
<th>13 vw</th>
</tr></thead><tbody style="text-align:right"><tr><td>320</td>
<td>42</td>
</tr><tr><td>768</td>
<td>100</td>
</tr><tr><td>1024</td>
<td>133</td>
</tr><tr><td>1280</td>
<td>166</td>
</tr><tr><td>1920</td>
<td>250</td>
</tr></tbody></table>
使用这种方式时，文字块级元素会在横竖屏下出现比例不协调的问题。由于字体大小是基于视口的宽度来设置，所以当设备横屏时会比竖屏是大出很多。
<figure><img alt="Portrait device next to landscape device with the same sized text" src="https://media.24ways.org/2016/rutter/viewport-units-vmin.svg"><figcaption>Landscape text is consistent with portrait text when using vmin units.</figcaption></figure>
比较vm和vmin在众多常用屏幕尺寸下的渲染大小，你可以看出使用vmin可以将字体设置为一个合适的大小。

<table><caption>Rendered font size (px)</caption>
<thead style="text-align:right"><tr><th>Viewport</th>
<th>13 vw</th>
<th>13 vmin</th>
</tr></thead><tbody style="text-align:right"><tr><td>320 × 480</td>
<td>42</td>
<td>42</td>
</tr><tr><td>414 × 736</td>
<td>54</td>
<td>54</td>
</tr><tr><td>768 × 1024</td>
<td>100</td>
<td>100</td>
</tr><tr><td>1024 × 768</td>
<td>133</td>
<td>100</td>
</tr><tr><td>1280 × 720</td>
<td>166</td>
<td>94</td>
</tr><tr><td>1366 × 768</td>
<td>178</td>
<td>100</td>
</tr><tr><td>1440 × 900</td>
<td>187</td>
<td>117</td>
</tr><tr><td>1680 × 1050</td>
<td>218</td>
<td>137</td>
</tr><tr><td>1920 × 1080</td>
<td>250</td>
<td>140</td>
</tr><tr><td>2560 × 1440</td>
<td>333</td>
<td>187</td>
</tr></tbody></table>

#### 混合字体设置

通过媒体查询根据屏幕尺寸直接设置展示型文字的大小时很好用。但是在面对标题性文字的时候就不是很推荐了，比如副标题，你可能希望它变化的比例不像展示型的文字那么大。举个例子，你可以使用视口单位去设置副标题的大小，让它和主标题同比例的变化。

```css
h1 {
    font-size: 13vmin;
}
h2 {
    font-size: 5vmin;
}
```

<figure class="fullwidth"><img alt="Phone next to tablet showing how text is scaled at the same rate" src="https://media.24ways.org/2016/rutter/viewport-hybrid-vmin.svg"><figcaption>Using vmin alone for supporting text can scale it too quickly</figcaption></figure>

展示型文字和标题型文字的平衡在手机屏幕上显示的还算不错，但是平板上的副标题，尽管随着主标题同时增大，还是会感觉到不协调的大和一点笨拙。这个问题会在更大的屏幕上变得更加严重。

一种解决方案是使用混合字体设置。我们可以使用CSS3中的calc()方法同时基于rem和视口单位去计算。举个例子：

```css
h2 {
    font-size: calc(0.5rem + 2.5vmin);
}
```

对于320px宽度的屏幕，字体大小为16px，计算如下：

```css
(0.5 × 16) + (320 × 0.025) = 8 + 8 = 16px	
```

对于768px宽度的屏幕，字体大小为27px，计算如下：

```css
(0.5 × 16) + (768 × 0.025) = 8 + 19 = 27px
```

结果得到了一个更平衡的副标题，不会喧宾夺主。

<figure class="fullwidth"><img alt="Phone next to tablet showing smaller subheading scaled using a hybrid method" src="https://media.24ways.org/2016/rutter/viewport-hybrid-calc.svg"></figure>

为了给你展示使用混合方式的效果，这里是一个混合方式和视口单位一一对应的表格。
<table class="ex--scale"><caption>
<i>top:</i> calc() hybrid method; <i>bottom</i>: vmin only</caption>
<tbody><tr class="ex--scale-size"><td style="font-size:0.875rem">16</td>
<td style="font-size:1.09375rem">20</td>
<td style="font-size:1.4765625rem">27</td>
<td style="font-size:1.75rem">32</td>
<td style="font-size:1.9140625rem">35</td>
<td style="font-size:2.1875rem">40</td>
<td style="font-size:2.40625rem; padding-top: 1rem;">44</td>
</tr><tr class="ex--scale-size"><td style="font-size:0.875rem">16</td>
<td style="font-size:1.3125rem">24</td>
<td style="font-size:2.078125rem">38</td>
<td style="font-size:2.625rem">48</td>
<td style="font-size:2.953125rem">54</td>
<td style="font-size:3.5rem">64</td>
<td style="font-size:3.9375rem; padding-top: 1rem;">72</td>
</tr><tr class="ex--scale-key"><td>320</td>
<td>480</td>
<td>768</td>
<td>960</td>
<td>1080</td>
<td>1280</td>
<td style="padding-top: 1rem;">1440</td>
</tr></tbody></table>

在这个节日，尝试一下使用rem和vmin的混合方式，感受下最适合你的设置。
