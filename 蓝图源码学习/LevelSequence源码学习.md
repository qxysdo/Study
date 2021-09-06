本篇源码剖析主要针对UE4的LevelSequence工具，打算分4篇来讲，第一篇是Runtime不分、第二篇是Editor部分、第三篇是SequenceRecorder、第四篇是Sequencer的编辑器拓展。下面开始讲LevelSequence的Runtime部分。

## Runtime代码架构

首先，Sequence运行时相关部分代码位于UnrealEngine\Engine\Source\Runtime\MovieScene下：

![image-20210628195244258](https://xiaoyao-picture.oss-cn-shanghai.aliyuncs.com/img/image-20210628195244258.png)

除了根目录下的头文件外，可以看到整体结构组织中大致分为了Channels、Evaluation、Sections、Tracks等几个比较重要的部分，整体的代码架构如下：

```text
ALevelSequenceActor
         |____UMovieSceneSequence
         |              |___UMovieSceneTrack
         |              |          |____UMovieSceneSection
         |              |                    |____UMovieSceneChannel
         |              |___FMovieSceneEvaluationTemplate
         |                               |_________FMovieSceneExecutionTokens
         |____ULevelSequencerPlayer
                        |____UMovieSceneSequence
```

------

## ALevelSequenceActor

ALevelSequenceActor作为Sequence在运行时控制流的入口，承担整个LevelSequence的逐帧更新工作，它包含了两个关键的类，一个是UMovieSceneSequence，一个是ULevelSequencePlayer。

- **ALevelSequenceActor的继承关系**

```text
class LEVELSEQUENCE_API ALevelSequenceActor : 
    public AActor,
    ，public IMovieScenePlaybackClient,
    ，public IMovieSceneBindingOwnerInterface
```

- **ALevelSequenceActor的Tick时序**

首先 ALevelSequence是一个Actor,那么它自己肯定会在GameThread中随着逻辑线程的更新而Tick，我们可以在LevelTick.cpp中找到：

```text
for (int32 i = LevelSequenceActors.Num() - 1; i >= 0; --i)
{
        if (LevelSequenceActors[i] != nullptr)
        {
            LevelSequenceActors[i]->Tick(DeltaSeconds);
        }
}
```

可以看到LevelSequenceActor的更新是独立于AActor和UActorComponent的, 它并不属于任何TickGroup，并且它位于RuntTickGroup之前，也就是说LevelSequenceActor的更新是在当前帧任何Actor和ActorComponent以及物理世界更新之前的。

- **IMovieScenePlaybackClient**

IMovieScenePlaybackClient中定义了一个非常重要的接口：

```text
virtual bool RetrieveBindingOverrides(const FGuid& InBindingId, 
    FMovieSceneSequenceID InSequenceID, 
    TArray<UObject*, TInlineAllocator<1>>& OutObjects
) const = 0;
```

这个接口负责找到BindingID对应的BindingObject。

- **IMovieSceneBindingOwnerInterface**

```text
void SetBinding(FMovieSceneObjectBindingID Binding, const TArray<AActor*>& Actors, bool bAllowBindingsFromAsset = false);
```

来完成绑定，对于调用该代码的C++使用者来说，非常不直观，但是UE4在编辑器下为蓝图使用者做了可视化的机制，BindingID对蓝图来说是透明的，这使得Binding过程变的非常方便、直观，这个Editor功能就是IMovieSceneBindingOwnerInterface提供的。

## **UMovieSceneSequence**

UMovieSceneSequence是一个抽象类，它代表一个LevelSequence资源的实例，一般来说我们在编辑器里新建一个LevelSequence，它对应的是ULevelSequence实例。LevelSequence资源类是UMovieScene。
UMovieSceneSequence中又有两个非常重要的概念，一个是SpawnableObject一个是PossessableObject。

1. *SpawnableObject*(自动Spawn的Object)
   表示该Object在Sequence被Evaluate的时候自动Spawn，并且由Sequence来管理生命周期
2. *PossessableObject*(可以被赋予的Object)
   表示该Object可以被外部赋予，比如我手动赋予一个Actor给某个Track

既然有了PossessableObject，那自然而然的UMovieSceneSequence提供了一个BindPossessableObjects和UnbindPossessableObjects。

## **ULevelSequencePlayer**

ULevelSequencePlayer是控制LevelSequence播放的控制器，但是它的更新还是由LevelSequenceActor的Tick来控制：

```text
void ALevelSequenceActor::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);
    if (SequencePlayer)
    {
        // If the global instance data implements a transform origin interface, use its transform as an origin for this frame
        {
            UObject*                          InstanceData = GetInstanceData();
            const IMovieSceneTransformOrigin* RawInterface = Cast<IMovieSceneTransformOrigin>(InstanceData);
            const bool bHasInterface = RawInterface || (InstanceData && InstanceData->GetClass()->ImplementsInterface(UMovieSceneTransformOrigin::StaticClass()));
            if (bHasInterface)
            {
                static FSharedPersistentDataKey GlobalTransformDataKey = FGlobalTransformPersistentData::GetDataKey();
                // Retrieve the current origin
                FTransform TransformOrigin = RawInterface ? RawInterface->GetTransformOrigin() : IMovieSceneTransformOrigin::Execute_BP_GetTransformOrigin(InstanceData);
                // Assign the transform origin to the peristent data so it can be queried in Evaluate
                FPersistentEvaluationData PersistentData(*SequencePlayer);
                PersistentData.GetOrAdd<FGlobalTransformPersistentData>(GlobalTransformDataKey).Origin = TransformOrigin;
            }
        }
        SequencePlayer->Update(DeltaSeconds);
    }
}
```

ULevelSequencePlayer中有两个比较重要的成员变量：World、Level：

```text
/** The world this player will spawn actors in, if needed */
    TWeakObjectPtr<UWorld> World;
/** The world this player will spawn actors in, if needed */
    TWeakObjectPtr<ULevel> Level;
```

我们可以指定SpawnableObject被Spawn到指定的World和Sublevel里，这个非常有用。

另外，整个Sequence的模拟时间单位是FrameNumber而不是时间，所以在SequencePlayer里面有一个FMovieSceneTimeController来完成FrameTime到FrameNumber的转换和时间控制。

## **UMovieSceneTrack**

讲完了Sequence的上层架构，接下来要将Sequence的底层架构了，一个UMovieScene由若干个MasterTrack(UMovieSceneTrack)组成，UMovieScene相当于是UMovieSceneTracks的容器，那么UMovieSceneTrack又由UMovieSceneSections组成，UMovieSceneSection就是一个Track中间的某一段。

## **UMovieSceneSection**

UMovieSceneSection是一个具体Section的抽象和封装，它保存了一个Section的开始FrameNumber和结束FrameNumber，除此之外还保存了该Section对应的SequenceChannel.

```text
/**
     * Channel proxy that contains all the channels in this section - must be populated and invalidated by derived types.
     * Must be re-allocated any time any channel pointer in derived types is reallocated (such as channel data stored in arrays)
     * to ensure that any weak handles to channels are invalidated correctly. Allocation is via MakeShared<FMovieSceneChannelProxy>().
*/
    TSharedPtr<FMovieSceneChannelProxy> ChannelProxy;
```

为了支持多线程环境下的Evaluation，这里Channel用了一个SharedPtr的Proxy来封装，FMovieSceneChannel实际上就是具体的关键帧数据了，比如Transform、int32、float等等。

## **Sequence的Eval过程**

接下来是两个非常重要的东西，EvalTemplate和ExecutionTokens，这两个东西一起完成了Track的模拟。由于Sequence的模拟是一个非常耗时的过程，所以它的模拟不一定在GameThread上进行。

我们可以看一下运行时的调用栈：

## **UMovieSceneSequence**

UMovieSceneSequence是一个抽象类，它代表一个LevelSequence资源的实例，一般来说我们在编辑器里新建一个LevelSequence，它对应的是ULevelSequence实例。LevelSequence资源类是UMovieScene。
UMovieSceneSequence中又有两个非常重要的概念，一个是SpawnableObject一个是PossessableObject。

1. *SpawnableObject*(自动Spawn的Object)
   表示该Object在Sequence被Evaluate的时候自动Spawn，并且由Sequence来管理生命周期
2. *PossessableObject*(可以被赋予的Object)
   表示该Object可以被外部赋予，比如我手动赋予一个Actor给某个Track

既然有了PossessableObject，那自然而然的UMovieSceneSequence提供了一个BindPossessableObjects和UnbindPossessableObjects。

## **ULevelSequencePlayer**

ULevelSequencePlayer是控制LevelSequence播放的控制器，但是它的更新还是由LevelSequenceActor的Tick来控制：

```text
void ALevelSequenceActor::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);
    if (SequencePlayer)
    {
        // If the global instance data implements a transform origin interface, use its transform as an origin for this frame
        {
            UObject*                          InstanceData = GetInstanceData();
            const IMovieSceneTransformOrigin* RawInterface = Cast<IMovieSceneTransformOrigin>(InstanceData);
            const bool bHasInterface = RawInterface || (InstanceData && InstanceData->GetClass()->ImplementsInterface(UMovieSceneTransformOrigin::StaticClass()));
            if (bHasInterface)
            {
                static FSharedPersistentDataKey GlobalTransformDataKey = FGlobalTransformPersistentData::GetDataKey();
                // Retrieve the current origin
                FTransform TransformOrigin = RawInterface ? RawInterface->GetTransformOrigin() : IMovieSceneTransformOrigin::Execute_BP_GetTransformOrigin(InstanceData);
                // Assign the transform origin to the peristent data so it can be queried in Evaluate
                FPersistentEvaluationData PersistentData(*SequencePlayer);
                PersistentData.GetOrAdd<FGlobalTransformPersistentData>(GlobalTransformDataKey).Origin = TransformOrigin;
            }
        }
        SequencePlayer->Update(DeltaSeconds);
    }
}
```

ULevelSequencePlayer中有两个比较重要的成员变量：World、Level：

```text
/** The world this player will spawn actors in, if needed */
    TWeakObjectPtr<UWorld> World;
/** The world this player will spawn actors in, if needed */
    TWeakObjectPtr<ULevel> Level;
```

我们可以指定SpawnableObject被Spawn到指定的World和Sublevel里，这个非常有用。

另外，整个Sequence的模拟时间单位是FrameNumber而不是时间，所以在SequencePlayer里面有一个FMovieSceneTimeController来完成FrameTime到FrameNumber的转换和时间控制。

## **UMovieSceneTrack**

讲完了Sequence的上层架构，接下来要将Sequence的底层架构了，一个UMovieScene由若干个MasterTrack(UMovieSceneTrack)组成，UMovieScene相当于是UMovieSceneTracks的容器，那么UMovieSceneTrack又由UMovieSceneSections组成，UMovieSceneSection就是一个Track中间的某一段。

## **UMovieSceneSection**

UMovieSceneSection是一个具体Section的抽象和封装，它保存了一个Section的开始FrameNumber和结束FrameNumber，除此之外还保存了该Section对应的SequenceChannel.

```text
/**
     * Channel proxy that contains all the channels in this section - must be populated and invalidated by derived types.
     * Must be re-allocated any time any channel pointer in derived types is reallocated (such as channel data stored in arrays)
     * to ensure that any weak handles to channels are invalidated correctly. Allocation is via MakeShared<FMovieSceneChannelProxy>().
*/
    TSharedPtr<FMovieSceneChannelProxy> ChannelProxy;
```

为了支持多线程环境下的Evaluation，这里Channel用了一个SharedPtr的Proxy来封装，FMovieSceneChannel实际上就是具体的关键帧数据了，比如Transform、int32、float等等。

## **Sequence的Eval过程**

接下来是两个非常重要的东西，EvalTemplate和ExecutionTokens，这两个东西一起完成了Track的模拟。由于Sequence的模拟是一个非常耗时的过程，所以它的模拟不一定在GameThread上进行。

我们可以看一下运行时的调用栈：