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



3. `https://github.com/jeffrey-xiao/kademlia-dht-rs /src/node/mod.rs lookup_nodes`

```rust
    /// Iteratively looks up nodes to determine the closest nodes to `key`. The search begins by
    /// selecting `CONCURRENCY_PARAM` nodes in the routing table and adding it to a shortlist. It
    /// then sends out either `FIND_NODE` or `FIND_VALUE` RPCs to `CONCURRENCY_PARAM` nodes not yet
    /// queried in the shortlist. The node will continue to fill its shortlist until it did not find
    /// a closer node for a round of RPCs or if runs out of nodes to query. Finally, it will query
    /// the remaining nodes in its shortlist until there are no remaining nodes or if it has found
    /// `REPLICATION_PARAM` active nodes.
    fn lookup_nodes(&mut self, key: &Key, find_node: bool) -> ResponsePayload {
        //首先加锁从本地routetable中请求离key最近的Nodes.
        let routing_table = self.routing_table.lock().unwrap();
        let closest_nodes = routing_table.get_closest_nodes(key, CONCURRENCY_PARAM);
        drop(routing_table); //代表立即释放锁。

        //通过比较找出本地closest nodes中最小距离Node。
        let mut closest_distance = Key::new([255u8; KEY_LENGTH]);
        for node_data in &closest_nodes {
            closest_distance = cmp::min(closest_distance, key.xor(&node_data.id))
        }

        // initialize found nodes, queried nodes, and priority queue
        //标记发现看到过的Nodes.
        let mut found_nodes: HashSet<NodeData> = closest_nodes.clone().into_iter().collect();
        found_nodes.insert((*self.node_data).clone());
        //标记已经请求过的Nodes.
        let mut queried_nodes = HashSet::new();
        queried_nodes.insert((*self.node_data).clone());

        //标记待请求优先级队列（按距离）
        let mut queue: BinaryHeap<NodeDataDistancePair> = BinaryHeap::from(
            closest_nodes
                .into_iter()
                .map(|node_data| NodeDataDistancePair(node_data.clone(), node_data.id.xor(key)))
                .collect::<Vec<NodeDataDistancePair>>(),
        );

        let (tx, rx) = channel(); //用于收集findnode rpc response.

        let mut concurrent_thread_count = 0; //用于累计并发异步请求数。

        // spawn initial find requests
        for _ in 0..CONCURRENCY_PARAM {
            if !queue.is_empty() { //开始并发异步findnode rpc.
                self.clone().spawn_find_rpc( 
                    queue.pop().unwrap().0,
                    key.clone(),
                    tx.clone(),
                    find_node,
                );
                concurrent_thread_count += 1;
            }
        }

        // loop until we could not find a closer node for a round or if no threads are running
        while concurrent_thread_count > 0 { //如果>0, 说明需要收集处理findnode rpc reponse.
            //如果发起的并发异步findnode rpc不够数，且待请求优先级队列不空，则继续并发异步请求findnode rpc.
            while concurrent_thread_count < CONCURRENCY_PARAM && !queue.is_empty() {
                self.clone().spawn_find_rpc(
                    queue.pop().unwrap().0,
                    key.clone(),
                    tx.clone(),
                    find_node,
                );
                concurrent_thread_count += 1;
            }

            let mut is_terminated = true; //用于标记是否停止继续逼近closet distance node.
            let response_opt = rx.recv().unwrap(); //同步阻塞等待findnode rpc reponse.
            concurrent_thread_count -= 1; //收到一个请求，意味着一个findnode rpc 完毕,故此减一。

            //具体处理findnode rpc response.
            match response_opt { 
                Some(Response {
                    payload: ResponsePayload::Nodes(nodes),
                    receiver,
                    ..
                }) => {
                    //本case处理远端Node反馈的closet nodes.
                    queried_nodes.insert(receiver); //将此给出响应的Node加入已请求Node集合。
                    for node_data in nodes { //处理此远端Node的closet node 集合（离target key）。
                        let curr_distance = node_data.id.xor(key);

                        if !found_nodes.contains(&node_data) { //如果从未发现看到过，则加入处理， 否则丢弃。
                            if curr_distance < closest_distance {
                                //如果发现更进的距离，即更近的Node， 则记录之。
                                closest_distance = curr_distance;
                                is_terminated = false; //发现比上一轮更近的Node, 意味着逼近需要继续，不要停止。
                            }

                            found_nodes.insert(node_data.clone()); //标记已经发现看到了此Node.
                            let dist = node_data.id.xor(key);
                            let next = NodeDataDistancePair(node_data.clone(), dist);
                            queue.push(next.clone()); //将此Node加入待请求优先级队列。
                        }
                    }
                }
                Some(Response {
                    payload: ResponsePayload::Value(value),
                    ..
                }) => return ResponsePayload::Value(value),
                _ => is_terminated = false, //默认case ,继续逼近closet node.
            }

            if is_terminated { //如果此标志为true, 则停止此循环迭代逼近。
                break;
            }
            debug!("CURRENT CLOSEST DISTANCE IS {:?}", closest_distance);
        }

        //走到此处代表逼近得到closet nodes. 或者没有findnode rpc 需要处理了。
        debug!(
            "{} TERMINATED LOOKUP BECAUSE NOT CLOSER OR NO THREADS WITH DISTANCE {:?}",
            self.node_data.addr, closest_distance,
        );

        //一般来说即使不足REPLICATION_PARAM个数Node，也不必再去逼近，故此下面while块可以忽略。
        // loop until no threads are running or if we found REPLICATION_PARAM active nodes
        while queried_nodes.len() < REPLICATION_PARAM {  //意思是说如果未凑足REPLICATION_PARAM， 则需继续逼近，直到凑足数。
            while concurrent_thread_count < CONCURRENCY_PARAM && !queue.is_empty() {
                self.clone().spawn_find_rpc(
                    queue.pop().unwrap().0,
                    key.clone(),
                    tx.clone(),
                    find_node,
                );
                concurrent_thread_count += 1;
            }
            if concurrent_thread_count == 0 {
                break;
            }

            let response_opt = rx.recv().unwrap();
            concurrent_thread_count -= 1;

            match response_opt {
                Some(Response {
                    payload: ResponsePayload::Nodes(nodes),
                    receiver,
                    ..
                }) => {
                    queried_nodes.insert(receiver);
                    for node_data in nodes {
                        if !found_nodes.contains(&node_data) {
                            found_nodes.insert(node_data.clone());
                            let dist = node_data.id.xor(key);
                            let next = NodeDataDistancePair(node_data.clone(), dist);
                            queue.push(next.clone());
                        }
                    }
                }
                Some(Response {
                    payload: ResponsePayload::Value(value),
                    ..
                }) => return ResponsePayload::Value(value),
                _ => {}
            }
        }

        //对已请求过的node集合按距离从小到大排序，然后返回指定个数Nodes.
        let mut ret: Vec<NodeData> = queried_nodes.into_iter().collect();
        ret.sort_by_key(|node_data| node_data.id.xor(key));
        ret.truncate(REPLICATION_PARAM);
        debug!("{} -  CLOSEST NODES ARE {:#?}", self.node_data.addr, ret);
        ResponsePayload::Nodes(ret)
    }
```

> 关切点：
>
> (1)  `findnode rpc 调用应该实现timeout机制，避免无限期无意义等待`。
>
> (2) 上面算法实现中，对于`未响应的peer node 或rpc error的Node`, 应该从本地queried_nodes中删除。
>
> (3) 如果lookup_nodes本身也可以并发调用， 那么不同`findnode rpc response`如何区分？每一个`rpc request and response`需要排队串行或者统统附加上`target key, and rpc sequence number or ID or Token`？ 总之需要唯一标记每一个`rpc request and respose 与lookup_nodes的对应关系`,这需要设计关切和决策。
>
> (4) `Kademlia paper`及其许多相关设计和实现都建议，采用`UDP`来管理和维护`Kademlia Overlay Network`, 这样比价轻便，无状态，代价小。`但是RPC request and repose设计实现需要自己考虑lookup_nodes -- findnode request -- findnode response的映射关系`。











