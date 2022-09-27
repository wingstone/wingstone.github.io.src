---
title: 'multithread rendering in unreal'
author: wingstone
date: 2022-09-26
categories:
- contents
metaAlignment: center
coverMeta: out
---

记录unreal引擎中的多线程渲染框架下的同步问题

<!--more-->

## Game Thread向Rendering Thread传递指令

通过RenderingThread.h中的`ENQUEUE_RENDER_COMMAND`宏来进行指令传递；

```c++
// RenderingThread.h
#define ENQUEUE_RENDER_COMMAND(Type) \
	struct Type##Name \
	{  \
		static const char* CStr() { return #Type; } \
		static const TCHAR* TStr() { return TEXT(#Type); } \
	}; \
	EnqueueUniqueRenderCommand<Type##Name>
```

顺着宏指令接着展开

```c++
// RenderingThread.h
template<typename TSTR, typename LAMBDA>
FORCEINLINE_DEBUGGABLE void EnqueueUniqueRenderCommand(LAMBDA&& Lambda)
{
	//QUICK_SCOPE_CYCLE_COUNTER(STAT_EnqueueUniqueRenderCommand);
	typedef TEnqueueUniqueRenderCommandType<TSTR, LAMBDA> EURCType;

#if 0 // UE_SERVER && UE_BUILD_DEBUG
	UE_LOG(LogRHI, Warning, TEXT("Render command '%s' is being executed on a dedicated server."), TSTR::TStr())
#endif
	if (IsInRenderingThread())
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
	else
	{
		if (ShouldExecuteOnRenderThread())  //这里为在GameThread中真正执行的指令
		{
			CheckNotBlockedOnRenderThread();
			TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));
		}
		else
		{
			EURCType TempCommand(Forward<LAMBDA>(Lambda));
			FScopeCycleCounter EURCMacro_Scope(TempCommand.GetStatId());
			TempCommand.DoTask(ENamedThreads::GameThread, FGraphEventRef());
		}
	}
}

```

语句`TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));`负责task任务的分发，并且将任务分发到rendering thread上面；为什么该语句可以分发到rendering thread上，可以继续跟进；

跟进后发现`ConstructAndDispatchWhenReady`的实现如下：

```c++
// TaskGraphInterfaces.h
template<typename...T>
FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args)
{
    new ((void *)&Owner->TaskStorage) TTask(Forward<T>(Args)...);
    return Owner->Setup(Prerequisites, CurrentThreadIfKnown);
}
```

其中`Setup`的实现如下：

```c++
//  TaskGraphInterfaces.h
FGraphEventRef Setup(const FGraphEventArray* Prerequisites = NULL, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
{
    TaskTrace::Launched(GetTraceId(), nullptr, Subsequents.IsValid(), ((TTask*)&TaskStorage)->GetDesiredThread());

    FGraphEventRef ReturnedEventRef = Subsequents; // very important so that this doesn't get destroyed before we return
    SetupPrereqs(Prerequisites, CurrentThreadIfKnown, true);
    return ReturnedEventRef;
}
```

其中的`((TTask*)&TaskStorage)->GetDesiredThread()`决定了要分发的thread类型；而我们要分发的任务为`EURCType`，即`TEnqueueUniqueRenderCommandType<TSTR, LAMBDA>`，能查看该类继承自`FRenderCommand`类，而该类中包含有以下函数：

```c++
// RenderingThread.h
static ENamedThreads::Type GetDesiredThread()
{
    check(!GIsThreadedRendering || ENamedThreads::GetRenderThread() != ENamedThreads::GameThread);
    return ENamedThreads::GetRenderThread();
}
```

到此，`ENamedThreads::GetRenderThread()`便指示出了我们最终要分发的线程为rendering thread；

## Game Thread与Rendering Thread同步

直接参考前人的总结[UE4主线程与渲染线程同步](https://zhuanlan.zhihu.com/p/80676205)；同步位置在Tick函数的以下部分：

```c++
// LaunchEngineLoop.cpp
{
    SCOPE_CYCLE_COUNTER(STAT_FrameSyncTime);
    // this could be perhaps moved down to get greater parallelism
    // Sync game and render thread. Either total sync or allowing one frame lag.
    static FFrameEndSync FrameEndSync;
    static auto CVarAllowOneFrameThreadLag = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.OneFrameThreadLag"));
    FrameEndSync.Sync( CVarAllowOneFrameThreadLag->GetValueOnGameThread() != 0 );
}
```

其中Sync函数用来完成同步，并且允许间隔一帧；Sync函数中同步部分的代码为：

```c++
// UnrealEngine.cpp
void FFrameEndSync::Sync( bool bAllowOneFrameThreadLag )
{
    // blablabla...

	// Since this is the frame end sync, allow sync with the RHI and GPU (true).
	Fence[EventIndex].BeginFence(true);

    // blablabla...

	// Use two events if we allow a one frame lag.
	if( bAllowOneFrameThreadLag )
	{
		EventIndex = (EventIndex + 1) % 2;
	}

    // blablabla...

	Fence[EventIndex].Wait(bEmptyGameThreadTasks);  // here we also opportunistically execute game thread tasks while we wait
}

```

BeginFence的代码如下：

```c++
// RenderingThread.cpp
void FRenderCommandFence::BeginFence(bool bSyncToRHIAndGPU)
{
    // blablabla...

    // Sync Game Thread with Render Thread only
    DECLARE_CYCLE_STAT(TEXT("FNullGraphTask.FenceRenderCommand"),
    STAT_FNullGraphTask_FenceRenderCommand,
        STATGROUP_TaskGraphTasks);

    CompletionEvent = TGraphTask<FNullGraphTask>::CreateTask(NULL, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(
        GET_STATID(STAT_FNullGraphTask_FenceRenderCommand), ENamedThreads::GetRenderThread());
}
```

可以看到，BeginFence会创建一个FNullGraphTask任务添加到RenderThread中；

而Sync函数中的Wait函数代码如下：

```c++
// RenderingThread.cpp
/**
 * Waits for pending fence commands to retire.
 */
void FRenderCommandFence::Wait(bool bProcessGameThreadTasks) const
{
	if (!IsFenceComplete())
	{
		StopRenderCommandFenceBundler();
#if 0
		// on most platforms this is a better solution because it doesn't spin
		// windows needs to pump messages
		if (bProcessGameThreadTasks)
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FRenderCommandFence_Wait);
			FTaskGraphInterface::Get().WaitUntilTaskCompletes(CompletionEvent, ENamedThreads::GameThread);
		}
#endif
		GameThreadWaitForTask(CompletionEvent, TriggerThreadIndex, bProcessGameThreadTasks);
	}
}

```

可以看到GameThread会通过GameThreadWaitForTask函数来等待之前（隔帧或者当前帧）创建的FNullGraphTask任务完成，若FNullGraphTask任务完成了，则之前RenderingThread队列中的任务也都完成了；也就完成了Game Thread与Rendering Thread的同步；

## Rendering Thread向RHI Thread传递指令

在RHICommandList.h中封装了大量可以在RenderingThread中可以调用的api，以其中的以下代码为例：

```c++
// RHICommandList.h
FORCEINLINE_DEBUGGABLE void SetShaderTexture(FRHIGraphicsShader* Shader, uint32 TextureIndex, FRHITexture* Texture)
{
    //check(IsOutsideRenderPass());
    ValidateBoundShader(Shader);
    if (Bypass())
    {
        GetContext().RHISetShaderTexture(Shader, TextureIndex, Texture);
        return;
    }
    ALLOC_COMMAND(FRHICommandSetShaderTexture<FRHIGraphicsShader>)(Shader, TextureIndex, Texture);
}
```

该函数在SceneRendering.cpp中的`AddResolveSceneDepthPass`函数的lambda参数里面进行了调用；其中的`ALLOC_COMMAND`宏复杂向RHI Thread传递指令；继续跟进可得：

```c++
// RHICommandList.h
#define ALLOC_COMMAND(...) new ( AllocCommand(sizeof(__VA_ARGS__), alignof(__VA_ARGS__)) ) __VA_ARGS__
```

展开即为：
```c++
// RHICommandList.h
// 展开前
ALLOC_COMMAND(FRHICommandSetShaderTexture<FRHIGraphicsShader>)(Shader, TextureIndex, Texture);
// 展开后
new (AllocCommand(sizeof(FRHICommandSetShaderTexture<FRHIGraphicsShader>), alignof(FRHICommandSetShaderTexture<FRHIGraphicsShader>))) FRHICommandSetShaderTexture<FRHIGraphicsShader>(Shader, TextureIndex, Texture);
```

这里用到了new关键字的另外一种用法，在内存的`AllocCommand(sizeof(FRHICommandSetShaderTexture<FRHIGraphicsShader>), alignof(FRHICommandSetShaderTexture<FRHIGraphicsShader>)))`位置，使用`FRHICommandSetShaderTexture<FRHIGraphicsShader>(Shader, TextureIndex, Texture)`来进行初始化；跟进AllocCommand可得：

```c++
// RHICommandList.h
FORCEINLINE_DEBUGGABLE void* AllocCommand(int32 AllocSize, int32 Alignment)
{
    checkSlow(!IsExecuting());
    FRHICommandBase* Result = (FRHICommandBase*) MemManager.Alloc(AllocSize, Alignment);
    ++NumCommands;
    *CommandLink = Result;
    CommandLink = &Result->Next;
    return Result;
}
```

可以看到RHICommand会以链表的形式存储起来，CommandLink的根节点在`RHICommandList`类下Root变量中；到目前为止，所有的RHICommand指令还未向RHI thread分发；跟进Root节点，最终能发现指令的分发在如下代码中：

```c++
// RHICommandList.cpp
void FRHICommandListExecutor::ExecuteInner(FRHICommandListBase& CmdList)
{
    // blablabla...

    RHIThreadTask = TGraphTask<FExecuteRHIThreadTask>::CreateTask(&Prereq, ENamedThreads::GetRenderThread()).ConstructAndDispatchWhenReady(SwapCmdList);

    // blablabla...    
}
```

跟进堆栈，能发现该函数在`ImmediateFlush`函数中调用，即

```c++
// RHICommandList.inc
FORCEINLINE_DEBUGGABLE void FRHICommandListImmediate::ImmediateFlush(EImmediateFlushType::Type FlushType)
{
	switch (FlushType)
	{
	case EImmediateFlushType::WaitForOutstandingTasksOnly:
		{
			WaitForTasks();
		}
		break;

	case EImmediateFlushType::DispatchToRHIThread:
		{
			if (HasCommands())
			{
				GRHICommandList.ExecuteList(*this);
			}
		}
		break;

	case EImmediateFlushType::WaitForDispatchToRHIThread:
		{
			if (HasCommands())
			{
				GRHICommandList.ExecuteList(*this);
			}
			WaitForDispatch();
		}
		break;

	case EImmediateFlushType::FlushRHIThread:
		{
			CSV_SCOPED_TIMING_STAT(RHITFlushes, FlushRHIThreadTotal);
			if (HasCommands())
			{
				GRHICommandList.ExecuteList(*this);
			}
			WaitForDispatch();
			if (IsRunningRHIInSeparateThread())
			{
				WaitForRHIThreadTasks();
			}
			WaitForTasks(true); // these are already done, but this resets the outstanding array
		}
		break;

	case EImmediateFlushType::FlushRHIThreadFlushResources:
		{
			CSV_SCOPED_TIMING_STAT(RHITFlushes, FlushRHIThreadFlushResourcesTotal);
			if (HasCommands())
			{
				GRHICommandList.ExecuteList(*this);
			}
			WaitForDispatch();
			WaitForRHIThreadTasks();
			WaitForTasks(true); // these are already done, but this resets the outstanding array

			PipelineStateCache::FlushResources();
			FRHIResource::FlushPendingDeletes(FRHICommandListExecutor::GetImmediateCommandList());
		}
		break;

	default:
		check(0);
	}
}
```

`ImmediateFlush`在RenderingThread线程的多处都有调用，至此就完成了RHICommand指令的分发；

## Game Thread与RHI Thread同步

unreal中Game Thread、Rendering Thread以及RHI Thread之间的同步策略是可选的，在RenderingThread.cpp文件中有以下代码

```c++
// RenderingThread.cpp
TAutoConsoleVariable<int32> CVarGTSyncType(
	TEXT("r.GTSyncType"),
	0,
	TEXT("Determines how the game thread syncs with the render thread, RHI thread and GPU.\n")
	TEXT("Syncing to the GPU swap chain flip allows for lower frame latency.\n")
	TEXT(" 0 - Sync the game thread with the render thread (default).\n")
	TEXT(" 1 - Sync the game thread with the RHI thread.\n")
	TEXT(" 2 - Sync the game thread with the GPU swap chain flip (only on supported platforms).\n"),
	ECVF_Default);
```

`CVarGTSyncType`控制值各线程之间的同步策略；Game Thread与Rendering Thread同步时会在`void FRenderCommandFence::BeginFence(bool bSyncToRHIAndGPU)`函数中向RenderingThread线程插入空任务，而Game Thread与RHI Thread同步时，同样在该函数中会执行以下代码：

```c++
// RenderingThread.cpp
void FRenderCommandFence::BeginFence(bool bSyncToRHIAndGPU)
{
    // blablabla...
    if (bSyncToRHIAndGPU)
    {
        if (IsRHIThreadRunning())
        {
            // Change trigger thread to RHI
            TriggerThreadIndex = ENamedThreads::RHIThread;
        }
        
        // Create a task graph event which we can pass to the render or RHI threads.
        CompletionEvent = FGraphEvent::CreateGraphEvent();

        FGraphEventRef InCompletionEvent = CompletionEvent;
        ENQUEUE_RENDER_COMMAND(FSyncFrameCommand)(
            [InCompletionEvent, GTSyncType](FRHICommandListImmediate& RHICmdList)
            {
                if (IsRHIThreadRunning())
                {
                    ALLOC_COMMAND_CL(RHICmdList, FRHISyncFrameCommand)(InCompletionEvent, GTSyncType);
                    RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
                }
                else
                {
                    FRHISyncFrameCommand Command(InCompletionEvent, GTSyncType);
                    Command.Execute(RHICmdList);
                }
            });
    }

    // blablabla...
}
```

可见，Game Thread与RHI Thread同步时，会在RHI Thread线程中插入`FSyncFrameCommand`指令，且该指令被GameThread记录，用于在Game Thread线程中进行同步，代码如下：

```c++
// UnrealEngine.cpp
void FFrameEndSync::Sync( bool bAllowOneFrameThreadLag )
{
    // blablabla...	
    
    // Since this is the frame end sync, allow sync with the RHI and GPU (true).
	Fence[EventIndex].BeginFence(true);

    // blablabla...	

	Fence[EventIndex].Wait(bEmptyGameThreadTasks);  // here we also opportunistically execute game thread tasks while we wait
 
    // blablabla...
}
```

至此，就完成了Game Thread与RHI Thread之间的同步；