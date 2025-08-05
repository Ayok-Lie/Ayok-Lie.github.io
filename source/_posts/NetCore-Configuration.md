---
abbrlink: ''
categories:
- - C#
date: '2025-08-04T10:57:49.602425+08:00'
excerpt: 介绍 .NET Core 配置系统，涵盖 XML、JSON 等多种配置源。通过 IConfiguration 接口统一管理，结合扩展包实现灵活配置。
sticky: ''
tags:
- .Net基础原理
title: .Net Core基础—Configuration
updated: '2025-08-05T11:06:38.917+08:00'
---
## 基础知识

配置的本质是键值对，微软对于配置提供了大量的配置源提供程序，包括xml，json，ini，环境变量，命令行参数，内存等等。还提供了一个扩展包用于配置绑定和类型转换。

- 依赖项

  Microsoft.Extensions.Configuration.Abstractions：抽象包，用于编写扩展时使用

  Microsoft.Extensions.Configuration：基础包提供了内存配置的方案

  Microsoft.Extensions.Configuration.Json：json支持

  Microsoft.Extensions.Configuration.EnvironmentVariables：环境变量支持

  Microsoft.Extensions.Configuration.CommandLine：命令行参数

  Microsoft.Extensions.Configuration.Binder：用于配置绑定到实体或者基本类型
- 核心接口

  IConfiguration：配置的核心接口，用于读取配置

  IConfigurationRoot：表示根配置，继承IConfiguration接口

  IConfigurationSection：表示子配置节点，继承IConfiguration接口

  IConfigurationBuilder：提供了大量的扩展，用于构建IConfiguration实列。

  ConfigurationManager：实现了IConfigurationRoot、IConfigurationBuilder接口。因此ConfigurationManager既可以用于构建配置，也可以读取配置。

## 注册配置

```json
{
  "MvcOptions": {
    "Host": "127.0.0.1",
    "Port": 8080,
    "Urls": [
      "http://127.0.0.1:80",
      "http://127.0.0.1:81",
      "http://127.0.0.1:82"
    ]
  },
  "ConnectionStrings": {
      "default": "127.0.0.1"
  }  
}
```

```C#
public class MvcOptions
{
    public string Host { get; set; }
    public int Port { get; set; }
}
```

```C#
//需要安装相应的依赖包
static void Main(string[] args)
{
    var configurationBuilder = new ConfigurationManager();
    //内存配置源
    configurationBuilder.AddInMemoryCollection(new Dictionary<string, string>()
    {
        { "m1","45"},
        { "m2","45"}
    });
    //json配置源
    configurationBuilder.SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("appsettings.json");
    //命令行配置源
    configurationBuilder.AddCommandLine(args);
    //环境变量配置源：可以指定前缀
    configurationBuilder.AddEnvironmentVariables("ASPNETCORE_");
}
```

## 构建配置

```C#
//由于ConfigurationManager实现了IConfiguration接口，并且没有build方法
//因此构建很简单，或者你根本不需要构建，但是建议把配置和读取分开
IConfiguration configuration = configurationBuilder;
```

## 读取配置

- 基础接口

```C#
//通过索引读取配置
string m1 = configuration["m1"];
//获取子配置
IConfigurationSection options = configuration.GetSection("MvcOptions");
var host = options["Host"];
//获取数组
IEnumerable<IConfigurationSection> sections = configuration.GetSection("MvcOptions:Urls").GetChildren();
foreach (IConfigurationSection item in sections)
{
    var url = item.Value;
}
//内置的一个获取链接字符串的api
var defaultConnectionString = configuration.GetConnectionString("default");
```

- 扩展接口，需要安装Binder支持

```C#
//将配置绑定到基本类型上，底层调用Convert.ToXXX();
var host = configuration.GetValue<string>("MvcOptions:Host");
var port = configuration.GetValue<int>("MvcOptions:Port");
//将配置绑定到实列上，底层调用GetValue
var options1 = new MvcOptions();
configuration.Bind("MvcOptions", options);
//将配置绑定到实列上，并返回这个实列，底层调用Bind
var options2 = configuration.GetSection("MvcOptions").Get<MvcOptions>();
```

## ChangeToken

- 依赖项

  Microsoft.Extensions.Primitives：提供配置更改的核心接口和实现
- 核心接口

  IChangeToken：用于注册回调

  ChangeToken：用于绑定生产者和消费者，注册一个回调，如果更改发生就获取使用生成者生产一个新的IChangeToken，并执行消费者，并在次注册回调

  CancellationChangeToken：IChangeToken的实现一种实现，

  CancellationTokenSource：用于产生取消令牌，执行取消令牌，

  CancellationToken：取消令牌，可以注册取消之后的回调。负责在CancellationChangeToken和CancellationTokenSource直接传递消息

  ```C#
  //当更改发送时
  private void OnChangeTokenFired()
  {
      //获取新的IChangeToken
      IChangeToken token = _changeTokenProducer();

      try
      {
          //执行消费者委托
          _changeTokenConsumer(_state);
      }
      finally
      {
          //使用新的token继续注册回调
          RegisterChangeTokenCallback(token);
      }
  }

  private void RegisterChangeTokenCallback(IChangeToken token)
  {
      if (token is null)
      {
          return;
      }
  	//通过IChangeToken注册回调
      IDisposable registraton = token.RegisterChangeCallback(s => ((ChangeTokenRegistration<TState>)s).OnChangeTokenFired(), this);

      SetDisposable(registraton);
  }
  ```
- 使用案例

  ```c#
  internal class Program
  {
      static void Main(string[] args)
      {
          var provider = new FileConfigurationProvider();
          //绑定
          provider.Watch();
          new TaskCompletionSource().Task.Wait();
      }
  }

  /// <summary>
  /// 文件配置程序超类
  /// </summary>
  public class FileConfigurationProvider
  {
      private CancellationTokenSource? tokenSource;

      public void Load()
      {
          Console.WriteLine($"[{DateTime.Now}]文件已加载...");
      }

      public void Watch()
      {
          Load();
          //将changeToken生产者和changeToken消费者进行绑定(订阅)
          ChangeToken.OnChange(GetReloadToken, Load);
          //触发Change事件，通知更新
          var t = new Thread(() =>
          {
              while (true)
              {
                  Thread.Sleep(3000);
                  var t = tokenSource;
                  tokenSource = null;//取消之前一定要设置成null          
                  t!.Cancel();//执行回调，发布取消事件。
              }
          });
          t.Start();
      }

      /// <summary>
      /// 更新令牌，通过该令牌可以注册回调，用于执行更新通知。
      /// </summary>
      /// <returns></returns>
      public IChangeToken GetReloadToken()
      {
          lock (this)
          {
              //如果被消费就创建一个新的
              if (tokenSource == null)
              {
                  tokenSource = new CancellationTokenSource();
              }
              return new CancellationChangeToken(tokenSource.Token);
          }
      }
  }
  ```
- 执行原理

  CancellationChangeToken：接收到了CancellationToken，并把回调注册到CancellationToken上。

  CancellationToken：构造器接收CancellationTokenSource即注册到CancellationToken的回调，本质上是注册到了CancellationTokenSource上。CancellationToken充当CancellationChangeToken和CancellationTokenSource的中间人。

  CancellationTokenSource：执行Cancel，执行注册的回调，即执行了CancellationChangeToken中注册的回调。
- 执行流程

  1. 首先执行Watch()
  2. 然后执行ChangeToken.OnChange
  3. ChangeToken.OnChange执行生成者委托GetReloadToken，获取一个IChangeToken，然后注册了一个嵌套回调使得消费者和生产者永久绑定
  4. 3秒之后CancellationTokenSource通知取消，并执行CancellationTokenSource上的回调。

## 监听更改

```C#
static void Main(string[] args)
{
    var configurationBuilder = new ConfigurationManager();
    //json配置源
    configurationBuilder.SetBasePath(Directory.GetCurrentDirectory())
        //需要设置reloadOnChange为true
        .AddJsonFile(path: "appsettings.json", optional: true, reloadOnChange: true);
    //构建
    IConfiguration configuration = configurationBuilder;
    //绑定生产者和消费者
    ChangeToken.OnChange(configuration.GetReloadToken, () => 
    {
        Console.WriteLine("配置更新了：Port = " + configuration["MvcOptions:Port"]);
    });
    //阻塞
    new TaskCompletionSource().Task.Wait();
}
```

## 自定义

如何自定义，希望大家掌握思路，我们可以通过模仿json扩展

```C#
//配置提供器选项：用于提供配置选项
public class HttpConfigurationSource : IConfigurationSource
{
    public bool ReloadOnChange { get; set; }

    public HttpConfigurationSource()
    {

    }

    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new HttpConfigurationProvider(this);
    }
}
//配置提供器：配置源逻辑
public class HttpConfigurationProvider : IConfigurationProvider
{
    private ConcurrentDictionary<string, string> values = new ConcurrentDictionary<string, string>();
    private HttpConfigurationSource options;
    private CancellationTokenSource? tokenSource;
    public HttpConfigurationProvider(HttpConfigurationSource options)
    {
        this.options = options;
        //如果需要监听
        if (this.options.ReloadOnChange)
        {
            Watch();
        }
    }
    private void Watch()
    {   
        //注册事件
        ChangeToken.OnChange(GetReloadToken, Load);
        //模拟更改
        var t = new Thread(() =>
        {

            while (true)
            {
                var token = tokenSource;
                tokenSource = null;
                //每3s之后发生更改
                Thread.Sleep(3000);
                //触发事件,触发之前一定要将tokenSource设置成null
                token!.Cancel();
            }
        });
        t.Start();
    }
    public IEnumerable<string> GetChildKeys(IEnumerable<string> earlierKeys, string parentPath)
    {
        return values.Keys;
    }

    public IChangeToken GetReloadToken()
    {
        lock (this)
        {
            if (tokenSource == null)
            {
                tokenSource = new CancellationTokenSource();
            }
            return new CancellationChangeToken(tokenSource!.Token);
        }
    }

    public void Load()
    {
        //假设我们从第三方地址获取
        //var client = new HttpClient();
        //var response = client.GetAsync(source.Url).GetAwaiter().GetResult();
        //var json = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
        values.TryAdd("t1", "1111111");
        values.TryAdd("t2", "2222222");
        Console.WriteLine("ison文件已加载...");
    }

    public void Set(string key, string value)
    {
        values.TryAdd(key, value);
    }

    public bool TryGet(string key, out string value)
    {
        var flag = values.TryGetValue(key, out string? data);
        value = data ?? string.Empty;
        return flag;
    }
}
//扩展IConfigurationBuilder
public static class HttpConfigurationExtensions
{
    public static IConfigurationBuilder AddJsonHttp(this IConfigurationBuilder builder, Action<HttpConfigurationSource> configure)
    {
        if (builder == null)
        {
            throw new ArgumentNullException(nameof(builder));
        }
        /*
         * Add会执行如下逻辑
         * 实列化HttpConfigurationSource
         * 执行委托配置HttpConfigurationSource实列
         * 调用ttpConfigurationSource实列的Build返回HttpConfigurationProvider实列
         * HttpConfigurationProvider实列又依赖了HttpConfigurationSource实列
         * 最后执行Load加载配置到ConfigurationManager实列上
         */
        return builder.Add(configure);
    }
}
```
