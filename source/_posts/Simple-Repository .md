---
abbrlink: ''
categories:
- - C#
date: '2025-10-31T17:07:32.253045+08:00'
excerpt: 从之前的仓储文档中精简出来的简易版本
sticky: ''
tags:
- 设计模式
- .Net基础原理
title: 简易版仓储
updated: '2025-10-31T17:07:34.068+08:00'
---
## 简易版仓储：从 EF 仓储到带工作单元（UoW）

在现代 .NET 开发中，**仓储模式（Repository Pattern）** 是实现数据访问层与业务逻辑解耦的关键手段。它将数据操作封装起来，使业务代码无需关心底层数据库的具体实现细节。

本文将通过一个简洁的 Demo，逐步演示两种常见的仓储实现方式：

1. **基础 EF 仓储**（Generic EF Repository）
2. **带工作单元（Unit of Work, UoW）的 EF 仓储**

我们将从公共实体定义出发，依次展示上下文、仓储实现、服务注册与使用方式，并分析各自的优缺点。

---

### 🧱 公共实体

所有示例均基于以下通用实体：

```csharp
public class product
{
    public int Id { get; set; }
    public string Name { get; set; }
}

```

## 1️⃣ 基础 EF 仓储（Generic EF Repository）

这是最常见、最直观的仓储实现方式，直接封装 `DbSet<T>` 的常用操作。

### 📁 数据库上下文

```
public class AppDbContext : DbContext
{
    public DbSet<product> product { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }
}
```

### 📦 仓储接口与实现

```
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}
```

```
public class Repository<T> : IRepository<T> where T : class
{
    private readonly AppDbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T> GetByIdAsync(int id) =>
        await _dbSet.FindAsync(id);

    public async Task<IEnumerable<T>> GetAllAsync() =>
        await _dbSet.ToListAsync();

    public async Task AddAsync(T entity) =>
        await _dbSet.AddAsync(entity);

    public async Task UpdateAsync(T entity) =>
        _dbSet.Update(entity);

    public async Task DeleteAsync(int id)
    {
        var entity = await _dbSet.FindAsync(id);
        if (entity != null)
            _dbSet.Remove(entity);
    }
}
```

> ⚠️ 注意：此实现**未调用 `_context.SaveChangesAsync()`**，需由上层统一提交变更。

### 🔧 服务注册（`Program.cs`）

```
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 注册泛型仓储
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// 注册业务服务
builder.Services.AddScoped<ProductService>();

var app = builder.Build();
app.Run();
```

### ✅ 优点 vs ❌ 缺点


| 优点                                                                           | 缺点                                                                                                            |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| ✅ 支持 LINQ 查询、强类型安全<br />✅ 与 EF Core 无缝集成<br />✅ 易于单元测试 | ❌**无法自动管理事务** <br />❌ 多个仓储操作需手动调用 `SaveChanges`<br />❌ 容易造成“过早提交”或“重复提交” |

> 📌 **适用场景**：小型项目、快速原型、单实体操作为主的应用。

---

## 2️⃣ 带工作单元（UoW）的 EF 仓储

为解决事务一致性问题，我们引入 **工作单元（Unit of Work）** 模式。它确保多个仓储操作在同一个数据库上下文中执行，并通过一次 `SaveChanges` 提交所有变更。

### 📁 数据库上下文（复用或独立）

```
public class UnitOfWorkDbContext : DbContext
{
    public UnitOfWorkDbContext(DbContextOptions<UnitOfWorkDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.ApplyConfiguration(new ProductConfiguration());

    }
}
```

### 🧩 工作单元接口与实现

```
public interface IUnitOfWork
{
    int SaveChanges();
}
```

```
public class UnitOfWork<TDbContext> : IUnitOfWork where TDbContext : DbContext
{
    private readonly TDbContext _dbContext;

    public UnitOfWork(TDbContext context) => _dbContext = context ?? throw new ArgumentNullException(nameof(context));

    public Int32 SaveChanges() => _dbContext.SaveChanges();
}
```

### 📦 仓储实现（**不自动 SaveChanges**）

> ⚠️ 关键区别：**仓储层不再调用 `SaveChanges`**，由 UoW 统一控制！

```
public interface IRepository<T> where T : class
{
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> GetByIdAsync(Int32 id);
    T Add(T entity);
    T Update(T entity);
    Task DeleteAsync(Int32 id);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly UnitOfWorkDbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(UnitOfWorkDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<IEnumerable<T>> GetAllAsync() => await _dbSet.ToListAsync();

    public async Task<T> GetByIdAsync(Int32 id) => await _dbSet.FindAsync(id);

    public T Add(T entity)
    {
        var newEntity= _dbSet.Add(entity).Entity;
        //await _context.SaveChangesAsync();
        return newEntity;
    }

    public T Update(T entity)
    {
        AttachIfNot(entity);
        _context.Entry(entity).State = EntityState.Modified;
        //await _context.SaveChangesAsync();
        return entity;
    }

    public async Task DeleteAsync(Int32 id)
    {
        var entity = await GetByIdAsync(id);
        if (entity != null)
        {
            AttachIfNot(entity);
            _context.Entry(entity).State = EntityState.Deleted;
            _dbSet.Remove(entity);
           // await _context.SaveChangesAsync();
        }
    }

    protected virtual void AttachIfNot(T entity)
    {
        var entry = _context.ChangeTracker.Entries().FirstOrDefault(ent => ent.Entity == entity);
        if (entry != null)
        {
            return;
        }

        _dbSet.Attach(entity);
    }
}
```

### 🔧 服务注册（控制台或 Web 应用）

```
var services = new ServiceCollection();

services.AddDbContext<UnitOfWorkDbContext>(options =>
    options.UseSqlite("DataSource=test.db"));
services.AddScoped<IUnitOfWork, UnitOfWork<TContext>>();
services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
services.AddTransient<ProductService>();
var serviceProvider = services.BuildServiceProvider();
```

### 🚀 服务层使用示例

```
public class ProductService
{
    private readonly IRepository<Product> _productRepository;
    private readonly IUnitOfWork _unitOfWork;

    public ProductService(IRepository<Product> productRepository,
        IUnitOfWork unitOfWork)
    {
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task<IEnumerable<Product>> GetAllProductsAsync() => await _productRepository.GetAllAsync();

    public async Task<Product> GetProductByIdAsync(Int32 id) => await _productRepository.GetByIdAsync(id);

    public void AddProduct(Product product)
    {
        _productRepository.Add(product);
        _unitOfWork.SaveChanges();
    }

    public void UpdateProductAsync(Product product)
    {
        _productRepository.Update(product);
        _unitOfWork.SaveChanges();
    }

    public async Task DeleteProductAsync(Int32 id)
    {
        await _productRepository.DeleteAsync(id);
        _unitOfWork.SaveChanges();
    }
}
```
