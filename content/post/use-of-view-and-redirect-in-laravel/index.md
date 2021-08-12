---
title: Laravel 中视图 view() 与重定向 redirect() 的使用
slug: use-of-view-and-redirect-in-laravel
description: 本文介绍了 Laravel 中视图 view() 与重定向 redirect() 的使用
date: 2018-09-04T12:46:00+08:00
tags:
  - PHP
  - Laravel
categories:
  - 某不存在的技术
---

## 一、 view() 的使用

1.  简单的返回视图

    ```php
    // 所传的参数是blade模板的路径
    // 如果目录是 resources/views/static_pages/home.blade.php 则可以使用
    return view('static_pages/home');
    或
    return view('static_pages.home');
    ```

2.  向视图传递数据

    ```php
    $title = 'Hello Laravel';
    $user = User::find(1);

    // view() 的第二个参数接受一个数组
    return view('static_pages/home', compact('user'));

    return view('articles.lists')->with('title',$title);

    // 所传递的变量在blade模板中用 {{ $title }} 或 {!! $title !!} 输出
    // 前者作为文本输出，后者作为页面元素渲染
    ```

## 二、redirect() 的使用

1.  基于 Url 的重定向

    ```php
    // 假设我们当前的域名为：http://localhost  则重定向到 http://localhost/home
    return redirect('home');
    ```

2.  基于路由的重定向

    ```php
    return redirect()->route('home');
    ```

3.  基于控制器的重定向

    ```php
    return redirect()->action('UserController@index')
    ```

4.  传递数据

    ```php
    return redirect('home')->with('title', 'Hello Laravel');

    // 将表单值保存到 Session 中，可以用 {{ old('param') }} 来获取
    return redirect('home')->withInput();

    // 接收一个字符串或数组，传递的变量名为 $errors
    return redirect('home')->withErrors('Error');
    ```

5.  其他用法

    ```php
    // 返回登录前的页面，参数为默认跳转的页面
    redirect()->intended(route('home'));

    // 返回上一个页面，注意避免死循环
    redirect()->back();
    ```

## 三、使用 view() 或 redirect() 的选择

### view() 和 redirect() 的异同

​ 使用 `return view()` 不会改变当前访问的 url ， `return redirect()` 会改变改变当前访问的 url

​ 使用 `return view()` 不会使当前 Session 的 Flash 失效 ，但是 `return redirect()` 会使 Flash 失效

​ 在 RESTful 架构中，访问 `Get` 方法时推荐使用 `return view()` ，访问其他方法推荐使用 `return redirect()`
