---
layout: post
title: "配置Octopress支持LaTex数学公式"
date: 2015-08-08 00:01:02 +0800
comments: true
categories: [备忘]
tag: Octopress
---

Octopress 默认不支持 LaTex 写数学公式需要更改配置才可以。

## 设置
需要使用kramdown来支持LaTex写数学公式

<!--more-->

### 用kramdown替换rdiscount
1. 安装kramdown

	``` bash
	$ sudo gem install kramdown
	```
2. 修改`_config.yml`配置文件，将所有`rdiscount`替换成`kramdown`
3. 修改`Gemfile`，把`gem 'rdiscount'`换成`gem 'kramdown'`
	
### 添加MathJax配置
在`/source/_includes/custom/head.html`文件中，添加如下代码：

``` javascript
<!-- mathjax config similar to math.stackexchange -->
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  jax: ["input/TeX", "output/HTML-CSS"],
  tex2jax: {
    inlineMath: [ ['$', '$'] ],
    displayMath: [ ['$$', '$$']],
    processEscapes: true,
    skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
  },
  messageStyle: "none",
  "HTML-CSS": { preferredFont: "TeX", availableFonts: ["STIX","TeX"] }
});
</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
```

### 修复 MathJax 右击页面空白 bug
修改`~/sass/base/theme.scss`文件，如下代码变为：

``` css
> div#main {
     background: $sidebar-bg $noise-bg;
     border-bottom: 1px solid $page-border-bottom;
     > div {
```

## 随之出现的问题，以及解决方法
将`rdiscount`替换成`kramdown`之后，以前写的博客里面，很多内容都不能正确显示了，并且在`rake generate`时候，会报错，内容大约是：`Error:  Pygments can't parse unknown language: </p>`

原生的语法高亮插件`Pygments`很强大，支持语言也很多，但是这时候报的错误让人一头雾水。

### 找出原因
报错部分代码在`/plugins/pygments_code.rb`文件中，

``` ruby
def self.pygments(code, lang)
    if defined?(PYGMENTS_CACHE_DIR)
      path = File.join(PYGMENTS_CACHE_DIR, "#{lang}-#{Digest::MD5.hexdigest(code)}.html")
      if File.exist?(path)
        highlighted_code = File.read(path)
      else
        begin
          highlighted_code = Pygments.highlight(code, :lexer => lang, :formatter => 'html', :options => {:encoding => 'utf-8', :startinline => true})
        rescue MentosError
          raise "Pygments can't parse unknown language: #{lang}."
        end
        File.open(path, 'w') {|f| f.print(highlighted_code) }
      end
```

修改一下代码，将出问题的代码高亮部分抛出来，

```
raise "Pygments can't parse unknown language: #{lang}#{code}."
```

Google了一下原因， 原来是因为最新版的`pygments`这个插件对于Markdown的书写要求更严格了：

> 1. Some of my older blog posts did not contain a space between the triple-backtick characters and the name of the language being highlighted. Earlier versions of pygments did not care, but the current version is a stickler.
2. pygments appears to want a blank line between any triple-backtick line and any other text in the blog post.

好吧，以后写文章要更细心一点了。:)

参考：

- [Octopress中使用Latex写数学公式](http://dreamrunner.org/blog/2014/03/09/octopresszhong-shi-yong-latexxie-shu-xue-gong-shi/)
- [Pygments Unknown Language](http://www.leexh.com/blog/2014/09/21/pygments-unknown-language/)
