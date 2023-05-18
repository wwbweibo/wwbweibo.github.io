---
layout: post
title:  "k8s调度器源码简单阅读"
date:   2021-06-07 23:33:27 +0800
categories: kubernetes
author: wwbweibo
math: true
mermaid: true
---
# k8s调度器源码简单阅读

*基于K8s v1.20.2版本源码*

调度器的代码位于`pkg/scheduler`下

## scheduler.go

 `scheduler.go` 中是调度器的实现源码，其中定义了调度器`Scheduler`结构体，代码如下
 
``` go
// Scheduler watches for new unscheduled pods. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
// Scheduler 监控新的未调度的pod，并为他们找到一个合适的节点并协会Api Server
type Scheduler struct {
	// It is expected that changes made via SchedulerCache will be observed
	// by NodeLister and Algorithm.
	SchedulerCache internalcache.Cache

	Algorithm core.ScheduleAlgorithm

	// NextPod should be a function that blocks until the next pod
	// is available. We don't use a channel for this, because scheduling
	// a pod may take some amount of time and we don't want pods to get
	// stale while they sit in a channel.
	NextPod func() *framework.QueuedPodInfo

	// Error is called if there is an error. It is passed the pod in
	// question, and the error
	Error func(*framework.QueuedPodInfo, error)

	// Close this to shut down the scheduler.
	StopEverything <-chan struct{}

	// SchedulingQueue holds pods to be scheduled
	SchedulingQueue internalqueue.SchedulingQueue

	// Profiles are the scheduling profiles.
	Profiles profile.Map

	client clientset.Interface
}
```
其中`SchedulerCache`中保存的是集群中的相关缓存数据，用于提升新能。`Algorithm`是具体的调度算法实现。`NextPod`是一个函数，返回待调度队列中的下一个pod。`SchedulingQueue`为调度队列。  
调度器的初始化很简单。  

### 调度去初始化

首先，初始化了一个集群状态缓存`schedulerCache`，之后初始化了一个可用的插件集合`runtime.Registry`,并合并给定配置中的其他插件

``` go
schedulerCache := internalcache.New(30*time.Second, stopEverything)

registry := frameworkplugins.NewInTreeRegistry()
if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
    return nil, err
}
```

在此之后会注册metrics，初始化调度算法，添加事件处理器

``` go
source := options.schedulerAlgorithmSource
switch {
// Provider是具体的调度算法提供者，如果不为空，根据该值初始化调度器，
// Provider必须是已经注册的
case source.Provider != nil:
    // Create the config from a named algorithm provider.
    sc, err := configurator.createFromProvider(*source.Provider)
    if err != nil {
        return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
    }
    sched = sc
// Policy是基于策略的调度算法
case source.Policy != nil:
    // Create the config from a user specified policy source.
    policy := &schedulerapi.Policy{}
    switch {
    case source.Policy.File != nil:
        if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
            return nil, err
        }
    case source.Policy.ConfigMap != nil:
        if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
            return nil, err
        }
    }
    // Set extenders on the configurator now that we've decoded the policy
    // In this case, c.extenders should be nil since we're using a policy (and therefore not componentconfig,
    // which would have set extenders in the above instantiation of Configurator from CC options)
    configurator.extenders = policy.Extenders
    sc, err := configurator.createFromConfig(*policy)
    if err != nil {
        return nil, fmt.Errorf("couldn't create scheduler from policy: %v", err)
    }
    sched = sc
default:
    return nil, fmt.Errorf("unsupported algorithm source: %v", source)
}
// Additional tweaks to the config produced by the configurator.
sched.StopEverything = stopEverything
sched.client = client

addAllEventHandlers(sched, informerFactory)
```

### 调度流程

调度流程的代码在方法 `scheduleOne` 中

``` go
// scheduleOne does the entire scheduling workflow for a single pod. It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne(ctx context.Context) { 
```

方法中首先会调用调度器的`NextPod`方法获取下一个等待调度的POD，该方法是一个阻塞方法。

```go
podInfo := sched.NextPod()
// pod could be nil when schedulerQueue is closed
if podInfo == nil || podInfo.Pod == nil {
    return
}
pod := podInfo.Pod
// 获取POD应该使用的插件，该方法返回一个FrameWork实例，FrameWork是用于管理调度框架中的插件集合
fwk, err := sched.frameworkForPod(pod)
if err != nil {
    // This shouldn't happen, because we only accept for scheduling the pods
    // which specify a scheduler name that matches one of the profiles.
    klog.Error(err)
    return
}
if sched.skipPodSchedule(fwk, pod) {
    return
}
```

之后会尝试为该POD寻找一个合适的节点（同步进行），具体逻辑是调用调度算法的`Schedule`方法实现的，该方法返回一个调度结果和error。当调度失败
时会尝试进入POD的驱逐逻辑。

当返回的err为FitError时且该POD定义了至少一个PostFilter时会调用PostFilter进行POD驱逐

``` go
// state是一个framewwork.CycleState的对象，为插件提供了一种存储和检索任意数据的机制
scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, fwk, state, pod)
// 以下是对于调度失败的处理。
if err != nil {
    // Schedule() may have failed because the pod would not fit on any host, so we try to
    // preempt, with the expectation that the next time the pod is tried for scheduling it
    // will fit due to the preemption. It is also possible that a different pod will schedule
    // into the resources that were preempted, but this is harmless.
    nominatedNode := ""
    if fitError, ok := err.(*core.FitError); ok {
        if !fwk.HasPostFilterPlugins() {
            klog.V(3).Infof("No PostFilter plugins are registered, so no preemption will be performed.")
        } else {
            // Run PostFilter plugins to try to make the pod schedulable in a future scheduling cycle.
            result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.FilteredNodesStatuses)
            if status.Code() == framework.Error {
                klog.Errorf("Status after running PostFilter plugins for pod %v/%v: %v", pod.Namespace, pod.Name, status)
            } else {
                klog.V(5).Infof("Status after running PostFilter plugins for pod %v/%v: %v", pod.Namespace, pod.Name, status)
            }
            if status.IsSuccess() && result != nil {
                nominatedNode = result.NominatedNodeName
            }
        }
        // Pod did not fit anywhere, so it is counted as a failure. If preemption
        // succeeds, the pod should get counted as a success the next time we try to
        // schedule it. (hopefully)
        metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
    } else if err == core.ErrNoNodesAvailable {
        // No nodes available is counted as unschedulable rather than an error.
        metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
    } else {
        klog.ErrorS(err, "Error selecting node for pod", "pod", klog.KObj(pod))
        metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
    }
    sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
    return
}
```

之后会进入节点的绑定过程（异步），在此之前，首先会调用`assume`方法告知缓存该POD已经在对应节点上运行，然后调用Reverse插件（我还不知道是用来干啥的）。
接着调用Permit插件（也不知道干啥的），最后开启新的GoRoutine进行绑定。

```go
// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
go func() {
    bindingCycleCtx, cancel := context.WithCancel(ctx)
    defer cancel()
    metrics.SchedulerGoroutines.WithLabelValues("binding").Inc()
    defer metrics.SchedulerGoroutines.WithLabelValues("binding").Dec()

    waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
    if !waitOnPermitStatus.IsSuccess() {
        var reason string
        if waitOnPermitStatus.IsUnschedulable() {
            metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
            reason = v1.PodReasonUnschedulable
        } else {
            metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
            reason = SchedulerError
        }
        // trigger un-reserve plugins to clean up state associated with the reserved Pod
        fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
        if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
            klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
        }
        sched.recordSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, "")
        return
    }

    // Run "prebind" plugins.
    preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
    if !preBindStatus.IsSuccess() {
        metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
        // trigger un-reserve plugins to clean up state associated with the reserved Pod
        fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
        if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
            klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
        }
        sched.recordSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, "")
        return
    }

    err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
    if err != nil {
        metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
        // trigger un-reserve plugins to clean up state associated with the reserved Pod
        fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
        if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
            klog.Errorf("scheduler cache ForgetPod failed: %v", err)
        }
        sched.recordSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, "")
    } else {
        // Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
        if klog.V(2).Enabled() {
            klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
        }
        metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
        metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
        metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

        // Run "postbind" plugins.
        fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
    }
}()
```