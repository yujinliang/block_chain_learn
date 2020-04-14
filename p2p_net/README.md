# 						`Kademlia Lookup nodes各种算法收集分析`

1. ​	`https://github.com/cfromknecht/kademlia /findnode.go IterativeFindNode ` 

```go
//for golang
func (k *Kademlia) IterativeFindNode(target NodeID, delta int, final chan Contacts) {
	done := make(chan Contacts)

	ret := make(Contacts, BucketSize) //最终返回给caller的node set
	frontier := make(Contacts, BucketSize)//从未看到过的node set for findnode calling.
	seen := make(map[string]struct{}) //标记已经看到过的node set

	//1. 从本地route table尽可能找出delta个离target最近的node.
	for _, node := range k.routes.FindClosest(target, delta) {
		ret = append(ret, node)
		heap.Push(&frontier, node)
		seen[node.ID.String()] = struct{}{}
	}

	pending := 0
	for i := 0; i < delta && frontier.Len() > 0; i++ {
		//并发异步call findnode for each contact
		pending++
		contact := heap.Pop(&frontier).(Contact)
		go k.FindNode(contact, target, done)
	}

	for pending > 0 {
		//pending累计findnode call count.
		nodes := <-done //注意此处为同步阻塞等待取出远端返回的node set.
		pending--
		for _, node := range nodes {
			if _, ok := seen[node.ID.String()]; !ok {
				//只收集从未看到过的node.
				ret = append(ret, node) //收集之。
				heap.Push(&frontier, node)//之前未看到过的Node用于下次findnode.
				seen[node.ID.String()] = struct{}{} //标记已看到的Node.
			}
		}

		for pending < delta && frontier.Len() > 0 {
			//如果并未达到delta个数且frontier中还有之前从未看到过的Node, then call findnode 
			pending++
			contact := heap.Pop(&frontier).(Contact)
			go k.FindNode(contact, target, done)
		}
	} //注意本算法只是个尽力算法， 并没有满足closet node 原则， 找到的Node只是接近！只是尝试询问delta个node，//就停止继续call findnode,去逼近closet node .

	//通俗地说，首先找出本地离target最近的n个node, 进而同时findnode询问其离target的node set, 收集回复的node set ，并且排除已经看到过的，
	//如果询问的node个数达到delta，则放弃继续逼近closet node, 直接针对ret结果集按距离从小到大排序后返回！
	//delta越大则越有可能逼近找到closet node, 但是也会越耗时， lookup nodes收敛终止的代价越大！
	//原始kademlia paper中这样描述：标记closet node, 每一轮findnode完成结果收集，都去检测closet node是否变化，不变说明逼近成功，停止逼近，否则继续call findnode逼近！
	//delta越小性能越好，但是结果越不准确。

	sort.Sort(ret) //按id distance从小到大排序， 只返回前面最小的BucketSize个。
	if ret.Len() > BucketSize {
		ret = ret[:BucketSize]
	}

	final <- ret
}

```

> `kademlia 遵循的原理：熟人扎堆`， 万能的朋友圈，圈套圈，总能找到你。



2. ​	`https://github.com/prettymuchbryce/kademlia /dht.go iterate`

```go
func (dht *DHT) iterate(t int, target []byte, data []byte) (value []byte, closest []*NetworkNode, err error) {
	sl := dht.ht.getClosestContacts(alpha, target, []*NetworkNode{})

	// We keep track of nodes contacted so far. We don't contact the same node
	// twice.
	//标记已经请求过的Node.
	var contacted = make(map[string]bool)

	// According to the Kademlia white paper, after a round of FIND_NODE RPCs
	// fails to provide a node closer than closestNode, we should send a
	// FIND_NODE RPC to all remaining nodes in the shortlist that have not
	// yet been contacted.
	//标记是否继续请求其余未请求Node.
	queryRest := false

	// We keep a reference to the closestNode. If after performing a search
	// we do not find a closer node, we stop searching.
	//如果本地routetable中没有找到closet contacts, 则无法对外发起请求。
	if len(sl.Nodes) == 0 {
		return nil, nil, nil
	}

	//记录closetNode , 用于检测每一轮findnode之后是否查找到更近的Node。
	closestNode := sl.Nodes[0]

	if t == iterateFindNode {
		bucket := getBucketIndexFromDifferingBit(target, dht.ht.Self.ID)
		dht.ht.resetRefreshTimeForBucket(bucket)
	}

	removeFromShortlist := []*NetworkNode{}

	for {//一次循环代表一轮
		expectedResponses := []*expectedResponse{}
		numExpectedResponses := 0

		// Next we send messages to the first (closest) alpha nodes in the
		// shortlist and wait for a response

		//遍历本地routetable中查找的离target最近的Node集合。
		for i, node := range sl.Nodes {
			// Contact only alpha nodes
			//同时并发异步请求的Node个数不得大于alpha.
			//queryRest 用于标定是否继续请求sl中剩余未请求Node。
			if i >= alpha && !queryRest {
				break
			}

			// Don't contact nodes already contacted
			//排除已经请求过的Node。
			if contacted[string(node.ID)] == true {
				continue
			}

			contacted[string(node.ID)] = true //标记为已经请求过的状态。
			query := &message{}
			query.Sender = dht.ht.Self
			query.Receiver = node

			switch t {
			case iterateFindNode:
				query.Type = messageTypeFindNode
				queryData := &queryDataFindNode{}
				queryData.Target = target
				query.Data = queryData
			case iterateFindValue:
				query.Type = messageTypeFindValue
				queryData := &queryDataFindValue{}
				queryData.Target = target
				query.Data = queryData
			case iterateStore:
				query.Type = messageTypeFindNode
				queryData := &queryDataFindNode{}
				queryData.Target = target
				query.Data = queryData
			default:
				panic("Unknown iterate type")
			}

			// Send the async queries and wait for a response
			//异步并发
			res, err := dht.networking.sendMessage(query, true, -1)
			if err != nil {
				// Node was unreachable for some reason. We will have to remove
				// it from the shortlist, but we will keep it in our routing
				// table in hopes that it might come back online in the future.
				//收集没有响应的Node。
				removeFromShortlist = append(removeFromShortlist, query.Receiver)
				continue
			}
			//收集已发出rpc请求的返回结果，用于随后集中等待收集peer node 响应结果。
			expectedResponses = append(expectedResponses, res)
		}

        //从sl(closet node set)中删除无响应者！切记：并非从route table中删除！因为去中心，分布式网络模式特性，决定了每一个节点随时可能上线或下线。
		//通俗地讲：离自己最近的熟人朋友，也不可能随叫随到！但是不能因为未响应就认定不再是熟人朋友！这是不稳定的！
		for _, n := range removeFromShortlist {
			sl.RemoveNode(n)
		}

		numExpectedResponses = len(expectedResponses) //统计需要等待收集结果的数量。


		resultChan := make(chan (*message)) //将结果统一汇入此channel.
		for _, r := range expectedResponses {
			//依次遍历需要收集结果的rpc 响应集， 并且为每一个rpc response启动一个独立的go 协程监测响应。
			go func(r *expectedResponse) {
				select {
				case result := <-r.ch:
					//每一个rpc 请求成功后都会返回一个channel，用于汇报远端响应结果。
					if result == nil {
						// Channel was closed
						return
					}
					dht.addNode(newNode(result.Sender)) //更新routetable, 认识result.Sender。
					resultChan <- result //向外层继续汇总每一个go 协程的响应结果。
					return
				case <-time.After(dht.options.TMsgTimeout):
					dht.networking.cancelResponse(r) //取消这个已经超时的rpc.
					return
				}
			}(r)
		}

		//同步等待收集最终rpc响应结果。
		var results []*message
		if numExpectedResponses > 0 {
		Loop:
			for {
				select {
				case result := <-resultChan:
					if result != nil {
						results = append(results, result)
					} else {
						numExpectedResponses--
					}
					if len(results) == numExpectedResponses {
						close(resultChan)
						break Loop
					}
				case <-time.After(dht.options.TMsgTimeout):
					close(resultChan)
					break Loop
				}
			}

			for _, result := range results {
				if result.Error != nil {
					sl.RemoveNode(result.Receiver) //去除未响应Node。
					continue
				}
				switch t {
				case iterateFindNode:
					responseData := result.Data.(*responseDataFindNode)
					//将某一个rpc 响应结果收入sl(本地closet node set)
					sl.AppendUniqueNetworkNodes(responseData.Closest)
				case iterateFindValue:
					responseData := result.Data.(*responseDataFindValue)
					// TODO When an iterativeFindValue succeeds, the initiator must
					// store the key/value pair at the closest node seen which did
					// not return the value.
					if responseData.Value != nil {
						return responseData.Value, nil, nil
					}
						//将某一个rpc 响应结果收入sl(本地closet node set)
					sl.AppendUniqueNetworkNodes(responseData.Closest)
				case iterateStore:
					responseData := result.Data.(*responseDataFindNode)
						//将某一个rpc 响应结果收入sl(本地closet node set)
					sl.AppendUniqueNetworkNodes(responseData.Closest)
				}
			}
		}

		//停止逼近closet node , 当queryRest=true && sl(本地closet node set)为空，
		//意味着lookup closet nodes失败了。
		if !queryRest && len(sl.Nodes) == 0 {
			return nil, nil, nil
		}

		sort.Sort(sl) //太重要了，按距离从小到大对sl排序。

		// If closestNode is unchanged then we are done
		//关键！！！ 停止逼近closet node的决定性条件
		//即本轮findnode rpc response处理后，监测closet node是否与上一轮不同，相同不变则停止逼近，或queryRest=true时停止逼近。
		if bytes.Compare(sl.Nodes[0].ID, closestNode.ID) == 0 || queryRest {
			// We are done
			switch t {
			case iterateFindNode:
				if !queryRest {
					queryRest = true
					continue
				}
				return nil, sl.Nodes, nil
			case iterateFindValue:
				return nil, sl.Nodes, nil
			case iterateStore:
				//向closet nodes 发送store命令。
				for i, n := range sl.Nodes {
					if i >= k {
						return nil, nil, nil
					}

					query := &message{}
					query.Receiver = n
					query.Sender = dht.ht.Self
					query.Type = messageTypeStore
					queryData := &queryDataStore{}
					queryData.Data = data
					query.Data = queryData
					dht.networking.sendMessage(query, false, -1)
				}
				return nil, nil, nil
			}
		} else {
			//非常关键， 记录本轮findnode rpc 响应中closet node. 每一轮找出一个最近的node全局记录，用于与下一轮找出的closet node比较，
			//相同不变，则停止逼近closet node, 否则继续。
			closestNode = sl.Nodes[0]
		}
	}
}

```

> `kademlia paper`中说，lookup nodes直到找不出更近的closet node为止, 我想首先`kademlia p2p overlay network`中每个节点都必须诚实且严格遵守`kademlia protocol` , 其次`kademlia `这种熟人朋友抱团扎堆的喜好，下线越多，lookup nodes失败的可能性越大或者说精度越差； 总结关键在于`kademlia node id`的合法性， 可否验证身份！不能随便模拟！
>
> 万能的朋友圈，但不是全能的！lookup nodes也许成功，也许是失败， 也许找到近似接近closet node。
>
> 大家（所有节点）按同一规则扎堆结伴(distance)，按同一个规则 (distance)找人， 理论上一定能找到target所在的熟人（朋友）圈， 除非节点不诚实或下线的比例太大。
>
> 当探测到一个node节点无反应(下线)时， 是否立即将其删除，加入新节点！若是则网络可用性提高， 但是可靠性和安全性下降，`kademlia`倾向于不会轻易删除未响应节点， 从而有效避免一定的安全风险！俗话说，好朋友并非天天腻乎在一起！喜新厌旧容易招致不安全。
>
> `kademlia`挺有人情味的，不轻易抛弃放弃。



- `kademlia attack`

> underlay network(底层网络):
>
> Spoofing, Eavesdropping, Packet modifications(欺骗、窃听、数据包修改)
>
> Overlay routing:
> Eclipse attack(日蚀攻击)
> Sybil attack(梅比尔攻击)
> Adversarial routing(对抗路由)
>
> Other attacks:
> Denial‐of‐Service
> Data Storage
>
> Overlay network must provide end‐to‐end security(覆盖网络必须提供端到端的安全性)
>
> Attacker: Cuts off a part of the network(攻击者：切断网络的一部分)
>
> 加入大量不诚实的伪装节点，从而控制overlay network.
>
> 导致：Lookups fail, data corruption, partitioning。
>
> 对策：Prevent a node from choosing its ID freely（关键在于`kademlia node id`的合法性， 可否验证身份！不能随便模拟！）
>
> [具体方案A]
>
> `Simple solution: Use NodeID := H( IP + Port )
> No authentication, problems with NAT(IP地址不固定)
> IP spoofing still possible（仍然可IP欺骗）`
>
> ---
>
> [具体方案B]
>
> `Better solution: Cryptographic NodeID
> NodeID := H( public‐key ) （采用不对称加密技术， 对公钥hash得到NodeID, 好处多多）
> Allows authentication, key exchange, signing messages（可以验证身份，签名消息）`
>
> 很显然方案B非常优秀，但是秘钥的生成管理也是个问题，当然由中心化的权威组织发放，则安全最有保障，但是不符合`p2p、区块链`去中心化的思想；如果私人自己生成秘钥，虽然有效降低了风险，但是仍然存在风险漏洞。















