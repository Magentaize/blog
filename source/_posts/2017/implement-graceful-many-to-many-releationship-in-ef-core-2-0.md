---
title: 在 EF Core 2.0 中实现优雅的多对多关系
date: 2017-11-10 10:50:53
categories:
 - Tech
tags:
 - EF
---

距离EF Core第一次发布已经过了很久很久，然而，缺失的Many To Many映射却迟迟没有加入Roadmap，并且在之后的一段时间内也很难看到EF Core团队对这一功能的态度发生变化。虽然说可以通过手动加入中间表的方式来用两个One To Many映射来替代，但是这无疑是非常影响流畅的编程体验。对此，本文将通过对DbContext的定义略作修改，来降低这种不便。
<!--more-->

>原文来自[Many-to-many relationships in EF Core 2.0 – Part 3: Hiding as ICollection](https://blog.oneunicorn.com/2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-3-hiding-as-icollection/)，有删改
>原作者保留一切权利

## 0x00 限制
在开始之前，我必须明确有两个限制不能被忽略：
* EF Core对任何没有映射的属性是一无所知的，这意味着对实体的查询依然要根据实体定义的关系来写。
* 中间表并不会消失，它仍会存在于应用程序中并且映射到数据库，只不过是在与实体交互时并不使用它。

解决这两个限制需要对EF Core的核心代码做出修改，在[issue 1368](https://github.com/aspnet/EntityFrameworkCore/issues/1368)被解决之前我不期望这些限制会消失。

## 0x01 最初的模型
根据官方的文档以及许多博客中所写的内容，定义一个Many To Many关系需要用中间表来建立两个One To Many映射，用最为简单明了的例子来说，那就是博客的Post和Tag的关系了。假设有以下理想实体类：
```csharp
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
 
    public ICollection<Tag> Tags { get; } = new List<Tag>();
}
 
public class Tag
{
    public int Id { get; set; }
    public string Text { get; set; }
 
    public ICollection<Post> Posts { get; } = new List<Post>();
}
```
之所以说这是一个理想模型，是因为EF Core并不支持直接定义这种关系，实际上这种关系隐含了一个中间表，这主要是关系型数据库的问题，它的数据结构就是这样定义的。

所以，实际上的实体类应该是这样定义的：
```csharp
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }
 
    public ICollection<PostTag> PostsTags { get; } = new List<PostTag>();
}
 
public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }
 
    public ICollection<PostTag> PostsTags { get; } = new List<PostTag>();
}

public class PostTag
{
    [Key]
    [Required]
    public Guid Id { get; set; }
    [Required]
    public Post Post { get; set; }
    [Required]
    public Tag Tag { get; set; }
}
```
中间表的属性我添加了`RequiredAttribute`特性，这会使EF Core自动建立外键和约束。

这样一来，关系映射出来确实是没错，但是每次要查询时，都要写`Posts.Include().ThenInclude().PostsTags.Tags`，非常麻烦，那么能不能把Tags作为公共属性导出，用一个get访问器来简化访问呢？

我们自然会想到这个问题，那答案当然是可以。

## 0x02 用 IEnumerable 来隐藏
为了将Tags属性导出，需要一个新的属性，于是实体类的定义会变成这样：
```csharp
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }
 
    public ICollection<PostTag> PostTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public IEnumerable<Tag> Tags => PostTags.Select(e => e.Tag);
}
 
public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }
 
    public ICollection<PostTag> PostTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public IEnumerable<Post> Posts => PostTags.Select(e => e.Post);
}
```
现在，进行多对多的查询会变得略微简单一些：
```csharp
var posts = context.Posts
    .Include(r => r.PostsTags)
    .ThenInclude(r => r.Tag)
    .ToList();
 
foreach (var post in posts)
{
    Console.WriteLine($"  Post {post.Title}");
    foreach (var tag in post.Tags)
    {
        Console.WriteLine($"    Tag {tag.Text}");
    }
}
```
但是如果想要添加数据，依然是非常复杂，需要手动向中间表插入数据，这该怎么办呢？

## 0x03 用 ICollection 来隐藏
为了能包装对另一个表的添加、删除操作，我们需要公开的这个属性是一个`ICollection`类型，以便它能够成为中间表的一个代理(Proxy)。

于是，这有一个这种集合的可选实现：
```csharp
public class JoinCollectionFacade<T, TJoin> : ICollection<T>
{
    private readonly ICollection<TJoin> _collection;
    private readonly Func<TJoin, T> _selector;
    private readonly Func<T, TJoin> _creator;
 
    public JoinCollectionFacade(
        ICollection<TJoin> collection,
        Func<TJoin, T> selector,
        Func<T, TJoin> creator)
    {
        _collection = collection;
        _selector = selector;
        _creator = creator;
    }
 
    public IEnumerator<T> GetEnumerator()
        => _collection.Select(e => _selector(e)).GetEnumerator();
 
    IEnumerator IEnumerable.GetEnumerator()
        => GetEnumerator();
 
    public void Add(T item)
        => _collection.Add(_creator(item));
 
    public void Clear()
        => _collection.Clear();
 
    public bool Contains(T item)
        => _collection.Any(e => Equals(_selector(e), item));
 
    public void CopyTo(T[] array, int arrayIndex)
        => this.ToList().CopyTo(array, arrayIndex);
 
    public bool Remove(T item)
        => _collection.Remove(
            _collection.FirstOrDefault(e => Equals(_selector(e), item)));
 
    public int Count
        => _collection.Count;
 
    public bool IsReadOnly
        => _collection.IsReadOnly;
}
```
> 译注：这个实现其实是有问题的，如果对集合进行LINQ查询，比如`ToList()`，会导致在`void CopyTo(T[] array, int arrayIndex)`函数中的堆栈溢出，不过接下来将会修正这个错误。

这个想法是比较简单的，大体上看来是把对实体类的操作转换成了对集合的操作，并且在需要时使用`selector`代理从联合实体中提取出目标实体。同样地，在添加时使用`creator`代理会用目标实体创建一个新的联合实体。

接下来需要对实体类的定义做出改变，要在实体类的构造函数中初始化这个代理的集合。
```csharp
public class Post
{
    public Post()
        => Tags = new JoinCollectionFacade<Tag, PostTag>(
            PostsTags,
            pt => pt.Tag,
            t => new PostsTag { Post = this, Tag = t });
 
    public int Id { get; set; }
    public string Title { get; set; }
 
    private ICollection<PostsTag> PostsTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public ICollection<Tag> Tags { get; }
}
 
public class Tag
{
    public Tag()
        => Posts = new JoinCollectionFacade<Post, PostTag>(
            PostsTags,
            pt => pt.Post,
            p => new PostsTag { Post = p, Tag = this });
 
    public int Id { get; set; }
    public string Text { get; set; }
 
    private ICollection<PostTag> PostsTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public ICollection<Post> Posts { get; }
}
```

在用ICollection代替了IEnumerable之后，就能在集合上进行更多的操作，查询代码也相应地可以这样来写：
```csharp
var tags = new[]
{
    new Tag { Text = "Golden" },
    new Tag { Text = "Pineapple" }
};
 
var posts = new[]
{
    new Post { Title = "Best Boutiques on the Eastside" },
    new Post { Title = "Avoiding over-priced Hipster joints" }
};
 
posts[0].Tags.Add(tags[0]);
posts[0].Tags.Add(tags[1]);
posts[1].Tags.Add(tags[0]);
posts[1].Tags.Add(tags[1]);
 
context.AddRange(tags);
context.AddRange(posts);
context.SaveChanges();
```
```csharp
foreach (var post in posts)
{
    var oldTag = post.Tags.FirstOrDefault(e => e.Text == "Pineapple");
    if (oldTag != null)
    {
        post.Tags.Remove(oldTag);
        post.Tags.Add(newTag1);
    }
    post.Tags.Add(newTag2);
}
 
context.SaveChanges();
```
注意：
* 在填充数据时，可以直接向Post的Tags集合中添加了实体
* 在查找或搜索时，可以直接搜索Tags并且从中移除已有的实体，而不经过中间表

## 0x04 为了更加抽象
对以上代码，其实可以进一步优化结构而增加复用度。

首先定义一个中间表实体的接口：
```csharp
public interface IJoinEntity<TEntity>
{
    TEntity Navigation { get; set; }
}
```

任何一个中间表都需要实现两次这个接口，为了两边映射的两个表：
```csharp
public class PostTag : IJoinEntity<Post>, IJoinEntity<Tag>
{
    public Post Post { get; set; }
    Post IJoinEntity<Post>.Navigation
    {
        get => Post;
        set => Post = value;
    }
 
    public Tag Tag { get; set; }
    Tag IJoinEntity<Tag>.Navigation
    {
        get => Tag;
        set => Tag = value;
    }
}
```

现在，可以用更强关系的泛型来重写JoinCollectionFacade集合：
```csharp
public class JoinCollectionFacade<TEntity, TOtherEntity, TJoinEntity> 
    : ICollection<TEntity>
    where TJoinEntity : IJoinEntity<TEntity>, IJoinEntity<TOtherEntity>, new()
{
    private readonly TOtherEntity _ownerEntity;
    private readonly ICollection<TJoinEntity> _collection;
 
    public JoinCollectionFacade(
        TOtherEntity ownerEntity,
        ICollection<TJoinEntity> collection)
    {
        _ownerEntity = ownerEntity;
        _collection = collection;
    }
 
    public IEnumerator<TEntity> GetEnumerator()
        => _collection.Select(e => ((IJoinEntity<TEntity>)e).Navigation).GetEnumerator();
 
    IEnumerator IEnumerable.GetEnumerator()
        => GetEnumerator();
 
    public void Add(TEntity item)
    {
        var entity = new TJoinEntity();
        ((IJoinEntity<TEntity>)entity).Navigation = item;
        ((IJoinEntity<TOtherEntity>)entity).Navigation = _ownerEntity;
        _collection.Add(entity);
    }
 
    public void Clear()
        => _collection.Clear();
 
    public bool Contains(TEntity item)
        => _collection.Any(e => Equals(item, e));
 
    // public void CopyTo(TEntity[] array, int arrayIndex)
    //     => this.ToList().CopyTo(array, arrayIndex);
    // 此处我做出了修改，修正了堆栈溢出的问题
    public void CopyTo(TEntity[] array, int arrayIndex)
        => _collection
            .Select(je => ((IJoinEntity<TEntity>)je).Navigation)
            .ToList()
            .CopyTo(array, arrayIndex);
 
    public bool Remove(TEntity item)
        => _collection.Remove(
            _collection.FirstOrDefault(e => Equals(item, e)));
 
    public int Count
        => _collection.Count;
 
    public bool IsReadOnly
        => _collection.IsReadOnly;
 
    private static bool Equals(TEntity item, TJoinEntity e)
        => Equals(((IJoinEntity<TEntity>)e).Navigation, item);
}
```
新的实现方式的最大的优点是，不再需要指定目标实体并创建中间实体，所以现在实体类的定义是这样的：
```csharp
public class Post
{
    public Post() => Tags = new JoinCollectionFacade<Tag, Post, PostTag>(this, PostsTags);
 
    public int Id { get; set; }
    public string Title { get; set; }
 
    private ICollection<PostTag> PostsTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public ICollection<Tag> Tags { get; }
}
 
public class Tag
{
    public Tag() => Posts = new JoinCollectionFacade<Post, Tag, PostTag>(this, PostsTags);
 
    public int Id { get; set; }
    public string Text { get; set; }
 
    private ICollection<PostTag> PostsTags { get; } = new List<PostTag>();
 
    [NotMapped]
    public IEnumerable<Post> Posts { get; }
}
```
之前在0x03中所写的任何查询代码都不需要改变，一切正常运行~