# K8s 调度器

## EventHandler

### 结构体

```go
// scheduler 缓存
type schedulerCache struct {
	stop   <-chan struct{}
	ttl    time.Duration
	period time.Duration

	// This mutex guards all fields within this cache struct.
	mu sync.RWMutex
	// a set of assumed pod keys.
	// The key could further be used to get an entry in podStates.
	assumedPods map[string]bool // 已经分配 nodeName，但未 绑定的 pod
	// a map from pod key to podState.
	podStates map[string]*podState // pod 信息
	nodes     map[string]*nodeInfoListItem // node 信息，是一个双向链表,越靠近头部的节点，表示最近更新的 node 
	
    // 指向最近信息跟行的 node ，为 nodes 的头指针
	headNode *nodeInfoListItem
	nodeTree *nodeTree
	// A map from image name to its imageState.
	imageStates map[string]*imageState // 拥有该镜像的节点列表
}

// 节点信息
type NodeInfo struct {
	node *v1.Node
    // 该 node 节点上所有的 pod
	pods             []*v1.Pod

    // 该节点上具有亲和性的 pod
	podsWithAffinity []*v1.Pod 

    // 若 pod 使用宿主机端口，进行记录，结构体 map[node.ip]map[ProtocolPort]struct{}
	usedPorts        HostPortInfo 

    // 该节点上所有 pod 请求的资源总和，包括 assumed pods,发送了 binding 信息，但还未完成 bind
	requestedResource *Resource 

    // 资源请求量，比 requestedResource 更加精细一些，应为若 pod 未设置 资源请求量，使用默认值，100m 200Mi
	nonzeroRequest *Resource 
	
    // 该节点上可调度的资源总和
	allocatableResource *Resource

	// 污点
	taints    []v1.Taint
	taintsErr error

	// 节点上的 所有镜像
	imageStates map[string]*ImageStateSummary

	// 准备被删除
	TransientInfo *TransientSchedulerInfo

	// 节点 内存，磁盘，pid 是否存在压力
	memoryPressureCondition v1.ConditionStatus
	diskPressureCondition   v1.ConditionStatus
	pidPressureCondition    v1.ConditionStatus

	// 若 nodeInfo 发生变化 ，自增1 ，防止未发生变化而克隆
	generation int64
}
```

### Pod

```text
pod infmormer 分为已调度的 pod 和未调度的 pod。
```

#### scheduled pod

##### 源码解析

1. informer 过滤

```go
// 过滤掉 nodeName 未设置的 pod
podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
                    // assignedPod : 若 pod.nodeName 为空，返回 false
					return assignedPod(t) 
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return assignedPod(pod)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)

func assignedPod(pod *v1.Pod) bool {
	return len(pod.Spec.NodeName) != 0
}
```

2. addPodToCache
    - 将 pod 添加到 cache 记录中
    - 更新 调度 node 信息

```go
func (sched *Scheduler) addPodToCache(obj interface{}) {
	pod, ok := obj.(*v1.Pod)
	if !ok {
		klog.Errorf("cannot convert to *v1.Pod: %v", obj)
		return
	}
	klog.V(3).Infof("add event for scheduled pod %s/%s ", pod.Namespace, pod.Name)

    // 添加 pod 信息
	if err := sched.SchedulerCache.AddPod(pod); err != nil {
		klog.Errorf("scheduler cache AddPod failed: %v", err)
	}

	sched.SchedulingQueue.AssignedPodAdded(pod)
}

func (cache *schedulerCache) AddPod(pod *v1.Pod) error {
    // key 为 pod.uid
	key, err := schedulernodeinfo.GetPodKey(pod)
	if err != nil {
		return err
	}

	cache.mu.Lock()
	defer cache.mu.Unlock()

    // 本地是否记录
    // 若已经记录，切为预备调度，判断 nodeName 是否一致，不一致跟新信息
    // 未记录，记录 pod 信息
	currState, ok := cache.podStates[key]
	switch {
	case ok && cache.assumedPods[key]:
		if currState.pod.Spec.NodeName != pod.Spec.NodeName {
			// The pod was added to a different node than it was assumed to.
			klog.Warningf("Pod %v was assumed to be on %v but got added to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
			// 移除旧 pod 信息
			if err = cache.removePod(currState.pod); err != nil {
				klog.Errorf("removing pod error: %v", err)
			}
			cache.addPod(pod)
		}
		delete(cache.assumedPods, key)
		cache.podStates[key].deadline = nil
		cache.podStates[key].pod = pod
	case !ok:
		// Pod was expired. We should add it back.
		cache.addPod(pod)
		ps := &podState{
			pod: pod,
		}
		cache.podStates[key] = ps
	default:
		return fmt.Errorf("pod %v was already in added state", key)
	}
	return nil
}

// 添加 pod 信息函数
// 判断 pod 所属的 node 是否存在，不存在，初始化
// 调用 AddPod 函数为 node 添加 pod 信息
func (cache *schedulerCache) addPod(pod *v1.Pod) {
	n, ok := cache.nodes[pod.Spec.NodeName]
	if !ok {
		n = newNodeInfoListItem(schedulernodeinfo.NewNodeInfo())
		cache.nodes[pod.Spec.NodeName] = n
	}
	n.info.AddPod(pod)
    // 将该 cache 的 nodes 链表头指针指向该 node
	cache.moveNodeInfoToHead(pod.Spec.NodeName)
}

// 添加 pod 信息具体处理函数
// 1. 累加资源请求量 (max(sum(podSpec.Containers), podSpec.InitContainers) + overHead) 
// nonzeroRequest：若未设置资源请求量，采用默认值 100m 200Mi
// 2. 记录具有亲和性的 pod
// 3. 若使用了 hostPort，记录端口使用信息
// 4. 记录 pod
func (n *NodeInfo) AddPod(pod *v1.Pod) {
	res, non0CPU, non0Mem := calculateResource(pod)
	n.requestedResource.MilliCPU += res.MilliCPU
	n.requestedResource.Memory += res.Memory
	n.requestedResource.EphemeralStorage += res.EphemeralStorage
	if n.requestedResource.ScalarResources == nil && len(res.ScalarResources) > 0 {
		n.requestedResource.ScalarResources = map[v1.ResourceName]int64{}
	}
	for rName, rQuant := range res.ScalarResources {
		n.requestedResource.ScalarResources[rName] += rQuant
	}
	n.nonzeroRequest.MilliCPU += non0CPU
	n.nonzeroRequest.Memory += non0Mem
	n.pods = append(n.pods, pod)
	if hasPodAffinityConstraints(pod) {
		n.podsWithAffinity = append(n.podsWithAffinity, pod)
	}

	// Consume ports when pods added.
	n.UpdateUsedPorts(pod, true)
    // nodeInfo 改变，自增一
	n.generation = nextGeneration()
}

// 移除 pod，操作为 移除 pod 在 node 上的记录的信息
// node 不存在直接返回
func (cache *schedulerCache) removePod(pod *v1.Pod) error {
	n, ok := cache.nodes[pod.Spec.NodeName]
	if !ok {
		return nil
	}
	if err := n.info.RemovePod(pod); err != nil {
		return err
	}
    // 将该 node 移动值 链表首部
	cache.moveNodeInfoToHead(pod.Spec.NodeName)
	return nil
}

// 移除 pod 详细操作
// 1. 冲 node.podsWithAffinity 列表中删除该 pod(若该 pod 存在 亲和性)
// 2. 删除 pod 在 node 上的 记录
// 3. 释放 pod 所占用的资源
// 4. 释放 pod 占用的 node 端口 （若）
func (n *NodeInfo) RemovePod(pod *v1.Pod) error {
	k1, err := GetPodKey(pod)
	if err != nil {
		return err
	}

    // 在亲和性列表 pod 中移除该 pod
	for i := range n.podsWithAffinity {
		k2, err := GetPodKey(n.podsWithAffinity[i])
		if err != nil {
			klog.Errorf("Cannot get pod key, err: %v", err)
			continue
		}
		if k1 == k2 {
			// delete the element
			n.podsWithAffinity[i] = n.podsWithAffinity[len(n.podsWithAffinity)-1]
			n.podsWithAffinity = n.podsWithAffinity[:len(n.podsWithAffinity)-1]
			break
		}
	}
    // 删除 pod 在 node 上的记录
    // 释放 该 pod 在该 node 上占有的资源
    // 释放端口，若该 pod 使用了宿主机端口
	for i := range n.pods {
		k2, err := GetPodKey(n.pods[i])
		if err != nil {
			klog.Errorf("Cannot get pod key, err: %v", err)
			continue
		}
		if k1 == k2 {
			// delete the element
			n.pods[i] = n.pods[len(n.pods)-1]
			n.pods = n.pods[:len(n.pods)-1]
			// reduce the resource data
			res, non0CPU, non0Mem := calculateResource(pod)

			n.requestedResource.MilliCPU -= res.MilliCPU
			n.requestedResource.Memory -= res.Memory
			n.requestedResource.EphemeralStorage -= res.EphemeralStorage
			if len(res.ScalarResources) > 0 && n.requestedResource.ScalarResources == nil {
				n.requestedResource.ScalarResources = map[v1.ResourceName]int64{}
			}
			for rName, rQuant := range res.ScalarResources {
				n.requestedResource.ScalarResources[rName] -= rQuant
			}
			n.nonzeroRequest.MilliCPU -= non0CPU
			n.nonzeroRequest.Memory -= non0Mem

			// Release ports when remove Pods.
			n.UpdateUsedPorts(pod, false)

			n.generation = nextGeneration()
			n.resetSlicesIfEmpty()
			return nil
		}
	}
	return fmt.Errorf("no corresponding pod %s in pods of node %s", pod.Name, n.node.Name)
}
```

3. updatePodInCache

```go
func (sched *Scheduler) updatePodInCache(oldObj, newObj interface{}) {
	oldPod, ok := oldObj.(*v1.Pod)
	if !ok {
		klog.Errorf("cannot convert oldObj to *v1.Pod: %v", oldObj)
		return
	}
	newPod, ok := newObj.(*v1.Pod)
	if !ok {
		klog.Errorf("cannot convert newObj to *v1.Pod: %v", newObj)
		return
	}
	// 修改 pod
	if err := sched.SchedulerCache.UpdatePod(oldPod, newPod); err != nil {
		klog.Errorf("scheduler cache UpdatePod failed: %v", err)
	}

	sched.SchedulingQueue.AssignedPodUpdated(newPod)
}


```