---
layout: post
title: Unity ECS中解决System间Job依赖的方式
categories: [ECS]
description: ECS规范中要求System必须按照指定方式组织内部Job的依赖，本篇解释这样做的理由，并介绍System间确立依赖的机制
keywords: Unity, ECS
mathjax: true
---

{:toc}

## 引言

本篇介绍Unity DOTS ECS中是如何确认System间的依赖关系，并介绍为什么要让Job和`SystemState.Dependency`之间有依赖。~~顺便讲讲如何避免同步点 todo...~~

## ECS规范的示例

以下代码取自Unity官方的ECS Sample仓库。[链接](https://sourcegraph.com/github.com/Unity-Technologies/EntityComponentSystemSamples@master/-/blob/EntitiesSamples/Assets/ExampleCode/Jobs.cs#:~:text=public%20partial-,struct,-MySystem%20%3A%20ISystem)

```c#
// A system that schedules and completes the above IJobChunk.
    public partial struct MySystem : ISystem
    {
        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            // Get an EntityCommandBuffer from
            // the BeginSimulationEntityCommandBufferSystem.
            var ecbSingleton = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>();
            var ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

            // Create the job.
            var job = new MyIJobChunk
            {
                FooHandle = state.GetComponentTypeHandle<Foo>(false),
                BarHandle = state.GetComponentTypeHandle<Bar>(true),
                Ecb = ecb.AsParallelWriter()
            };

            var myQuery = SystemAPI.QueryBuilder().WithAll<Foo, Bar, Apple>().WithNone<Banana>().Build();

            // Schedule the job.
            // By calling ScheduleParallel() instead of Schedule(),
            // the chunks matching the job's query will be split up
            // into batches, and these batches may be processed
            // in parallel by the worker threads.
            // We pass state.Dependency to ensure that this job depends upon
            // any overlapping jobs scheduled in prior system updates.
            // We assign the returned handle to state.Dependency to ensure
            // that this job is passed as a dependency to other systems.
            state.Dependency = job.ScheduleParallel(myQuery, state.Dependency);
        }
    }
}
```

这是一个标准的System示例，效果是查询具有Foo，Bar和Apple类型的Component的Chunk，并遍历它们。需要注意的是，由于这个示例中只有一个Job，所以代码写的简单了些。如果有多个Job，遵循规范，System中的多个Job都必须依赖于`state.Dependency`，并且最后需要把所有内部的Job Combine到一起，组合成新的JobHandle再设置回state.Dependency.

在创建job时，提供了其所需的`ComponentTypeHandle`，分别是Foo和Bar；以及ECB。System的`OnUpdate`最后一行，将新开Job依赖于`state.Dependency`，并且将生成的新JobHandle设置回`state.Dependency`。接下来我们介绍一下为什么要这样做。

### System对Component的使用

遵循ECS规范，在System中的函数，不要使用`EntityManager`来查询entity、访问Component，而是要借助于`ComponentTypeHandle`，在Job中通过访问chunk上指定Handle的方式来获取组件。这是因为，调用`GetComponentTypeHandle`时，其实将指定Component的类型注册到了System中，参数类型true/false则标识着对Component的读写权限，true代表只读。若是通过`EntityManager`来获取，则不会注册这份System对Component的使用。

而知道了各个System对Component的使用情况，ECS机制会将那些那些使用到相同组件的System管理起来。例如，如果System A和B对组件Foo分别有读写、只读的访问注册，那么System B的Job执行便对System A的Job执行产生了依赖，如果System B的OnUpdate执行时，还有正在执行的System A的Job，那就有必要等它完成。有点像数据库的事务关系吧！

（读写的System对只读的System是没有依赖的。）

### Job依赖于Dependency的理由

每个System实例有自己的SystemState，其作为参数传到`OnUpdate`中。state上保存了许多状态，其中包括了`Dependency`字段，在OnUpdate时，其代表着该System所依赖的Job的Handle；在其他时刻，其代表着自己的全部Job的Handle。

在System的`OnUpdate`执行之前，会经历如下几个涉及Dependency的操作：

1. 把Dependency给Complete，确保依赖的Job都执行完成了(这一过程会阻塞主线程)
2. 将所有可能有依赖关系的其他System的dependency Combine到一起，得到一个新的JobHandle，设置给Dependency
3. OnUpdate执行...
4. 到最后，将System中新调度的Job的Handle Combine到一起，得到一个新的JobHandle，设置给Dependency（这一步不是自动完成的，需要在OnUpdate的末尾，由开发者自行添加）

首先第一条确保了上次本System OnUpdate时调度的Job都执行完了。其次，第二条确保当前System要执行的所有Job，可能涉及到的依赖的其他Job的修改，都被归总在OnUpdate调用之前的Dependency中了，System内部的Job只要依赖Dependency，就很安全。

然而这个机制要达成离不开最后对Dependency的设置，把新开的Job归总，设置给Dependency，这样其他System OnUpdate前获取到的就是你自己Job的Handle。

### 其他

首先要明确一点，ECS对System的依赖管理，严格程度也只是到了共享数据是组件类型的这一粒度。如果你的多个Job中使用到了其他相同的临界资源（例如，同一个`NativeArray`），那就要自己管理了。

需要注意的是，System中的Job只需要直接/间接依赖自OnUpdate调用一开始时的Dependency就好。而最后设置新的Dependency时也只要确保新的Handle涵盖了所有Job。如果定义了Job A和B，B依赖于A，那么只需要A依赖于Dependency，然后把B的Handle设置给Dependency就好了。

把所有的Job都Combine到一起，设置给Dependency，看起来有点过了头，毕竟System可能调度了多个Job，但它们分别使用了不同的组件，而遵循规范的话，则是把它们按照System作为依赖的最小单位管理起来，可能有点浪费。但是没办法呀，在Job上传入的是`ComponentTypeHandle`，这是在System上注册的，只有System上保存了对Component的使用信息。Well，其实也还好了。

## 用不到依赖的场景

虽然ECS规范建议（要求）我们这样去实现System，但是具体情况具体分析，有些时候可以例外。例如，如果System中没有对任何Component进行访问，就没必要让System中的Job依赖state.dependency。这种情况比较少见，但是如果不依赖于组件数据，那就没必要等待其他Job的完成再执行。归根结底，System的dependency机制是为了确保**访问相同Component的不同System上Job依赖**没有问题的。
