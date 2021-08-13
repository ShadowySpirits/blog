---
title: php 几种非阻塞方式分析
slug: analysis-of-several-nonblocking-modes-of-php
date: 2019-03-02T12:11:00+08:00
tags:
  - PHP
categories:
  - 某不存在的技术
---

本文介绍了几种非阻塞执行 php 代码的方法，并分析他们的性能与适用环境

<!--more-->

本文用如下代码来测试这几种非阻塞实现的性能：

```php
// 1.php:
$stime = microtime(true);
$stime .= PHP_EOL;
// 执行非阻塞逻辑
file_put_contents(__DIR__ . '/debug.log', 'stime：' . $stime, FILE_APPEND);

// 2.php:
// 模拟耗时操作
sleep(2);
$etime = microtime(true);
$etime .= PHP_EOL;
file_put_contents(__DIR__ . '/debug.log', 'etime：' . $etime, FILE_APPEND);
```

## curl

利用 php curl 函数设置一个较小的 timeout，来间接达成非阻塞。这种方法会主动断开连接，所以对于有这方面限制的 api 是不适用的

### CURLOPT_TIMEOUT

较老的 curl 拓展只支持 `CURLOPT_TIMEOUT` 这个参数，也就是最低 timeout 为 1s

```php
function curl() {
  $ch = curl_init();
  curl_setopt($ch, CURLOPT_URL, '127.0.0.1/aaa.php');
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
  curl_setopt($ch, CURLOPT_TIMEOUT, 1); // 一秒后关闭连接
  curl_setopt($ch, CURLOPT_HEADER, true);
  curl_exec($ch);
  curl_close($ch);
}

output:
3.002
3.001
2.9951
```

### CURLOPT_TIMEOUT_MS

这个参数在 curl 7.16.2 中被加入，用于设置毫秒级的 timeout

{{< tip warning >}}
如果 libcurl 编译时使用系统标准的名称解析器（ standard system name resolver），那部分的连接仍旧使用以秒计的超时解决方案，最小超时时间还是一秒钟
{{< /tip >}}

```php
function curl() {
  $ch = curl_init();
  curl_setopt($ch, CURLOPT_URL, '127.0.0.1/aaa.php');
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
  curl_setopt($ch, CURLOPT_TIMEOUT_MS, 10); // 10 毫秒后关闭连接
  curl_setopt($ch, CURLOPT_HEADER, true);
  curl_exec($ch);
  curl_close($ch);
}

output:
2.0141
2.0043
2.0127
```

## curl_multi

利用 cURL 中的 `curl_multi_*` 函数发送异步请求，并且可以并发请求

```php
function curl_multi() {
  $mh = curl_multi_init();
  $ch = curl_init();
  curl_setopt($ch, CURLOPT_URL, '127.0.0.1/aaa.php');
  curl_multi_add_handle($mh, $ch);
  curl_multi_exec($mh, $active);
  curl_close($ch);
  curl_multi_remove_handle($mh, $ch);
  curl_multi_close($mh);
}

output:
2.0134
2.0226
2.0157
```

## fsocketopen 或 stream_socket_client

用 `fsocketopen()` 打开一个连接，然后用 `stream_set_blocking()` 设置非阻塞模式

```php
function sock() {
  $fp = fsockopen('127.0.0.1', 80, $error_code, $error_msg, 1);
  if (!$fp) {
  return array('error_code' => $error_code, 'error_msg' => $error_msg);
 }
  stream_set_blocking($fp, 0);
  $header = "GET /aaa.php HTTP/1.1\r\n";
  $header .= "Host: 127.0.0.1\r\n";
  $header .= "Connection: close\r\n\r\n";
  fwrite($fp, $header);
  fclose($fp);
  return array('error_code' => 0);
}

output:
2.0348
2.0142
2.0149
```

## fastcgi_finish_request

这个函数可以返回缓冲区内的所有响应的数据给客户端并结束请求，但是仍继续运行当前脚本直到运行完成或者达到 timeout。所以我们可以在耗时操作前使用该函数以实现非阻塞

注意：

1.  虽然这个函数叫 fastcgi 但是这是个实实在在的 FPM 函数，需要在 FPM 环境下调用
2.  使用该函数后，这个脚本将占用一个 FPM 进程，所以如果对高耗时脚本滥用此函数会导致 502 bad gateway
3.  使用这个函数会给当前 session 加锁，如不需要修改 session 推荐使用 `session_write_close()` 解除占用

```php
$stime = microtime(true);
if (PHP_SAPI === 'fpm-fcgi' && function_exists('fastcgi_finish_request')) {
  fastcgi_finish_request();
}
sleep(2);
$etime = microtime(true);
$etime .= PHP_EOL;
file_put_contents(__DIR__ . '/debugtest.log', $stime - $etime, FILE_APPEND);

output:
2.00048
2.00043
2.00042
```

## pcntl_fork

用 `pcntl_fork` 创建子进程的方式实现真正的异步，这个函数由 pcntl 扩展提供。
为了防止子进程变成僵尸进程，在父进程使用 `pcntl_wait` 等待子进程返回并回收资源
我这里采用了二次 fork 的方式，让第一次 fork 出的子进程 a 再 fork 出实际的工作进程 b，让 a 先行退出，使得 b 成为孤儿进程，被 init 进程托管。这样实现了父进程非阻塞，而且子进程不会成为僵尸进程。

```php
class Arrow
{

    static $instance;

    public static function getInstance()
    {
        if (null === self::$instance)
            self::$instance = new self();
        return self::$instance;
    }

    public function run($rb)
    {
        cli_set_process_title('process_a');
        $pid = pcntl_fork();
        if ($pid > 0) {
            pcntl_wait($status);
        } elseif ($pid == 0) {
            $cid = pcntl_fork();
            if ($cid > 0) {
                cli_set_process_title('process_b');
                exit();
            } elseif ($cid == 0) {
                cli_set_process_title('process_c');
                $rb();
            } else {
                exit();
            }
        } else {
            exit();
        }
    }

$stime = microtime(true);
$stime .= PHP_EOL;
Arrow::getInstance()->run(function () use ($stime) {
    sleep(2);
    $etime = microtime(true);
    $etime .= PHP_EOL;
    file_put_contents(__DIR__ . '/debug.log', $stime - $etime, FILE_APPEND);
});

output:
2.023
2.004
2.010

```

我们用 `ps ax` 查看一下进程：
TTY 为 ? 的进程即为与终端无关，是守护进程（daemon）

```bash
ps ax | grep -v grep | grep -E 'process_|PID'
  PID TTY      STAT   TIME COMMAND
 7750 ?        S      0:00 process_c
```
