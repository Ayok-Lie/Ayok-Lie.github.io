---
abbrlink: ''
categories: []
date: '2025-08-04T11:28:41.270714+08:00'
excerpt: AOP（面向切面编程）将日志、异常、权限等横切逻辑与业务解耦。在 .NET Core 中，借助 Castle DynamicProxy 或 源生成器，结合依赖注入，可轻松实现方法拦截与切面注入。本文带你用最简方式落地
  AOP，提升代码整洁度与可维护性。
sticky: ''
tags:
- '.Net基础原理 '
title: .Net Core基础—Aop
updated: '2025-10-18T09:20:28.122+08:00'
---
![](https://raw.githubusercontent.com/Ayok-Lie/images.house/main/Blog/Images/20250805103624014.png?token=AWT4OJBYG7FYIKHJ5UNBG43ISFXGO)

## 基础知识

假设这是一个U型管道，污水水从一端流入，另一端流出。

现在我们要对污水进行过滤，A负责过滤B负责消毒。显然有四个处理点，他们的顺序分别是：1->2->3->4。其中1，4是A过滤器的处理点，2，3是B过滤器的处理点。显然3个过滤器就有6个处理点。我们可以随意调整A，B过滤器的顺序，可随意插拔。这就是AOP的思想。

执行顺序：先进后出（栈）

执行点数：过滤器数 * 2

AOP是对OOP的一种补充，即面向切面编程，一种编程思想。我们管A，B为切面。1~4为切入点。AOP的优势是面向切面编程，每个切面负责独立的系统逻辑，降低代码的复杂度，提高代码的复用率。可以随意调整顺序，随意插拔。用于对业务逻辑进行增强。面向切面编程可以使得系统逻辑和业务逻辑进行分离。

系统逻辑：比如身份认证，异常处理，参数校验

业务逻辑：就是我们真正关心不得不写的业务逻辑。

```C#
public class Demo
{
    //A可以拦截所有异常
    public void A()
    {
        try
        {
            Console.WriteLine(1);
            B();
            Console.WriteLine(4);
        }
        catch(Exception e)
        {
            Console.WriteLine(e.Message);
        }
    }
    //B只能拦截自己和C的异常
    public void B()
    {
        Console.WriteLine(2);
        C();
        Console.WriteLine(3);
    }
    public void C()
    {
        Console.WriteLine("水变清澈了");
    }
}
//这段代码没有实际意义，但是它展示了函数调用的执行过程。（先进后出）
```

## 静态代理

假设我们需要实现一个IList接口，我们知道IList接口有很多方法，实现成本非常高。我们可以通过代理模式来实现

代理模式可以降低实现的成本，还可以对目标对象进行加强。代理者不需要实现具体的业务逻辑，只需要编写加强逻辑即可。

```C#
//实现IEnumerable接口只能加强两个方法，但是实现IList接口可以加强很多方法
class MyCollection : IEnumerable<object>
{
    private IEnumerable<object> _target;

    public MyCollection(IEnumerable<object> target)
    {
        _target = target;
    }

    public IEnumerator<object> GetEnumerator()
    {
        //编写加强逻辑比如打印
        Console.WriteLine("调用迭代器了");
        //通过target来实现，代理类之关系加强逻辑，不关心接口实现
        return _target.GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return ((IEnumerable)_target).GetEnumerator();
    }
}

//测试
var target= new List<object>();
//可以看到MyCollection对target进行了代理，加强了GetEnumerator函数（可以打印消息）
var collection = new MyCollection(target);
//此时GetEnumerator就会被加强，返回target的迭代器。
var it = collection.GetEnumerator();
```

我们可以通过静态代理来实现链式调用，完成污水处理问题。

```C#
public interface IWater
{
     void Invoke();
}
public class Water : IWater
{
    public void Invoke()
    {
        //业务逻辑
        Console.WriteLine("水已经净化了");
    }
}
//实现目标对象的接口IWater
public class WaterProxy1 : IWater
{
    private readonly IWater _target;
  
    public WaterProxy1(IWater target)
    {
        _target = target;
    }
   
    public void Invoke()
    {
        Console.WriteLine("开始消毒杀菌");//系统逻辑
        _target = target;
        Console.WriteLine("完成消毒杀菌");//系统逻辑
    }
}
public class WaterProxy2 : IWater
{
    private readonly IWater _target;
  
    public WaterProxy2(IWater target)
    {
        _target = target;
    }
   
    public void Invoke()
    {
        Console.WriteLine("开始去除杂质");//系统逻辑
        _target = target;
        Console.WriteLine("完成去除杂质");//系统逻辑
    }
}
//此时是先去除杂质后在消毒
//p1的target是Water，p2的target是p1
var target = new Water();
var p1 = new WaterProxy1(target);
var p2 = new WaterProxy2(p1);
p2.Invoke();
//此时是先消毒后在去除杂质
//p2的target是Water，p1的target是p2
var target = new Water();
var p2 = new WaterProxy2(target);
var p1 = new WaterProxy1(p2);
p2.Invoke();
```

可以看到系统逻辑和业务逻辑进行了分离，系统逻辑写到了不同的切面。切面之间何以随意组合，增减。这就是AOP思想的一种呈现方式。代码服用度很高，可以代理所有的IWater的实现。（假设Mercury也实现了IWater接口，那么WaterProxy1和WaterProxy2也能对他进行增强）

静态代理的本质是子类继承父类，或者实现接口，对目标对象进行增强。

静态代理的弊端是只能实现一个接口（标准），无法代理其他类型的实列。他的切面的可复用率有限，限定在它实现的接口。

## 管道模式

这是aspnetcore管道的核心代码

```C#
public class HttpContext
{
    //表示请求参数
    public string Request { get; }
    //表示响应数据
    public string Response { get; }
}

//定义一个委托
public delegate void RequestDelegate(HttpContext context);

public class ApplicationBuilder
{
    private readonly List<Func<RequestDelegate, RequestDelegate>> _components = n

    public void Use(Func<RequestDelegate, RequestDelegate> middleware)
    {
        _components.Add(middleware);
    }

    public RequestDelegate Build()
    {
        //负责兜底
        WaterDelegate app = c =>
        {
            throw new InvalidOperationException("无效的管道");
        };
        for (int i = _components.Count - 1; i > -1; i--)
        {
            app = _components[i](app);//完成嵌套
        }
        return app;
    }  
}  
```

测试

```C#
var builder = new ApplicationBuilder();
//过滤器1
builder.Use(next => 
{
    return context =>
    {
        Console.WriteLine("开始去污");
        next(context);
        Console.WriteLine("完成去污");
    };
});
//过滤器2
builder.Use(next =>
{
    return context =>
    {
        Console.WriteLine("开始消毒");
        next(context);
        Console.WriteLine("完成消毒");
    };
});
//端点
builder.Use(next =>
{
    return context =>
    {
    	Console.WriteLine("Hello World!");
    };
});
//构建管道
var app = builder.Build();
var context = new HttpContext();
//开始处理
app.Invoke(context);
```

## 模板方法模式

模板类实现接口，如果是重写必须加上sealed关键词来修饰，防止子类重写覆盖

```C#
public interface IServlet
{
    Task InvokeAsync(HttpContext context);
}
//如果IServlet是一个抽象类，InvokeAsync是一个虚函数，则必须使用sealed来保护模板方法
public abstract class HttpServlet : IServlet
{
    //模板方法，定义执行线路
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Method == "Get")
        {
            await GetAsync(context);
        }
        else if (context.Method == "Post")
        {
            await PostAsync(context);
        }
    }
	//具体线路通过抽象方法由子类来实现
    public abstract Task GetAsync(HttpContext context);
    public abstract Task PostAsync(HttpContext context);
}
//子类分别实现GetAsync和PostAsync线路
public class HelloServlet : HttpServlet
{
    public override Task GetAsync(HttpContext context)
    {
        Console.WriteLine("Hello World");
        return Task.CompletedTask;
    }

    public override Task PostAsync(HttpContext context)
    {
        Console.WriteLine("Hello World");
        return Task.CompletedTask;
    }
}
```

## 链路器模式

链路器负责链接切面，一般提供一个链路器接口负责适配不同类型的切面调用。

```C#
public class HttpContext
{
    public string Method { get; }

    public HttpContext(string method)
    {
        Method = method;
    }
}
//链路器
public interface IChain
{
    Task NextAsync();
}
//用于执行filter
public interface IFilter
{
    Task InvokeAsync(HttpContext context, IChain chain);
}
public interface IServlet
{
    Task InvokeAsync(HttpContext context);
}
//适配Filter链路，用于执行Filter
public class FilterChain : IChain
{
    private readonly IFilter _filter;
    private readonly HttpContext _context;
    private readonly IChain _next;
    public FilterChain(IFilter filter, HttpContext context, IChain next)
    {
        _filter = filter;
        _context = context;
        _next = next;
    }
    public async Task NextAsync()
    {
        await _filter.InvokeAsync(_context, _next);
    }
}
//适配Servlet链路，用于执行servlet
public class ServletChain : IChain
{
    private readonly IServlet _servlet;
    private readonly HttpContext _context;

    public ServletChain(IServlet servlet, HttpContext context)
    {
        _servlet = servlet;
        _context = context;
    }

    public async Task NextAsync()
    {
        await _servlet.InvokeAsync(_context);
    }
}
public class Filter1 : IFilter
{
    public async Task InvokeAsync(HttpContext context, IChain chain)
    {
        Console.WriteLine("身份认证开始");
        await chain.NextAsync();
        Console.WriteLine("身份认证结束");
    }
}
public class Filter2 : IFilter
{
    public async Task InvokeAsync(HttpContext context, IChain chain)
    {
        Console.WriteLine("授权认证开始");
        await chain.NextAsync();
        Console.WriteLine("授权认证结束");
    }
}
/// <summary>
/// 模板方法设计模式
/// </summary>
public abstract class HttpServlet : IServlet
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Method == "Get")
        {
            await GetAsync(context);
        }
        else if (context.Method == "Post")
        {
            await PostAsync(context);
        }
    }

    public abstract Task GetAsync(HttpContext context);
    public abstract Task PostAsync(HttpContext context);
}
public class HelloServlet : HttpServlet
{
    public override Task GetAsync(HttpContext context)
    {
        Console.WriteLine("Hello World");
        return Task.CompletedTask;
    }

    public override Task PostAsync(HttpContext context)
    {
        Console.WriteLine("Hello World");
        return Task.CompletedTask;
    }
}
public class WebHost
{
    private readonly List<IFilter> _filters = new List<IFilter>();

    public void AddFilter(IFilter filter)
    {
        _filters.Add(filter);
    }

    public async Task ExecuteAsync(HttpContext context, IServlet servlet)
    {
        if (_filters.Count > 0)
        {
            var filter = _filters[0];
            var chain = GetFilterChain(context, servlet, _filters.ToArray(), 1);
            await filter.InvokeAsync(context, chain);
        }
        else
        {
            await servlet.InvokeAsync(context);
        }
    }
    /// <summary>
    /// 构建链路器-递归
    /// </summary>
    /// <returns></returns>
    private IChain GetFilterChain(HttpContext context, IServlet servlet, IFilter[] filters, int index)
    {
        if (index < filters.Length)
        {
            var filter = filters[index];
            //递归构建下一个链路器
            var next = GetFilterChain(context, servlet, filters, index + 1);
            return new FilterChain(filter, context, next);
        }
        else
        {
            return new ServletChain(servlet, context);
        }
    }
}

//测试
var host = new WebHost();
host.AddFilter(new Filter1());
host.AddFilter(new Filter2());
var servlet = new HelloServlet();
await host.ExecuteAsync(new HttpContext("Get"), servlet);
```

## 动态代理

动态代理可以通过Castle.Core来实现。我们说静态代理和动态代理的区别是，静态代理在代码编译之前就已经确立的代理关系。而动态代理的原理是，在编译之后，运行时通过Emit来动态创建目标对象的子类，或者实现目标对象的接口。把拦截器织入到动态生成的类中，这里的拦截器可以织入到任意的实现类中。（Emit技术可以在运行时生成一个class，大家可以通过打印castle.core返回的实列的类名来进行验证）

动态代理和静态代理的本质都是继承或者实现，但是静态代理是需要手动编写代理类，而动态代理由框架动态生成代理类。动态代理性能更差，对异步支持不友好。

注意：如果是通过实现类的方式，那么无论静态代理还是动态代理，都只能代理父类中的虚函数(virtual)，因为子类只能重写父类中的虚函数。所以建议使用接口的方式。

```C#
public interface IWater
{
    void Invoke();
}
public class Water
{
    public void Invoke()
    {
        //业务逻辑
     	Console.WriteLine("水已经净化了");
    }   
}
//拦截器1-切面
public class Interceptor1 : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
         Console.WriteLine("开始去除杂质");//系统逻辑
         invocation.Proceed();
         Console.WriteLine("完成去除杂质");//系统逻辑
    }
}
//拦截器2-切面
public class Interceptor2 : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
         Console.WriteLine("开始消毒");//系统逻辑
         invocation.Proceed();
         Console.WriteLine("完成消毒");//系统逻辑
    }
}
var generator = new ProxyGenerator();
var target = new Water();
//通过框架生成实现IWater接口的代理类
var proxy = generator.CreateInterfaceProxyWithTarget<IWater>(target,new Interceptor1(),new Interceptor2());
Console.WriteLine(proxy.GetType().FullName);//可以看到这个类并不是我们生成的
proxy.Invoke();
//如果通过继承方式会怎么样？
```

## 手写Castle.Core的代理类

```C#
public interface IWater
{
    void Invoke();
}
public class Water:IWater
{
    public void Invoke()
    {
        //业务逻辑
        Console.WriteLine("水已经净化了");
    }
}
//拦截器1-切面
public class Interceptor1 : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine("开始去除杂质");//系统逻辑
        invocation.Proceed();
        Console.WriteLine("完成去除杂质");//系统逻辑
    }
}
//拦截器2-切面
public class Interceptor2 : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine("开始消毒");//系统逻辑
        invocation.Proceed();
        Console.WriteLine("完成消毒");//系统逻辑
    }
}

public class IWaterProxy : IWater
{
    private readonly IWater _target;
   
    private readonly IInterceptor[] _interceptors;

    public IWaterProxy(IWater target,params IInterceptor[] interceptors)
    {
        _target = target;
        _interceptors = interceptors;
    }

    public void Invoke()
    {
        var method = _target.GetType().GetMethod(nameof(IWater.Invoke));
        InvocationUtilities.Invoke(_interceptors, _target, method,new object[] { });
    }
}

public static class InvocationUtilities
{
    private static IInvocation GetNextInvocation(IInvocation target, IInterceptor[] inteceptors, int index)
    {
        if (index < inteceptors.Length)
        {
            var proxy = inteceptors[index];
            var next = GetNextInvocation(target, inteceptors, index + 1);
            var args = new object[] { next };
            var method = typeof(IInterceptor).GetMethod(nameof(IInterceptor.Intercept));
            return new NextInvocation(proxy, args, method);
        }
        else
        {
            return target;
        }
    }

    public static void Invoke(IInterceptor[] interceptors, object target, MethodInfo? method, object[] arguments)
    {
        if (method == null)
        {
            throw new ArgumentNullException(nameof(method));
        }
        var targetInvocation = new NextInvocation(target, arguments, method);
        if (interceptors.Any())
        {
            var inteceptor = interceptors[0];
            var invocation = GetNextInvocation(targetInvocation, interceptors, 1);
            inteceptor.Intercept(invocation);
        }
        else
        {
            targetInvocation.Proceed();
        }
    }
    //因为是反射调用，因此只需要实现一个链路器方案就好了
    class NextInvocation : IInvocation
    {
        public object[] Arguments { get; }
        public object InvocationTarget { get; }

        public MethodInfo Method  { get; }
   
        public void Proceed()
        {
            Method.Invoke(InvocationTarget, Arguments);
        }

        public NextInvocation(object target, object[] arguments,  MethodInfo method)
        {
            Arguments = arguments;
            Method = method;
            InvocationTarget = target;
        }



        #region Other
        public Type[] GenericArguments => throw new NotImplementedException();


        public MethodInfo MethodInvocationTarget => throw new NotImplementedException();

        public object Proxy => throw new NotImplementedException();

        public object ReturnValue { get => throw new NotImplementedException(); set => throw new NotImplementedException(); }

        public Type TargetType => throw new NotImplementedException();

        public IInvocationProceedInfo CaptureProceedInfo()
        {
            throw new NotImplementedException();
        }

        public object GetArgumentValue(int index)
        {
            throw new NotImplementedException();
        }

        public MethodInfo GetConcreteMethod()
        {
            throw new NotImplementedException();
        }

        public MethodInfo GetConcreteMethodInvocationTarget()
        {
            throw new NotImplementedException();
        }

   

        public void SetArgumentValue(int index, object value)
        {
            throw new NotImplementedException();
        }
        #endregion
    }
}

//测试
 var target = new Water();
 IWater proxy = new IWaterProxy(target,new Interceptor1(),new Interceptor2());
 proxy.Invoke();
```

## 手写Castle.Core的ProxyGenerator

```C#
public interface IInvocation
{
    void Proceed();
}
public class InvocationInteceptor : IInvocation
{
    private IInteceptor _inteceptor;
    private IInvocation _invocation;
    public InvocationInteceptor(IInteceptor inteceptor, IInvocation invocation)
    {
        _inteceptor = inteceptor;
        _invocation = invocation;
    }

    public void Proceed()
    {
        _inteceptor.Intercept(_invocation);
    }
}
public class InvocationTraget : IInvocation
{
    private object instance;
    private MethodInfo method;
    private object[] arguments;

    public InvocationTraget(object instance, MethodInfo method, object[] arguments)
    {
        this.instance = instance;
        this.method = method;
        this.arguments = arguments;
    }

    public void Proceed()
    {
        method.Invoke(instance, arguments);
    }
}
//拦截器
public interface IInteceptor
{
    void Intercept(IInvocation invocation);
}

//该工具类帮助我们少写emit代码
public static class InvocationUtilities
{
    private static IInvocation GetNextInvocation(IInvocation target, IInteceptor[] inteceptors, int index)
    {
        if (index < inteceptors.Length)
        {
            var next = GetNextInvocation(target, inteceptors, index + 1);
            return new InvocationInteceptor(inteceptors[index], next);
        }
        else
        {
            return target;
        }
    }

    public static void Invoke(IInteceptor[] interceptors, object target, MethodInfo? method, object[] arguments)
    {
        if (method == null)
        {
            throw new ArgumentNullException(nameof(method));
        }
        if (interceptors.Any())
        {
            var inteceptor = interceptors[0];
            var invocation = GetNextInvocation(new InvocationTraget(target, method, arguments), interceptors, 1);
            inteceptor.Intercept(invocation);
        }
        else
        {
            method.Invoke(target, arguments);
        }
    }
}

//实现拦截器1
public class Inteceptor1 : IInteceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine("prox1:start");
        invocation.Proceed();
        Console.WriteLine("prox1:end");
    }
}

//实现拦截器1
public class Inteceptor2 : IInteceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine("prox2:start");
        invocation.Proceed();
        Console.WriteLine("prox2:end");
    }
}

public interface ILogger
{
    void Log(string message);
}

public class ConsoleILogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine("Console:" + message);
    }
}

public class ConsoleILoggerProxy : ILogger
{
    private readonly ILogger _target;
    private readonly IInteceptor[] _inteceptors;

    public ConsoleILoggerProxy(ILogger target, IInteceptor[] inteceptors)
    {
        _target = target;
        _inteceptors = inteceptors;
    }

    public void Log(string message)
    {
        var method = _target.GetType().GetMethod("Log");
        var args = new object[1];
        args[0] = message;
        InvocationUtilities.Invoke(_inteceptors, _target, method, args);
    }
}

public static class ProxyGenerator
{
    static AssemblyBuilder _assemblyBuilder;
    static ModuleBuilder _moduleBuilder;
    static ProxyGenerator()
    {
        //创建一个程序集
        var assemblyName = new AssemblyName("DynamicProxies");
        _assemblyBuilder = AssemblyBuilder
            .DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.Run);
        //创建一个模块
        _moduleBuilder = _assemblyBuilder.DefineDynamicModule("Proxies");
    }
    public static TInterface Create<TInterface>(object target, params IInteceptor[] inteceptor)
        where TInterface : class
    {

        #region 定义类型
        //定义一个class，如果这个类型已定义直接返回，缓存
        var typeName = $"{target.GetType().Name}EmitProxy";
        var typeBuilder = _moduleBuilder.DefineType(
            typeName,
            TypeAttributes.Public, 
            typeof(object),
            new Type[]
            {
                typeof(TInterface)
            });
        #endregion

        #region 定义字段
        //定义字段
        var targetFieldBuilder = typeBuilder.DefineField("target", typeof(object), FieldAttributes.Private);
        var inteceptorFieldBuilder = typeBuilder.DefineField("inteceptor", typeof(IInteceptor[]), FieldAttributes.Private);
        #endregion

        #region 定义构造器
        //定义构造器
        var constructorBuilder = typeBuilder.DefineConstructor(MethodAttributes.Public, CallingConventions.ExplicitThis, new Type[]
        {
            typeof(TInterface),
            typeof(IInteceptor[])
        });
        //获取IL编辑器
        var generator = constructorBuilder.GetILGenerator();
        generator.Emit(OpCodes.Ldarg_0);//加载this
        generator.Emit(OpCodes.Call, typeof(object).GetConstructor(Type.EmptyTypes) ?? throw new InvalidOperationException());
        generator.Emit(OpCodes.Nop);
        // this._target = target;
        generator.Emit(OpCodes.Ldarg_0);//加载this
        generator.Emit(OpCodes.Ldarg_1);//加载target参数
        generator.Emit(OpCodes.Stfld, targetFieldBuilder);//加载target字段
        // this._inteceptor = inteceptor;
        generator.Emit(OpCodes.Ldarg_0);//加载this
        generator.Emit(OpCodes.Ldarg_2);//加载inteceptor参数
        generator.Emit(OpCodes.Stfld, inteceptorFieldBuilder);//加载inteceptor字段
        generator.Emit(OpCodes.Ret);

        #endregion

        #region 实现接口
        var methods = typeof(TInterface).GetMethods();
        foreach (var item in methods)
        {
            var parameterTypes = item.GetParameters().Select(s => s.ParameterType).ToArray();
            var methodBuilder = typeBuilder.DefineMethod(item.Name,
                MethodAttributes.Public | MethodAttributes.Final | MethodAttributes.Virtual | MethodAttributes.NewSlot | MethodAttributes.HideBySig,
                CallingConventions.Standard | CallingConventions.HasThis,
                item.ReturnType,
                parameterTypes);
            var generator1 = methodBuilder.GetILGenerator();
            //init
            var methodInfoLocal = generator1.DeclareLocal(typeof(MethodInfo));
            var argumentLocal = generator1.DeclareLocal(typeof(object[]));
            generator1.Emit(OpCodes.Nop);//{
            // MethodInfo method = this.GetType().GetMethod("Log");
            generator1.Emit(OpCodes.Ldarg_0);
            generator1.Emit(OpCodes.Ldfld, targetFieldBuilder);//this._target
            generator1.Emit(OpCodes.Callvirt, typeof(object).GetMethod(nameof(Type.GetType), Type.EmptyTypes));
            generator1.Emit(OpCodes.Ldstr, item.Name);
            generator1.Emit(OpCodes.Callvirt, typeof(Type).GetMethod(nameof(Type.GetMethod), new Type[] { typeof(string)}));
            generator1.Emit(OpCodes.Stloc, methodInfoLocal);
            // object[] array = new object[0];
            generator1.Emit(OpCodes.Ldc_I4,1);
            generator1.Emit(OpCodes.Newarr, typeof(object));
            generator1.Emit(OpCodes.Stloc, argumentLocal);
            // array[0] = message;
            generator1.Emit(OpCodes.Ldloc, argumentLocal);
            generator1.Emit(OpCodes.Ldc_I4, 0);
            generator1.Emit(OpCodes.Ldarg_1);
            generator1.Emit(OpCodes.Stelem_Ref);
            // InvocationUtilities.Invoke(_inteceptors, _target, method, array);
            generator1.Emit(OpCodes.Ldarg_0);
            generator1.Emit(OpCodes.Ldfld, inteceptorFieldBuilder);//this._interceptor
            generator1.Emit(OpCodes.Ldarg_0);
            generator1.Emit(OpCodes.Ldfld, targetFieldBuilder);//this._target
            generator1.Emit(OpCodes.Ldloc, methodInfoLocal);
            generator1.Emit(OpCodes.Ldloc, argumentLocal);
            generator1.Emit(OpCodes.Call, typeof(InvocationUtilities).GetMethod(nameof(InvocationUtilities.Invoke)));
            generator1.Emit(OpCodes.Nop);
            generator1.Emit(OpCodes.Ret);
        }
        #endregion
        //创建:这个type可以用一个线程安全的字典缓存起来，第二次需要这个代理类的时候，就不需要在生成一次emit代码了。
        var type = typeBuilder.CreateType() ?? throw new ArgumentException();
        var instance = Activator.CreateInstance(type, target, inteceptor);
        return (TInterface)instance;
    }
}
//测试
var target = new ConsoleILogger();
var logger = ProxyGenerator.Create<ILogger>(target, new Inteceptor1(), new Inteceptor2());
logger.Log("ff");
```
