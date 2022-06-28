# k8s-scheduler 抢占

## ![avatar](./images/%E6%80%BB%E4%BD%93%E6%B5%81%E7%A8%8B.png)



## 抢占触发条件

1. 执行完节点过滤操作后（Filter 扩展点）没有和合适节点
2. 抢占功能打开 （默认开启)

```go
scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
	if err != nil {
		if fitError, ok := err.(*core.FitError); ok {
            // 是否关闭了抢占功能
			if sched.DisablePreemption {
				klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
			} else {
				preemptionStartTime := time.Now()
				sched.preempt(schedulingCycleCtx, prof, state, pod, fitError)
				metrics.PreemptionAttempts.Inc()
				metrics.SchedulingAlgorithmPreemptionEvaluationDuration.Observe(metrics.SinceInSeconds(preemptionStartTime))
				metrics.DeprecatedSchedulingDuration.WithLabelValues(metrics.PreemptionEvaluation).Observe(metrics.SinceInSeconds(preemptionStartTime))
			}
        }
        // 返回错误，重新入队
        sched.recordSchedulingFailure(prof, podInfo.DeepCopy(), err, v1.PodReasonUnschedulable, err.Error()) 
			...
		return
	}

// 当执行完过滤操作后没有合适的节点，返回 fitError
func (g *genericScheduler) Schedule(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	...
	preFilterStatus := prof.RunPreFilterPlugins(ctx, state, pod)
	if !preFilterStatus.IsSuccess() {
		return result, preFilterStatus.AsError()
	}
	trace.Step("Running prefilter plugins done")

	startPredicateEvalTime := time.Now()
	filteredNodes, filteredNodesStatuses, err := g.findNodesThatFitPod(ctx, prof, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")

	if len(filteredNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           g.nodeInfoSnapshot.NumNodes(),
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}
    ...
}
```

## 抢占逻辑

1. 从 apiServer 中拿到抢占 pod 的信息
2. 调用 Preempt 进行节点选取
    - pod 为非抢占模式 || pod.nominatedNodeName 被设置，且该节点上有一个低优先级处于删除状态（表示正在被驱逐）退出
    - 调用 nodesWherePreemptionMightHelp 找到潜在 nodes(抢占 pod 可能被调度到的节点)
        - 因资源存在压力无法被调用
        - 因端口压力无法被调用
        - 因为 pod 亲和性无法被调用
    - 调用 selectNodesForPreemption 找到所有适合抢占 pod 调度的节点
        - 对每个潜在 nodes 做一下操作
            - 删除所有优先级低于抢占 pod 的 pods,若仍然无法调度，过滤掉该 node
            - 将低于抢占 Pod 优先级的 pods 按照是否破环 pdb 规则分为 violatingVictims（破坏 pdb规则） 和 unviolatingVictims(没有破环 pdb 规则) pod队列（队列里面的 pod 按照优先级从高到低排序，相同优先级，创建时间最早的在前面）
            - 优先操作 violatingVictims 队列里面的 pod,尝试将其加回到 node,看是否会影响抢占 pod 的调度，若影响，将其加入到 victims 队列中。unviolatingVictims做同样操作。
            - 返回 victims 队列，和 numViolatingVictim（需要驱逐的破坏 pdb 规则的 pod 的总数）
    - 调用 pickOneNodeForPreemption 找寻最合适的 node 节点
        - 无需驱逐任何 pod 即可满足抢占调度（可以在驱逐期间满足了调度条件）
        - 需要驱逐 破坏 pdb 规则 pods 的数量最少
        - 若含有 破坏了 pdb 规则的 pods,每个节点选取各自准备驱逐的破环 pdb 规则的 pods 中最高优先级的 pod 进行比较，选取优先级较低的 node。若没有则选取各自节点准备驱逐的 pods 的最高优先级进行比较。（一般情况下就是比较各个 node节点上 需要驱逐的 pods 中最高优先级的 pod,选取优先级最低点（若要驱逐破环了 pdb 规则的 pods,比较这些 pods 的最高优先级））
        - 需要驱逐的 pod 优先级总和最少的 node
        - 需要驱逐的 pod 数量最少
        - 比较各个节点上最高优先级 pod 的创建时间，时间越早越被选取
        - 仍存在多个node,返回第一个 node
    - 调用 getLowerPriorityNominatedPods 获取所有优先级低于抢占 pod 的 pods
3. 为抢占 pod 设置 nominatedNodeName 为选取的抢占节点，跟新到 apiServer (跟新后，scheduler pod.informer 会收到改事件，并将该 pod 入队到 activeQ,进行调度)
4. 删除所有被驱逐的 pod
5. 清空低优先级 pods 的 nominatedNodeName

```go
func (sched *Scheduler) preempt(ctx context.Context, prof *profile.Profile, state *framework.CycleState, preemptor *v1.Pod, scheduleErr error) (string, error) {
    // 从 apiServer 中拿到抢占 pod 信息
	preemptor, err := sched.podPreemptor.getUpdatedPod(preemptor)
	if err != nil {
		klog.Errorf("Error getting the updated preemptor pod object: %v", err)
		return "", err
	}

    // 找到调度节点，驱逐的 pods,移除 nominatedNodeName 的pod
	node, victims, nominatedPodsToClear, err := sched.Algorithm.Preempt(ctx, prof, state, preemptor, scheduleErr)
	if err != nil {
		klog.Errorf("Error preempting victims to make room for %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
		return "", err
	}
	var nodeName = ""
	if node != nil {
		nodeName = node.Name
		// 在内存中设置 
		sched.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

		// 设置 pod.NominatedNodeName 并更新到 apiServer
		err = sched.podPreemptor.setNominatedNodeName(preemptor, nodeName)
		if err != nil {
			klog.Errorf("Error in preemption process. Cannot set 'NominatedPod' on pod %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
			sched.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
			return "", err
		}


		for _, victim := range victims {
            // 删除 pod
			if err := sched.podPreemptor.deletePod(victim); err != nil {
				klog.Errorf("Error preempting pod %v/%v: %v", victim.Namespace, victim.Name, err)
				return "", err
			}
			if waitingPod := prof.GetWaitingPod(victim.UID); waitingPod != nil {
				waitingPod.Reject("preempted")
			}
			prof.Recorder.Eventf(victim, preemptor, v1.EventTypeNormal, "Preempted", "Preempting", "Preempted by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)

		}
		metrics.PreemptionVictims.Observe(float64(len(victims)))
	}
	// 清空所有比抢占 pod 优先级级别低的 pod 的 NominatedNodeName
	for _, p := range nominatedPodsToClear {
		rErr := sched.podPreemptor.removeNominatedNodeName(p)
		if rErr != nil {
			klog.Errorf("Cannot remove 'NominatedPod' field of pod: %v", rErr)
			// We do not return as this error is not critical.
		}
	}
	return nodeName, err
}

// 抢占逻辑

func (g *genericScheduler) Preempt(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod, scheduleErr error) (*v1.Node, []*v1.Pod, []*v1.Pod, error) {
	
    // 检查错误类型
	fitError, ok := scheduleErr.(*FitError)
	if !ok || fitError == nil {
		return nil, nil, nil, nil
	}

	// 判断 pod 是否为 非抢占
	// 若 pod.nominatedNodeName 被设置，切在该节点上存在一个 pod (比其优先级低 && 处于删除状态)
	if !podEligibleToPreemptOthers(pod, g.nodeInfoSnapshot.NodeInfos(), g.enableNonPreempting) {
		klog.V(5).Infof("Pod %v/%v is not eligible for more preemption.", pod.Namespace, pod.Name)
		return nil, nil, nil, nil
	}

    // 获取快照中的所有 node
	allNodes, err := g.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, nil, nil, err
	}
	if len(allNodes) == 0 {
		return nil, nil, nil, ErrNoNodesAvailable
	}

	// 找到潜在 node
	potentialNodes := nodesWherePreemptionMightHelp(allNodes, fitError)

    // 若没有潜在 node ，返回
	if len(potentialNodes) == 0 {
		klog.V(3).Infof("Preemption will not help schedule pod %v/%v on any node.", pod.Namespace, pod.Name)
		return nil, nil, []*v1.Pod{pod}, nil
	}

    // 获取所有 pdb
	var pdbs []*policy.PodDisruptionBudget
	if g.pdbLister != nil {
		pdbs, err = g.pdbLister.List(labels.Everything())
		if err != nil {
			return nil, nil, nil, err
		}
	}

    // 找到适合抢占的所有节点信息
	nodeToVictims, err := g.selectNodesForPreemption(ctx, prof, state, pod, potentialNodes, pdbs)
	if err != nil {
		return nil, nil, nil, err
	}

	// 运行用户通过 extender 方式扩展的抢占逻辑 
	nodeToVictims, err = g.processPreemptionWithExtenders(pod, nodeToVictims)
	if err != nil {
		return nil, nil, nil, err
	}

    // 找到最合适的 node 
	candidateNode := pickOneNodeForPreemption(nodeToVictims)
	if candidateNode == nil {
		return nil, nil, nil, nil
	}

    // 获取该 node 节点上所有比该 pod 优先级低的 pod
	nominatedPods := g.getLowerPriorityNominatedPods(pod, candidateNode.Name)
	return candidateNode, nodeToVictims[candidateNode].Pods, nominatedPods, nil
}

// 获取潜在 node

func nodesWherePreemptionMightHelp(nodes []*schedulernodeinfo.NodeInfo, fitErr *FitError) []*schedulernodeinfo.NodeInfo {
	var potentialNodes []*schedulernodeinfo.NodeInfo
	for _, node := range nodes {
		name := node.Node().Name
        // 两种情况
        // Unschedulable
        // 表示 pod 通过抢占有可能被调度
        // 因为亲和性被过滤
        // 因为端口（node 上的端口，即 hostPort 存在冲突）被过滤
        // 因为资源被过滤
        // 因为 csi 被过滤
        // 因为 nonCSI 被过滤
        // 因为 pod 拓扑被过滤
        // 因为 ServiceAffinity 被过滤
        // 因为 VolumeRestrictions 被过滤

        // UnschedulableAndUnresolvable
        // 表示 pod 不能被调度
		if fitErr.FilteredNodesStatuses[name].Code() == framework.UnschedulableAndUnresolvable {
			continue
		}
		klog.V(3).Infof("Node %v is a potential node for preemption.", name)
		potentialNodes = append(potentialNodes, node)
	}
	return potentialNodes
}

// 获取所有适合的 node 节点

func (g *genericScheduler) selectNodesForPreemption(
	ctx context.Context,
	prof *profile.Profile,
	state *framework.CycleState,
	pod *v1.Pod,
	potentialNodes []*schedulernodeinfo.NodeInfo,
	pdbs []*policy.PodDisruptionBudget,
) (map[*v1.Node]*extenderv1.Victims, error) {
	nodeToVictims := map[*v1.Node]*extenderv1.Victims{}
	var resultLock sync.Mutex

	checkNode := func(i int) {
		nodeInfoCopy := potentialNodes[i].Clone()
		stateCopy := state.Clone()

        // pods: 要驱逐的 pod 数量，
        // numPDBViolations: 驱逐的 pod 中是 pdb 类型的总量
        // fits: 该 node 是否合适
		pods, numPDBViolations, fits := g.selectVictimsOnNode(ctx, prof, stateCopy, pod, nodeInfoCopy, pdbs)
		if fits {
			resultLock.Lock()
			victims := extenderv1.Victims{
				Pods:             pods,
				NumPDBViolations: int64(numPDBViolations),
			}
			nodeToVictims[potentialNodes[i].Node()] = &victims
			resultLock.Unlock()
		}
	}
    // 最多 16 个协程 min(16,len(potentialNodes))
	workqueue.ParallelizeUntil(context.TODO(), 16, len(potentialNodes), checkNode)
	return nodeToVictims, nil
}

// 尽可能保证 pdb 的规则
// 尽可能移除优先级低的 pod
func (g *genericScheduler) selectVictimsOnNode(
	ctx context.Context,
	prof *profile.Profile,
	state *framework.CycleState,
	pod *v1.Pod,
	nodeInfo *schedulernodeinfo.NodeInfo,
	pdbs []*policy.PodDisruptionBudget,
) ([]*v1.Pod, int, bool) {
	var potentialVictims []*v1.Pod

    // 移除 pod
	removePod := func(rp *v1.Pod) error {
		if err := nodeInfo.RemovePod(rp); err != nil {
			return err
		}
		status := prof.RunPreFilterExtensionRemovePod(ctx, state, pod, rp, nodeInfo)
		if !status.IsSuccess() {
			return status.AsError()
		}
		return nil
	}
    
    // 添加 pod
	addPod := func(ap *v1.Pod) error {
		nodeInfo.AddPod(ap)
		status := prof.RunPreFilterExtensionAddPod(ctx, state, pod, ap, nodeInfo)
		if !status.IsSuccess() {
			return status.AsError()
		}
		return nil
	}
	
    // 移除所有优先级低于抢占 pod 的 pods，并将这些 pod 记录在 potentialVictims 内
	podPriority := podutil.GetPodPriority(pod)
	for _, p := range nodeInfo.Pods() {
		if podutil.GetPodPriority(p) < podPriority {
			potentialVictims = append(potentialVictims, p)
			if err := removePod(p); err != nil {
				return nil, 0, false
			}
		}
	}
	
    // 判断移除完所有低优先级 pods 后，抢占 pod 是否可以调度至该节点
	if fits, _, err := g.podPassesFiltersOnNode(ctx, prof, state, pod, nodeInfo); !fits {
		if err != nil {
			klog.Warningf("Encountered error while selecting victims on node %v: %v", nodeInfo.Node().Name, err)
		}

		return nil, 0, false
	}

	var victims []*v1.Pod
	numViolatingVictim := 0

    // 按照优先级排队，优先级越高越在前面
    // 优先级一致，按照创建时间排队，创建的时间越长，越在前面
	sort.Slice(potentialVictims, func(i, j int) bool { return util.MoreImportantPod(potentialVictims[i], potentialVictims[j]) })

    // violatingVictims: 破坏 pdb 规则的 pods
    // nonViolatingVictims: 没有破坏 pdb 规则的pod
	violatingVictims, nonViolatingVictims := filterPodsWithPDBViolation(potentialVictims, pdbs)

    // 尝试将 pod 其添加回到节点上，看是否能满足 抢占pod 的调度
	reprievePod := func(p *v1.Pod) (bool, error) {
		if err := addPod(p); err != nil {
			return false, err
		}
		fits, _, _ := g.podPassesFiltersOnNode(ctx, prof, state, pod, nodeInfo)
        // 必须移除该 pod,添加到要移除的列表中
		if !fits {
			if err := removePod(p); err != nil {
				return false, err
			}
			victims = append(victims, p)
			klog.V(5).Infof("Pod %v/%v is a potential preemption victim on node %v.", p.Namespace, p.Name, nodeInfo.Node().Name)
		}
		return fits, nil
	}
    // 优先尝试破环了 pdb 规则 pod 是否可以继续在该节点上运行，尽可能不驱逐它
	for _, p := range violatingVictims {
		if fits, err := reprievePod(p); err != nil {
			klog.Warningf("Failed to reprieve pod %q: %v", p.Name, err)
			return nil, 0, false
		} else if !fits {
			numViolatingVictim++
		}
}
	// 移除正常的 pods，尽可能移除优先级比较低的 pod
	for _, p := range nonViolatingVictims {
		if _, err := reprievePod(p); err != nil {
			klog.Warningf("Failed to reprieve pod %q: %v", p.Name, err)
			return nil, 0, false
		}
	}
    // 返回移除的 pods 列表，和 破坏 pdb 规则的 pod 总和，以及该 node 适合抢占标识
	return victims, numViolatingVictim, true
}

// 从满足条件的 nodes 中获取最合适的 node 节点
func pickOneNodeForPreemption(nodesToVictims map[*v1.Node]*extenderv1.Victims) *v1.Node {
	if len(nodesToVictims) == 0 {
		return nil
	}
	minNumPDBViolatingPods := int64(math.MaxInt32)
	var minNodes1 []*v1.Node
	lenNodes1 := 0
    // 查找需要驱逐 pdb 类型 pod 数量最少的节点
	for node, victims := range nodesToVictims {
        // 该 node 无需驱逐其他 pod
		if len(victims.Pods) == 0 {
			return node
		}
		numPDBViolatingPods := victims.NumPDBViolations
		if numPDBViolatingPods < minNumPDBViolatingPods {
			minNumPDBViolatingPods = numPDBViolatingPods
			minNodes1 = nil
			lenNodes1 = 0
		}
		if numPDBViolatingPods == minNumPDBViolatingPods {
			minNodes1 = append(minNodes1, node)
			lenNodes1++
		}
	}
	if lenNodes1 == 1 {
		return minNodes1[0]
	}

    // 若 存在 pdb 被破坏的情况，victims.Pods[0] 为 violatingVictims 中优先级最高的 pod
    // 若不存在，victims.Pods[0] 为优先级最高的 pod
    // 找到最高优先级最低的 node
	minHighestPriority := int32(math.MaxInt32)
	var minNodes2 = make([]*v1.Node, lenNodes1)
	lenNodes2 := 0
	for i := 0; i < lenNodes1; i++ {
		node := minNodes1[i]
		victims := nodesToVictims[node]
		highestPodPriority := podutil.GetPodPriority(victims.Pods[0])
		if highestPodPriority < minHighestPriority {
			minHighestPriority = highestPodPriority
			lenNodes2 = 0
		}
		if highestPodPriority == minHighestPriority {
			minNodes2[lenNodes2] = node
			lenNodes2++
		}
	}
	if lenNodes2 == 1 {
		return minNodes2[0]
	}

	// 驱逐的 pods 的优先级之和最小
	minSumPriorities := int64(math.MaxInt64)
	lenNodes1 = 0
	for i := 0; i < lenNodes2; i++ {
		var sumPriorities int64
		node := minNodes2[i]
		for _, pod := range nodesToVictims[node].Pods {
			sumPriorities += int64(podutil.GetPodPriority(pod)) + int64(math.MaxInt32+1)
		}
		if sumPriorities < minSumPriorities {
			minSumPriorities = sumPriorities
			lenNodes1 = 0
		}
		if sumPriorities == minSumPriorities {
			minNodes1[lenNodes1] = node
			lenNodes1++
		}
	}
	if lenNodes1 == 1 {
		return minNodes1[0]
	}

    // 需要驱逐 pod 数量最少的 pod
	minNumPods := math.MaxInt32
	lenNodes2 = 0
	for i := 0; i < lenNodes1; i++ {
		node := minNodes1[i]
		numPods := len(nodesToVictims[node].Pods)
		if numPods < minNumPods {
			minNumPods = numPods
			lenNodes2 = 0
		}
		if numPods == minNumPods {
			minNodes2[lenNodes2] = node
			lenNodes2++
		}
	}
	if lenNodes2 == 1 {
		return minNodes2[0]
	}

	// 从每个node节点选取 优先级最高的，创建时间最久的pod,然后比较，取最新创建的
    // 选取第一个
	latestStartTime := util.GetEarliestPodStartTime(nodesToVictims[minNodes2[0]])
	if latestStartTime == nil {
		klog.Errorf("earliestStartTime is nil for node %s. Should not reach here.", minNodes2[0])
		return minNodes2[0]
	}
	nodeToReturn := minNodes2[0]
	for i := 1; i < lenNodes2; i++ {
		node := minNodes2[i]
		earliestStartTimeOnNode := util.GetEarliestPodStartTime(nodesToVictims[node])
		if earliestStartTimeOnNode == nil {
			klog.Errorf("earliestStartTime is nil for node %s. Should not reach here.", node)
			continue
		}
		if earliestStartTimeOnNode.After(latestStartTime.Time) {
			latestStartTime = earliestStartTimeOnNode
			nodeToReturn = node
		}
	}

	return nodeToReturn
}


```