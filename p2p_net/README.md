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
		nodes := <-done //取出远端返回的node set.
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
	} //注意本算法只是个尽力算法， 并没有满足closet node set原则， 找到的Node只是接近！只是尝试询问delta个node，//就停止继续call findnode,去逼近closet node .

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



