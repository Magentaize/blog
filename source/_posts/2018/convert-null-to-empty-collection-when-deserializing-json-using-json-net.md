---
title: Convert Null to Empty Collection When Deserializing Json Using Json.Net
date: 2018-07-02 15:05:47
categories:
 - Tech
---

## TL;DR
Insert a custom `JsonConverter` into `JsonSerializerSettings`, when reading json string stream, this converter will split JArray into JTokens, so, when JTokens end with "JsonToken.EndArray", we can handle the property value which will be injected to the corresponding instance.
<!--more-->

## Get Start
Json.Net uses a assembly line as json processor, the assembly line contains a series of aspects which be used to bind json field to type field, create type of field, select constructor of type, create parameters of selected constructor, convert original value to target value, call deserialize interceptor and etc. All those aspects are connected to a special object called "Contract", if we dig into it, it can be easily find out that all contracts are built by a contract factory called "ContractResolver". By default, Json.Net provides some built-in resolvers like DefaultContractResolver and DynamicContractResolver. 

Do not underestimate ContractResolver even though its function is just supplying a contract(strategy) for each type, constructor, token and so on. Due to the reason ContractResolver can control everything during serializeing and deserializeing, honstly saying, it's the second complex part in json processor.

## Try to Touch ContractResolver
Although ContractResolver is so complex, the easiest thing is to inherit it. We can create a custom contract resolver to override some methods so we can voyeur what happens inside.

```csharp
public class CustomContractResolver : DefaultContractResolver
{
    protected override IValueProvider CreateMemberValueProvider(MemberInfo member)
    {
        var ret = base.CreateMemberValueProvider(member);

        return ret;
    }
}
```

If we set a breakpoint at line `var ret`, we can get its inner properties in watch window. By this method, other content of contracts can be easily clarified.

## How to Replace Null ?
If you have read here, maybe you want to have a try to replace null in your custom contract resolver, but, please wait for a second. If you do that, you'll have no idea about how to deal with it.

Before we start work, let's thinks about how to do correct things. There's a mistake needed be rectified, that is, ContractResolve not controls object instance, it just organizes.

So, why organizing? As we known, when deserializing, json will be spilted into a stream of token, the benefit of it is we can deserialize an instance only reading json once but the outer instance will not be created until its children instances have been created, however, to create child instance, we need other converter. That means, the progress of deserializing is recursive not linear and the signal of end deserializing is the progress of assembly line backtracks to the outermost object.

You maybe notice I have mentioned converter for many times, so, what is converter?
Converter is JsonConverter, the most complex and important part in json processor. After json reader spilts json into pieces of token, converters are responsible for converting tokens to object instances. Therefor, we can use converter to convert null to empty collection.

## Create a JsonConverter
A JsonConverter are composed by three methods: WriteJson, ReadJson and CanConvert. Because we don't serialize now, we temporarily not involved in WriteJson. 

CanConvert is the first method will be invoked, it will tell to ContractResolver whether it can handle this type or not. For our purpose, we want it only handles Collection type. So, it should be that:

```csharp
public override bool CanConvert(Type objectType)
{
    var ret = false;
    if (objectType.IsGenericType)
    {
        var ifs = objectType.GetTypeInfo().ImplementedInterfaces.Where(i => i.IsGenericType).ToArray();
        var b = ifs.Any(i => i.GetGenericTypeDefinition() == typeof(ICollection<>)) &&
        ifs.All(i => i.GetGenericTypeDefinition() != typeof(IDictionary<,>));
        if (b)
            ret = true;
    }

    return ret;
}
```

Be a attention to Dictionary also implements ICollection interface, we must filter it.

Once the converter tells it can handle this type, when json reader faces this type, reader will redirect json token stream to it. This means, ReadJson will be invoked.
And it should be that:

```csharp
public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
{
    var ret = Activator.CreateInstance(objectType);
    var addMethod = ret.GetType().GetMethod("Add");

    if (reader.TokenType == JsonToken.StartArray)
    {
        reader.Read();
        while (reader.TokenType != JsonToken.EndArray)
        {
            var item = serializer.Deserialize(reader, objectType.GetGenericArguments()[0]);
            reader.Read();
            addMethod.Invoke(ret, new[] {item});
        }
    }

    return ret;
}
```

First, I use Activator.CreateInstance to create an instance from target type, and use reflection to get "Collection&lt;T&gt;.Add()" method because the type of it is uncertain when compiling and you can use dynamic object instead.
    
Next, we judge whether the current token is JsonToken.StartArray or not, if true, move json reader to next token and deserialize it whole children objects, if false, directly return the collection instance.

Now, the whole work is down, and you will say: hey, wake up, where is null?

Yep, there is no null explicitly, because if the first token isn't JsonToken.StartArray, it will be JsonToken.Null, and we directly return an empty collection instead of default null.
