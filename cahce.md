web cache主干
===

main()
=====
	assemblePipeline（）	// 组装pipeline
		CCachePipeline::open（）// 注册15个ACE_Module，
	csvc->open() 	 // 启动cache serverice 
 	thrman.spawn_n(threadsnumber, &loader_event_loop, NULL);   //  loader_event_loop不重要，只是等待信号，不处理请求。

CCacheService
=============
open()
----
	配置服务端口，log server，
	连接redis服务         //  百度问问结果替换为搜狗问问，存储对应关系
	qdbAgent->open()  // 初始化
	CCacheHandler::instance()->open()  主要的服务启动动作，接收请求


CCacheHandler
==============
open()
----
	activate()  // 自己激活自己
svc()	
---
	handleSuccessRequest(CCacheQueuedRequest* req, std::string* request)
		req->request.swap(*request); // 将search_hub的请求存储到了CCacheQueuedRequest的request成员中。



CLocalSearchTask
=================
cacheID 是search_hub计算后传过来的 

svc()
---
	req->parseRequest() //解析请求
	dispatchCCacheQueuedRequest // 查local cache，分发请求，是localSearch 的主要逻辑
		if searchtype == URLLOOKUP :  
			 // urllookup 含义：summary的key是通过url计算得到的，所以直接查询summary即可
			请求summary服务。 // CSend2QueryTask::instance()->put函数中根据shouldQuery_确定是否请求summary服务
      
		NORMQUERY:
		SITEQUERY1:
		SITEQUERY2:   
			ReadCacheEntry；  //  读cache
			如果isLocationalRequest，则更新cacheID（只在二查时更新）
			query cache没命中：
				if  interOnly：  
					直接返回空        //  interOnly 含义：特殊请求，只查缓存，减小压力。   
				if cacheLevel == 1：   
					直接返回空     // cacheLevel从seach_hub传来的参数，特殊渠道请求，只查缓存。      
				else  
					准备请求倒排服务     
			query cache命中：
				(1)  过期，准备请求倒排服务；
						setAllInstQueryExpired()中设置了请求倒排标记      
				(2) 查询结果不够，准备请求倒排服务；
						在not_enough(req) 中设置了请求倒排标记      
				(3)  cacheRequest.needdump_，排查用，debug				
				(4) if is_HitAll , 判断函数中访问了正排cache
						做了读取summary cache的动作，太隐蔽了！
						hit4, hit5，命中summary cache，读取了结果后直接返回。
						CCachePolicy::judge_summary
							CQdbAgent::instance()->LoadSummaryContent
				(5) else if req->shouldSummary_ // 需要请求summary server
						CQdbAgent::instance()->LoadSummaryContent

			if shouldQuery_	// 需要请求query server
				if cacheLevel == 1:  // hitcache2
					只请求summary服务
				else 
					请求倒排服务
			else
				if needdump_ || eplace_url: 
					CPreMergeQueryTask
				else
					请求正排服务

		CCachePolicy::instance()->getConvertContentResult(req)作用：阻塞等待解析结果。

HITCACHE_CACHEENTRY：如何确定命中了一部分？？

hit1：HITCACHE_NONE，无倒排cache,  首次访问的query。
hit2：HITCACHE_CACHEENTRY，命中部分倒排cache, 部分倒排cache过期。
hit3：HITCACHE_DOCID，命中倒排cache，无正排cache，一般为翻页请求
hit4：HITCACHE_ALL，命中倒排cache，硬盘中正排cache。
hit5：HITCACHE_MEM，命中倒排cache，内存中正排cache。


CSend2QueryTask
========================
back_id含义：用于确定连接query server的client，MTransport *transport = &collect_clients_[server_id][back_id]
CollectResultBase 中有记录连接query server用的类似连接池的二维数组 
MTransport collect_clients_[s_max_server_cnt][s_max_back_cnt]
s_max_back_cnt 表示每个web_cache上与query server建立连接的client数量上限

back_id = back_index_[actual_group_id]        //  标记使用的collect client
server_id = queryServerGroups[actual_group_id].serverids[index_in_group]  // 标记query server

// query server 与 client轮询, 两个index同时累加, 没有真正轮询？
back_index_[actual_group_id] = ( back_id + 1 ) % back_count_;    // 实现client轮询，没有负载均衡机制
qs_index_in_group_[actual_group_id] = (index_in_group + 1) % qs_count_in_group_[actual_group_id]; // 实现query server 轮询

MTransport*   transport = &collect_clients_[server_id][back_id]   //  server_id与back_id 确定一个连接，一台cache与一台query server之间只要一条tcp连接就够了吧？


sendQueryRequest  // 其中有qps保护机制，类似令牌筒
	CollectQueryTask::send_request真正实现的发送动作
		重连机制待细化


CollectQueryTask
=============
发送request，svc中Mpoller接收response。

继承自CollectResultBase，成员：
	std::string collertor_name_;
    	Mpoller* mpoller_;
    	size_t thread_num_;
   	size_t client_count_;
    	size_t back_count_;
   	int flag_active_;
    	MTransport collect_clients_[s_max_server_cnt][s_max_back_cnt];
   	static const int s_reconnect_interval_ = 1; // 重连时间间隔(即当前发现发送失败，则1s后尝试重连)

CollectQueryTask::svc()
	deserialization()  // 初步解析返回结果，临时存储在InternalQueryResult中
		CCachePolicy::decode_queryresult()
			cache_result::decode()  // 解析到 InternalQueryResult::_query_result中
	CollectQueryTask::fillInternalQueryResult
		CCacheQueuedRequest::fillQueryResult // 最终解析到CCacheQueuedRequest::cache_content_中
	put_next  // ACE函数，传递到下级流水


CPreMergeQueryTask::svc()
	CCachePolicy::mergeQueryResultFirst（）
		CQueryMerger::Merge
			CQueryMerger::MergeFirst


CQueryMerger
============
query::DocIdResult * ids_[MaxQueryGroupCount];  // 记录doc id 结果

MergeFirst
	initMerger
	doFirstMerge
		多路结果中会有相同doc么？
		CRerankDataCenter::setMerge  // 每次merge前时，处理公用数据
		mergeResult  // 合并多路Query的结果 
			getGroupSpan  // 选取待merge的group ??
			heapSorter::push_i(hNode) // 根据 rank数值维护堆，这是个关于query group的堆
			req->setIdIndexMerged() // merge、排序后的结果放在 CCacheQueuedRequest::idindex_merged 中 
		CRerankManager::reRank()  // 预排序，前面的堆排序与此处什么关系
	WriteMergeRes2Req    //    与排序结果copy到CacheContent::idindex中，与idindex_merged 是重复操作？


CRerankDataCenter
================
setMerge //每次merge前时，处理公用数据
        


CSend2TitleSummaryTask
===================
put()
	CCachePolicy::instance()->makeTitleSummaryRequests  // 
		只有normal和fast有title结果, 仅对这两种doc请求title
		query::DocIdResult *result = get_docid_result()
		
		遍历cache_content中的docId，设置summary_request字段
		sum_Request.doc_id = result->gdocid;
		sum_Request.request_type = summary::TITLE_REQUERST_;
		sum_Request.summary_id = summary_id;  //  summary_id什么作用？
	sendTitleSummaryRequest()


CollectTitleSummaryTask  // title summary server 与summary server有何不同？返回结果一样么？
=================
svc()
	result = CCachePolicy::instance()->getInternalSummaryResult()  // 申请result存储空间
	CCachePolicy::decode_summaryresult ()  // 解析title summary结果到result中，没看到title存储在哪
	req->title_summary_result_[summary_index]  = result  // 记录到context中
	put_next(). //  传给下级流水：Send2GpuServer


CSend2GpuServerTask
=================
通过基类CSyncSendTaskBase::svc() 实现gpu request发送
CSyncSendTaskBase::svc()
	CSyncSendTaskBase::sendSyncRequest()
		makeServerRequests
			CCachePolicy::makeGpuServerRequests()
		collect_base_->sendRequest  //  collect_base_指向了CCollectGpuServerTask 类对象


CCollectGpuServerTask
==================
主要逻辑在基类CSyncCollectTaskBase中实现
open()
	指定poller_thread_num = 2; 即Mpoller中含有两个pooler
	CSyncCollectTaskBase::initTask  //  activate AEC_task 时线程数指定的 Send2GpuServerTaskThreadCount (配置的4)
 										  实际不需要多线程??  没有调用getq。AEC_task内线程调用机制？？


CSyncCollectTaskBase
==================
WebCache和GpuServer采用protobuf序列化和反序列化 
svc()
	mpoller_->get(&res, svc_get_interval);
	InternalDataResult* result = getInternalDataResult();
	CCollectGpuServerTask::fillDataResult(CCacheQueuedRequest* req, InternalDataResult* result)
		req->fillGpuServerResult()
			从cache_content->idIndex[i]得到doc，更新rank相关变量 DocIdResult::dnn_rank_5000


CMergeQueryTask
===============
svc()
	CCachePolicy::instance()->mergeQueryResult(req)
		CCachePolicy::mergeQueryResultFinal()
			merger->Merge(queue_req) 


CSend2SummaryTask
==================
put()
---
	CCachePolicy::instance()->makeSummaryRequests
	sendSummaryRequest()

Summary机群组织为分环的方式，每个Summary server保存的内容是不同的，请求不同doc摘要的请求，可能被分流到不同的Summary Server上
summary::summary_request  // request 结构定义
makeSummaryRequests_type(）//生成某类summary的请求
	groupindex = CacheOptions::instance()->onlineSummaryGroup[type];
	index_in_group = CCachePolicy::getSummaryServerID(sum_Request->doc_id, ringCount, numPerRing);
	server_id = CacheOptions::instance()->summaryServerGroups[groupindex].serverids[index_in_group];


ring的含义：表示一类summary集群，normal/fast/instant…
ring内的不同server存储正的排内容不一样。
summary_server_id的作用：确定ring中的某台服务器

struct summaryServerGroup_t  // 含义：表示一种summary，一环
{
    BYTE summarytype;               //<summarytype
    uint16_t ringCount;                  //<组内服务器数
    uint16_t numPerRing;              //<每环包含summary数目, 无意义，不用关心
    bool useStandby;               // 是否用备份
    uint16_t serverids[MaxSSCountInGroup];      //<组内summary对应的id，MaxSSCountInGroup=2048
    char groupName[32];             //<组名
};

summarytypename
{
    "NormalSummary",
    "Index1Summary",
    "InstantSummary",
    "FastSummary",
    "BizSummary",
    "SohuBlogSummary",
    "LowHumanSummary",
    "MetaSummary",
    "AllSummary",
    "HumanSummary",
    "BackupSummary",
    "QaListLongSummary",
    "QaViewSummary",
}; 


queryServerGroup_t queryServerGroups[MaxQueryGroupCount]
===============
含义：表示一个分片，按doc id划分
    BYTE updateStrategy;             //<更新策略
    uint16_t serverCount;               //<组内服务器数？？
    uint16_t serverids[MaxQSCountInGroup];      //<组内query server对应的id, 一个server id对应一台服务器
    QueryType query_type;            // normal, fast …

结构：queryServerGroups[group_id].serverids[index_in_group]  确定了


CacheOptions  // 加载config信息
=======================
loadSummaryServers  // 加载summary服务器的配置到内存中

成员变量：
summaryServerGroup_t  summaryServerGroups[MaxSummaryGroupCount];  // summary集群配置信息


CCacheDirectory  // 管理cahce_id，防止重复访问
==============================
RegisterCacheID(cacheID) 的作用：以cacheID做hash map的key，查看此cacheID是否已经查询过cahce
UnRegisterCacheID ，仅在地域相关查询时才解绑？
ReadCacheEntry() 
	CQdbAgent::instance()->LoadCacheContent()
		CCachePolicy::instance()->loadConentDbData()


CQdbAgent  // query cache模块client端
===========================
LoadCacheContent()
	CCachePolicy::instance()->loadConentDbData()

CMemcacheClient // summary cache模块client端
=====================================


CCachePolicy // 实现模块间request与response处理相关逻辑
=========================================
loadConentDbData()  
	// 解析PB结果，得到CacheEntry和CacheContent，
       // CacheContent是异步解析的，解析成功标志记录在 CCacheQueuedRequest::fconvert_content中      // 如果为地域请求，则不解析content，只解析entry，为什么？ 


CCacheQueuedRequest  // 模块间收发消息时共用的上下文
==========================================
记录各类请求的requst/result信息，提供了各请求结果的fill/get函数
    
	CacheReqPipeStat stat_;  // 当前请求处于流水的哪个部分
	CacheEntry entry_;
	CacheContent* cache_content_;

	CacheRequest  cacheRequest //存储cache查询的请求信息，存储从search_hub收到请求消息的大部分参数。
	query::QueryRequest  queryRequest //
	CacheControlRequest controlRequest //

	InternalSummaryResult * summary_result_[MAX_SUMMARY_INDEX];
   	InternalSummaryResult * title_summary_result_[MAX_SUMMARY_INDEX];
    	InternalSummaryResult * wenda_summary_result_[MAX_SUMMARY_INDEX];


cache过期（更新）判断
===================
检查每一个query_group组
	对于手动更新策略：
		version相同：
			if 配置了UpdateAccordingQuality，则按更新周期判断。
			else 不过期；
		version小：
			判断更新周期；
	对于自动更新策略：
		version小：按更新周期判断
		version大：过期
		version相同：
			判断更新周期。
			翻页情况下不进行更新	
	等价于过期的情况：请求的页面在缓存之外，或者接近缓存尾部


4种cache更新策略
==============
manual_update_ = 0,         
auto_update_ = 1,          
qc = 2,
news_update_ = 3,


Query result buffer
====================
struct maxqueryresult {
    BYTE data[MaxQueryResultSize];
};
MaxQueryResultSize = sizeof(QueryResult) + sizeof(DocIdResult)*1000;
struct QueryResult {
	DocIdResult doc[];
};


CCacheQueuedRequestManager
========================
request_board_   // 是个map的数组，记录缓存的request


HotQuery: 热搜，由算法根据doc查询频率实施计算出某个query是否属于热搜

pooler  // 不是类
=============
poller_start(poller_t *poller)
	pthread_create(&tid, NULL, __poller_thread_routine, poller);  // 一个poller对应一个线程

__poller_thread_routine (poller）
	epoll_wait(poller）
	__poller_handle_read
		__poller_add_result(node, poller);  // 结果存储到队列里（poller->params.result_queue）
	__poller_handle_listen


mpooler  // Multi Pooler，不是类
=======================
mpoller_t *mpoller_create(const struct mpoller_params *params)
	mpoller->result_queue = poller_queue_create(params->queued_results_max); // mpoller里有缓存结果的队列
	__mpoller_create(mpoller_t *mpoller)    // 创建多个pooler
		mpoller->poller[i] = poller_create(&params);

Mpoller // 类
========
封装了成员 mpoller_t *mpoller_;
get_message(void *buf, size_t *len, int timeout)


MTransport  // 底层socket封装，代表一个链接
===================





 





