### es
#分片
	1.number_of_shards 
		主分片，设置后不可能再调整，写操作时总是先写入主分片，然后异步从主分片同步到各个从分片。
	2.number_of_replicas
		从分片，设置后可再调整，读操作可以是任何分片（包括主分片）
		
	如果主分片和从分片都在一个node上的时候就会提示 yellow，所以主分片数量可以超过节点数会提示green，但从分片超过节点数时会提示yellow，因为从分片超过节点数
	就一定会有从分片于主分片在同一个节点上。
	
#节点:
	主节点
		只有主节点知道所有其他节点的位置，所以集群时，master主要也作为集群的协调角色。当master宕机，会从从节点中选举出一个成为master
		所有请求都是由master分配处理，如果请求发送到了其他节点，即便其他节点上存在本次的主分片，也依然会把请求转发给master，然后由master处理转发。
		当操作是写操作时：master会根据 document id 来判断这个文档的主分片在哪个node，然后将消息转发给这个node。
		当操作是读操作时：master会根据 document id 来判断这个文档在哪些node，然后根据负载均衡选择读取其中一个。（于写操作不同，它不会只选择主分片，所以会有一定同步延迟）
		
	从节点
			

match: 会进行分词，然后拿分词进行模糊比配，并附上 _score
term: 不会进行分词，直接拿整个词进行模糊，没有_score