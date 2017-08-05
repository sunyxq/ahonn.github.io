---
title: CodeIgniter URL 路由
date: 2015-12-19 00:00:00
tags:
  - CodeIgniter
  - PHP
categories:
  - 技术文章
---

## 设置 URL 路由规则
CodeIgniter 中的 URL 一般遵循之前说的下面这种规则：
	http://example.com/index.php/controllers/method/arguments

但是有些原因会使得我们需要使用不同的规则去让 URl 映射。

当有这种需求的时候，可以在 `application/config/router.php` 的 `$route` 数组中设置需要的路由规则，在路由规则中可以使用通配符或正则表达式。

## 通配符
使用通配符的路由规则：
	$route['name/:num'] = 'foo/bar';
<!--more-->

在路由规则数组 `$route` 中，数组的 key 表示要匹配的 URL ，数组的 value 表示重定向的位置。

可以使用纯字符串去匹配，或者是使用下列两种通配符：
- (:num)	匹配数字：对应正则表达式中的 [0-9]+ ，即匹配1到多个数字
- (:any)	匹配任意字符：对于正则表达式中 [^/]+ ，即匹配1到多个除`/`外的字符

在 `$route` 数组中，前面的规则优先于后面的规则，就是说它是按照数组顺序匹配的，当匹配到规则时就不会继续匹配后面的规则了。

## 正则表达式
使用正则表达式的路由规则：
	$route['name/([a-z]+)/(\d+)'] = '$1/$2';

当访问类似 `name/foo/233` 这样的 URL 的时候会被重定向到 `foo/233`。$1、$2 表示被匹配到的第几个字符串。

## 保留路由
`$route['default_controller'] = 'welcome';` ：默认的控制器。

`$route['404_override'] = '';` ：当页面不存在时访问的控制器。

`$route['translate_uri_dashes'] = FALSE;` ：这是一个路由设置，不是规则，表示是否把 URL 中的 `-` 替换为 `_`。
