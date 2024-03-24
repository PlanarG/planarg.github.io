---
title: Some warm tips
date: 2024-03-14 23:42:50
categories: Configuration
---

其实主要是一些页面元素的备忘录，我太容易搞忘了 0. 0

这个主题好像是支持 [Bulma](https://bulma.io/documentation/overview) 的，也就是说可以直接向 `.md` 文件里面插入它支持的 `html` 元素。

## 分栏显示

<div class="columns">
  <div class="column">
    <article class="message is-primary">
      <div class="message-header">
        <p>这里是第一列</p>
      </div>
      <div class="message-body">
      	这里有一些内容
      </div>
    </article>
  </div>
  <div class="column">
    {% codeblock "html code" lang:html >folded %}
    <div class="columns">
      <div class="column">
        <article class="message is-primary">
          <div class="message-header">
            <p>这里是第一列</p>
          </div>
          <div class="message-body">
            这里有一些内容
          </div>
        </article>
      </div>
      <div class="column">
        哇，这里能递归吗
      </div>
    </div>
    {% endcodeblock %}
  </div>
</div>

<!-- more -->

## message

`message` 一共有 $7$ 种，它们分别是 `dark`, `primary`, `link`, `info`, `success`, `warning` 以及 `danger`。值得一提的是可以在 `class` 内加入 `message-immersive`，它会变得非常帅，比如这样

<article class="message message-immersive is-primary">
  <div class="message-body">
    此处的代码为
    ```html
    <article class="message message-immersive is-primary">
      <div class="message-body">
        此处的代码为...
      </div>
    </article>
    ```
  </div>
</article>

## 小标签页

<div class="tabs is-left is-boxed is-fullwidth">
  <ul class="mx-0 my-0">
    <li class="is-active"><a href="#first">第一个</a></li>
    <li><a href="#second">第二个</a></li>
  </ul>
</div>
<div id="first" class="tab-content">我在第一个标签页里</div>
<div id="second" class="tab-content is-hidden">我在第二个标签页里</div>

```html
<div class="tabs is-boxed is-fullwidth">
  <ul class="mx-0 my-0">
    <li class="is-active"><a href="#first">第一个</a></li>
    <li><a href="#second">第二个</a></li>
  </ul>
</div>
<div id="first" class="tab-content">我在第一个标签页里</div>
<div id="second" class="tab-content is-hidden">我在第二个标签页里</div>
```

## 代码样式

可以分别制定每个代码块是否折叠，方式是采用 `hexo` 提供的内嵌代码，它长这样

```
{% codeblock "optional file name" lang:code_language_name >folded %}
...code block content...
{% endcodeblock %}
```

## 插入图片

在文件中加入如下代码

```
{% asset_img image.jpg 200 400 This is an image %}
```

其中 `200 400` 的作用是限制图片大小。



