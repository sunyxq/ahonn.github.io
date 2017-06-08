---
title: CodeIgniter MVC简单示例
date: 2015-12-13 22:50:06
tags: ['CodeIgniter', 'PHP']
categories: ['技术']
---

> 通过使用 CodeIgniter 写一个 Hello World 程序来学习框架的基本使用。

默认该 Hello World 程序放在 Apache 服务器的 `www` 目录下。

## URL
在 Codelgniter 中只有单一的程序入口 index.php，之前说过它是 MVC 架构的框架，所以通常在 URL 中来确定使用的控制器及方法。

例如：
	http://example.com/index.php/controllers/method/arguments

`controllers`表示调用的控制器的类，`method`表示调用的类中的函数或方法，`argument`以及后面的表示传给控制器的参数。

<!--more-->
在开启 Apache 服务器的 `mod_rewirite` 模块下可以添加一个 `.htaccess` 文件隐藏 `index.php`，使得 URL 变成 `http://example.com/index.php/controllers/method/arguments`

.htaccess
```
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php/$1 [L]
```

## Controllers
控制器是一个类文件，存放在 `application/controllers` 下，通过 URL 关联来访问，一般 URL 中访问的控制器名与控制器的文件名，类名相同。

**控制器类文件的文件名必须使用大写字母开头，类中的类名与文件名相同**

当 `controllers` 为空时，将会调用默认的控制器，安装完成 CodeIgniter 后默认的控制器是 `welcome`。可以通过修改 `application/config/routes.php` 文件中的 `$route['default_controller']` 来设置默认的控制器。

当 URL 中`method`为空时，会默认调用`index()`，不为空时则调用控制器类中的想对应的方法。

当 URL 中存在`controllers`与`method`时，其余后面的会当做参数去接收。

在 `application/controllers` 下创建 `HelloWorld.php`:
``` php
<?php
class HelloWorld extends CI_Controller
{
	public function index()
	{
		echo 'Hello World';
	}

	public function say($message)
	{
		echo $message;
	}
}
```

通过 http://localhost/index.php/HelloWorld 来调用`HelloWorld`控制器中的`index()`。

或者通过 http://localhost/index.php/HelloWorld/say/HelloWorld 来调用`say`方法打印后面的参数`HelloWorld`。

## View
视图简单来说就是一个 Web 页面，或者是页面的一部分。视图在 MVC 模式中不是直接被调用的，是通过控制器加载，来显示特定的视图。

在控制器中使用 `$this->load->view('name')` 加载制定的视图。如果需要添加动态内容，可以通过视图加载方法的第二个参数向视图传入数据。**传入的参数是一个数组或者对象。**

在 `application/views` 下创建 `HelloWorld.php`:
``` html
<html>
	<head>
	<title> Hello World </title>
	</head>
	<body>
		<p><?= $message; ?></p>
	</body>
</html>
```

再将 `application/controllers` 下的 `HelloWorld.php` 修改为：
``` php
<?php
class HelloWorld extends CI_Controller
{
	public function index()
	{
		$data['message'] = 'Hello World';
		$this->load->view('HelloWorld', $data);
	}
}
```

通过访问 http://localhost/index.php/HelloWorld 来调用`HelloWorld`控制器中的 `index()` 加载名为 `HelloWorld` 的视图，并为视图添加默认的动态数据 `Hello World`。

## Medel
模型在 MVC 模式中是专门处理数据的，对数据库的操作等都在模型类中实现。当多个控制器需要使用同一个方式处理或者获取数据时，就实现了代码复用。

模型一般在控制器中加载并调用，使用 `$this->load->model('model_name')` 来加载，与控制器不同的是，模型的类文件名与类名都要有 `_model` 后缀。然后在控制器中可以通过 `$this->model_name->method()` 来调用模型中的方法。

如果有一个模型在整个程序中都需要使用，那么可以通过在 `application/config/autoload.php` 文件中将模型添加到 `$autoload['model']` 中。

在 `application/models` 中添加 `HelloWorld_model.php`:
``` php
class HelloWorld_model extends CI_Model
{
	public function get_message()
	{
		return 'Hello World';
	}
}
```

然后将`application/controllers` 下的 `HelloWorld.php` 修改为：
``` php
<?php
class HelloWorld extends CI_Controller
{
	public function index()
	{
		$this->load->model('HelloWorld');
		$data['message'] = $this->HelloWorld->get_message();
		$this->load->view('HelloWorld', $data);
	}
}
```

这样一个简单的使用了 MVC 模式的 Hello World 程序就完成了。

可以通过访问 http://localhost/index.php/HelloWorld 来查看，虽然使用了 M 跟 V 之后所显示的页面与只使用 C 的显示是一样的，这是由于在这个简单的 Hello World 程序中没有实际到比较复杂的过程，MVC 在实际使用中可以很好的解耦，使得代码大量复用，也易于程序分层。
