---
abbrlink: ''
categories:
- - C#
date: '2025-08-07T14:38:32.234003+08:00'
excerpt: NetCore基于EntityFramework和Aop的工作单元模式(UnitOfWork)简单实现
sticky: ''
tags:
- .Net基础原理
- 设计模式
title: NetCore基于EntityFramework和Aop的工作单元模式(UnitOfWork)简单实现
updated: '2025-08-07T14:38:32.498+08:00'
---
## Unit of Work是什么

Unit of Work 是一种重要的设计模式。Unit of Work 模式提供了一种有效的方式来管理数据库事务和跟踪对数据库的所有更改。

使用 Unit of Work 模式的好处是多方面的。首先，它允许我们将多个数据库操作组合成一个事务。这意味着要么全部操作成功提交，要么都失败回滚。这确保了数据的完整性和一致性。

其次，Unit of Work 跟踪对数据库所做的所有更改。无论是插入、更新还是删除操作，都会被记录下来。在提交事务之前，可以检查并验证这些更改，以确保数据的正确性。

此外，通过优化数据库操作的执行顺序，Unit of Work 可以减少不必要的往返数据库的次数，从而提高性能。它还可以确保对同一实体的修改在同一个事务中进行，从而维护实体之间的一致性。

在实际开发中，Unit of Work 往往与仓储（Repository）模式一起使用。仓储负责处理数据的持久化和检索，而 Unit of Work 则负责管理仓储的工作单元和事务。这种组合可以提高代码的可维护性和可测试性

## 代码实现

创建工作单元的依赖接口：

```C#
    public interface IUnitOfWork : IDisposable
    {
        /// <summary>
        /// 开始事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        Task<IDbContextTransaction> BeginTransactionAsync(CancellationToken cancellationToken = default);

        /// <summary>
        /// 开始事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        IDbContextTransaction BeginTransaction();

        /// <summary>
        /// 保存更改
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);

        /// <summary>
        /// 提交事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        void CommitTransaction();

        /// <summary>
        /// 提交事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        Task CommitTransactionAsync(CancellationToken cancellationToken = default);

        /// <summary>
        /// 回滚事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        void RollbackTransaction();

        /// <summary>
        /// 回滚事务
        /// </summary>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        Task RollbackTransactionAsync(CancellationToken cancellationToken = default);
    }
```

接口实现

```C#
public sealed class UnitOfWork<TDbContext> : IUnitOfWork, IDisposable, ITransientDependency 
    where TDbContext : DbContext
{
    // 指示当前 UnitOfWork 是否已被释放
    public bool IsDisposed { get; private set; }

    // 指示当前 UnitOfWork 是否已完成
    public bool IsCompleted { get; private set; }

    // 数据库上下文实例
    private readonly TDbContext _dbContext;

    // 服务提供者实例，用于获取其他服务
    private readonly IServiceProvider _serviceProvider;

    /// <summary>
    /// 构造函数，初始化 UnitOfWork 实例
    /// </summary>
    /// <param name="dbContext">数据库上下文实例</param>
    public UnitOfWork(TDbContext dbContext)
    {
        _dbContext = dbContext ?? throw new ArgumentNullException($"db context {nameof(dbContext)} is null");
    }

    /// <summary>
    /// 异步开始一个数据库事务
    /// </summary>
    /// <param name="cancellationToken">取消令牌</param>
    /// <returns>返回一个 IDbContextTransaction 实例</returns>
    public async Task<IDbContextTransaction> BeginTransactionAsync(CancellationToken cancellationToken = default)
    {
        IsCompleted = false; // 标记事务未完成
        return await _dbContext.Database.BeginTransactionAsync(cancellationToken);
    }

    /// <summary>
    /// 开始一个数据库事务
    /// </summary>
    /// <returns>返回一个 IDbContextTransaction 实例</returns>
    public IDbContextTransaction BeginTransaction()
    {
        IsCompleted = false; // 标记事务未完成
        return _dbContext.Database.CurrentTransaction ?? _dbContext.Database.BeginTransaction();
    }

    /// <summary>
    /// 异步提交数据库事务
    /// </summary>
    /// <param name="cancellationToken">取消令牌</param>
    /// <returns>返回一个 Task 实例</returns>
    public async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (IsCompleted)
        {
            return; // 如果事务已完成，直接返回
        }

        IsCompleted = true; // 标记事务已完成
        try
        {
            // 保存更改并提交事务
            await _dbContext.SaveChangesAsync(cancellationToken).ConfigureAwait(false);
            await _dbContext.Database.CommitTransactionAsync(cancellationToken).ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            // 发生异常时回滚事务
            await RollbackTransactionAsync(cancellationToken).ConfigureAwait(false);
            throw new Exception(ex.Message);
        }
    }

    /// <summary>
    /// 提交数据库事务
    /// </summary>
    public void CommitTransaction()
    {
        if (IsCompleted)
        {
            return; // 如果事务已完成，直接返回
        }

        IsCompleted = true; // 标记事务已完成
        try
        {
            // 保存更改并提交事务
            _dbContext.SaveChanges();
            _dbContext.Database.CommitTransaction();
        }
        catch (Exception x)
        {
            // 发生异常时回滚事务
            RollbackTransaction();
            throw;
        }
    }

    /// <summary>
    /// 异步回滚数据库事务
    /// </summary>
    /// <param name="cancellationToken">取消令牌</param>
    /// <returns>返回一个 Task 实例</returns>
    public async Task RollbackTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (IsCompleted)
        {
            return; // 如果事务已完成，直接返回
        }
        await _dbContext.Database.RollbackTransactionAsync(cancellationToken).ConfigureAwait(false);
    }

    /// <summary>
    /// 回滚数据库事务
    /// </summary>
    public void RollbackTransaction()
    {
        if (IsCompleted)
        {
            return; // 如果事务已完成，直接返回
        }
        _dbContext.Database.RollbackTransaction();
    }

    /// <summary>
    /// 异步保存数据库上下文中的更改
    /// </summary>
    /// <param name="cancellationToken">取消令牌</param>
    /// <returns>返回保存更改的行数</returns>
    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _dbContext.SaveChangesAsync(cancellationToken).ConfigureAwait(false);
    }

    /// <summary>
    /// 释放资源
    /// </summary>
    public void Dispose()
    {
        if (IsDisposed)
        {
            return; // 如果已释放，直接返回
        }

        IsDisposed = true; // 标记已释放
    }
}
```

服务注入到容器中：

```
services.AddTransient<IUnitOfWork, UnitOfWork>(); // 注册工作单元到容器
```

此时，工作单元已经添加到容器中，为了更方便的使用，我们可以使用AspNetCore框架自带的过滤器实现AOP的方式来使用工作单元。

我们默认接口是启用工作单元的，但为了更加全面，添加一个工作单元特性

```C#
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
    public class DisabledUnitOfWorkAttribute : Attribute
    {
        public readonly bool Disabled;

        public DisabledUnitOfWorkAttribute(bool disabled = true)
        {
            Disabled = disabled;
        }
    }
```

为了避免每次执行操作都要手动调用一下工作单元来进行保存，我们添加一个全局过滤器。

```C#
    /// <summary>
    /// 工作单元Action过滤器
    /// </summary>
    public class UnitOfWorkFilter : IAsyncActionFilter, IOrderedFilter
    {

        private readonly ILogger<UnitOfWorkFilter> _logger;
        public UnitOfWorkFilter(ILogger<UnitOfWorkFilter> logger)
        {
            this._logger = logger;
        }
        /// <summary>
        /// 过滤器排序
        /// </summary>
        internal const int FilterOrder = 999;

        /// <summary>
        /// 排序属性
        /// </summary>
        public int Order => FilterOrder;

        /// <summary>
        /// 拦截请求
        /// </summary>
        /// <param name="context">动作方法上下文</param>
        /// <param name="next">中间件委托</param>
        public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
        {
            // 获取动作方法描述器
            var actionDescriptor = context.ActionDescriptor as ControllerActionDescriptor;
            var method = actionDescriptor.MethodInfo;

            // 获取请求上下文
            var httpContext = context.HttpContext;

            // 如果没有定义工作单元过滤器，则跳过
            if (method.IsDefined(typeof(DisabledUnitOfWorkAttribute), true))
            {
                // 调用方法
                _ = await next();

                return;
            }

            // 打印工作单元开始消息
            _logger.LogInformation($@"{nameof(UnitOfWorkFilter)} Beginning");

            // 解析工作单元服务
            var unitOfWorks = httpContext.RequestServices.GetServices<IUnitOfWork>();
            foreach (var unitOfWork in unitOfWorks)
            {
                // 开启事务
                await unitOfWork.BeginTransactionAsync();
            }
            try
            {
                await next();
                foreach (var unitOfWork in unitOfWorks)
                {
                    // 提交事务
                    await unitOfWork.CommitTransactionAsync();
                }

                _logger.LogInformation($@"{nameof(UnitOfWorkFilter)} Ending");
            }
            catch (Exception ex)
            {
                foreach (var d in unitOfWorks)
                {
                    await d.RollbackTransactionAsync();
                }
                throw;
            }
        }
    }
```

将过滤器添加到通信管道中

```C#
services.AddControllers(options =>
{
    // 添加 工作单元过滤器 
    options.Filters.Add<UnitOfWorkFilter>();
});
```

完结：散花(只是简单实现，存在很多不足。)
