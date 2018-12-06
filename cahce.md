
1.	Pipeline类
===
1.1	main()
---
WebCacheLoader.cpp中
main()
	assemblePipeline（）             // 组装pipeline
		CCachePipeline::open（）// 注册15个ACE_Module
	csvc->open() 	                    // 启动cache serverice 
 	thrman.spawn_n(threadsnumber, &loader_event_loop, NULL);   //  loader_event_loop不重要，只是等待信号，不处理请求。

1.2	CCacheService
---
open()
	配置服务端口，log server，
	连接redis服务         //  百度问问结果替换为搜狗问问，存储对应关系
qdbAgent->open()  // 初始化
directory->open()

	CCacheHandler::instance()->open()  主要的服务启动动作，接收请求

1.3	CCacheHandler
---
相当于web_cache于search_hub网络通信的server端，负责监听连接，接收request。
open()
	创建Mpoller，内含两个pooler，即两个线程 
	activate()  // 激活自己, CacheHandlerTaskThreadCount=4,  多线程会并行执行
svc，但只有一个Mpoller, Mpoller是多线程安全的吗？貌似是

svc()	// 这里不应有业务处理逻辑，尽快分发，否则会有瓶颈
	handleSuccessRequest(CCacheQueuedRequest* req, std::string* request)
		req->request.swap(*request); // 原始请求存到CCacheQueuedRequest的
request成员中。
		LocalSeachTask::put

1.4	CLocalSearchTask
---
active了10个线程，LocalSearchTaskThreadCount = 10

cacheID 是search_hub计算后传过来的
svc()
req->parseRequest() // 将search_hub的参数经过解析、包装成
// CacheRequest结构体类型的对象
	dispatchCCacheQueuedRequest // 查local cache，分发请求，是localSearch 的主要逻辑
		URLLOOKUP :  
			// summary的key是通过url计算得到的，所以直接查询summary即可
			请求summary服务。 // CSend2QueryTask::instance()->put函数中根据shouldQuery_确定是否请求summary服务
		NORMQUERY:
		SITEQUERY1:
		SITEQUERY2:
ReadCacheEntry
LoadCacheContent 
// 读cache ，异步async task存入req->fconvert_content中，并行load。倒排和正排cache存一起会更好？为什么分开cache? key都是一样的，hash_id + page。或者同时load正排cache?
if isLocationalRequest
计算地域相关newCacheID
ReadCacheEntry(newCacheID)  // 读取地域相关query cache
			query cache没命中：
				if  interOnly || cacheLevel == 1：
					直接返回空    // 特殊请求，只查缓存，减小压力。
				else 
					req->hitCache_ = HITCACHE_NONE  // hit1
					set needQuery_ , 标记请求倒排服务。
			query cache命中： // 检查访问query/summary服务的条件，设置标记
req->hitCache_ = HITCACHE_CACHEENTRY // 初始化为 hit2
命中部分query group
if CCachePolicy::isExpired // 过期，准备请求query server；
					setAllInstQueryExpired()中设置请求倒排标记, set needQuery_
					shouldSummary_ = true
				else if 结果不够，准备请求query server；
					在not_enough(req) 中设置请求倒排标记, set needQuery_
					shouldSummary_ = true
				else if is_HitAll  //在判断函数中LoadSummaryContent
					Is_HitAll逻辑：
					CCachePolicy::judge_summary()
						CQdbAgent::LoadSummaryContent
						if load成功：
							shouldSummary_ = false
hitCache_ = HITCACHE_ALL / HITCACHE_MEM;
						else
							shouldSummary_ = true
							hitCache_ = HITCACHE_DOCID // hit3
CCachePolicy::should_summary()
	if shouldSummary_ == false 
if getConvertContentResult成功 // 阻塞
			CReplyTask::put()  // hit4, hit5，直接返回。
							else 失败
			CSend2QueryTask::put() // 重试
{
	//走到这里的逻辑是不需要请求query/summary server的
return  // hit4, hit5，直接返回。
}

				// 处理hit2, hit3，hit3表示读summary cache失败了，再读一次？
				// 如果需要请求query server, 此时load summary cache无意义
				if req->shouldSummary_ // 需要请求summary server
					CQdbAgent::LoadSummaryContent
	
// 后面逻辑处理的是hit1，hit2，hit3的情况
// 是需要请求query / summary 服务的逻辑
			if shouldQuery_
				if cacheLevel == 1: 
					getConvertContentResult()  //阻塞
					CSend2SummaryTask::put() //只请求summary服务
				else 
					CSend2QueryTask::put()
			else
				if replace_url: 
req->hitCache_ = HITCACHE_DOCID;
					CPreMergeQueryTask::put()
				else
					CSend2SummaryTask::put()

CCachePolicy::instance()->getConvertContentResult(req) 作用：
(1)	先阻塞等待query content的解析；
loadConentDbData中启动的异步任务，负责解析query cache的PB结果到cache_content中。
(2)	再阻塞等待fillInternalQueryResult；
实际是解析query server返回的结果到cache_content中。（在CollectQueryTask中设置的延迟任务/同步任务）
entry的解析未阻塞。

hit1：HITCACHE_NONE，无倒排cache,  首次访问的query。
hit2：HITCACHE_CACHEENTRY，命中query cache，但是部分过期,或等效于过期
hit3：HITCACHE_DOCID，倒排cache正常，但无正排cache，一般为翻页请求
						   1 读summary cache失败时设置
                                       2 在CReplyVerifier::check中发现黑名单URL时设置
hit4：HITCACHE_ALL，  命中倒排cache、以及硬盘中正排cache。
hit5：HITCACHE_MEM，命中倒排cache、以及内存中正排cache。

1.5	CSend2QueryTask
---
back_id含义：用于确定连接query server的client，MTransport *transport = &collect_clients_[server_id][back_id]
CollectResultBase 中有记录连接query server用的类似连接池的二维数组 
MTransport collect_clients_[s_max_server_cnt][s_max_back_cnt]
s_max_back_cnt 表示每个web_cache上与query server建立连接的client数量上限

back_id = back_index_[actual_group_id]   //  标记使用的collect client
server_id = queryServerGroups[actual_group_id].serverids[index_in_group]  // 标记query server

// query server 与 client轮询, 两个index同时累加, 没有真正轮询
back_index_[actual_group_id] = ( back_id + 1 ) % back_count_;    // 实现client轮询，没有负载均衡机制
qs_index_in_group_[actual_group_id] = (index_in_group + 1) % qs_count_in_group_[actual_group_id]; // 实现query server 轮询

MTransport*   transport = &collect_clients_[server_id][back_id]   //  server_id与back_id 确定一个连接，一台cache与一台query server之间只要一条tcp连接就够了吧？

sendQueryRequest  // 其中有qps保护机制，类似令牌筒
	CollectQueryTask::send_request真正实现的发送动作
		重连机制待细化

1.6	CollectQueryTask
---
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

先将结果解压到InternalQueryResult中，再解析到cache_content中。cache_entry字段的填充分散到了后续的各级流水中
CollectQueryTask::svc()
	deserialization()  // 初步解析返回结果，存储在InternalQueryResult中
		CCachePolicy::decode_queryresult()
			cache_result::decode() //解析到InternalQueryResult::_query_result中
	fillInternalQueryResult  // deferred task记录到 req->ffill_query_result中
		req->fillQueryResult // 最终解析到 req->cache_content_中
	put_next  // ACE函数，传递到下级流水

1.7	CPreMergeQueryTask
---
svc()
	CCachePolicy:: mergeQueryResult(req)
		CCachePolicy::mergeQueryResultFirst（）
			CQueryMerger::Merge
				CQueryMerger::MergeFirst

1.8	CSend2TitleSummaryTask
---
put()
	CCachePolicy::instance()->makeTitleSummaryRequests  // 
		只有normal和fast有title结果, 仅对这两种doc请求title
		query::DocIdResult *result = get_docid_result()
		
		遍历cache_content中的docId，设置summary_request字段
		sum_Request.doc_id = result->gdocid;
		sum_Request.request_type = summary::TITLE_REQUERST_;
		sum_Request.summary_id = summary_id;  //  summary_request_id
	sendTitleSummaryRequest()

1.9	CollectTitleSummaryTask  
---
// title summary server 与summary server有何不同？返回结果一样么？
svc()
	result = CCachePolicy::instance()->getInternalSummaryResult()  // 申请result存储空间
	CCachePolicy::decode_summaryresult ()  // 解析title summary结果到result中，没看到title存储在哪
	req->title_summary_result_[summary_index]  = result  // 记录到context中
	put_next(). //  传给下级流水：Send2GpuServer

1.10	CSend2GpuServerTask
---
通过基类CSyncSendTaskBase::svc() 实现gpu request发送
CSyncSendTaskBase::svc()
	CSyncSendTaskBase::sendSyncRequest()
		makeServerRequests
			CCachePolicy::makeGpuServerRequests()
		collect_base_->sendRequest  //  collect_base_指向了CCollectGpuServerTask 类对象

1.11	CCollectGpuServerTask
---
主要逻辑在基类CSyncCollectTaskBase中实现
open()
	指定poller_thread_num = 2; 即Mpoller中含有两个pooler
	CSyncCollectTaskBase::initTask  //  activate AEC_task 时线程数指定的 Send2GpuServerTaskThreadCount (配置的4)，不需要多线程?  没有调用getq。
从GPU拿到排序特作为LTR模型的输入，拿到哪些特征？

1.12	CSyncCollectTaskBase
---
// 与GpuServer的通信采用protobuf序列化和反序列化 
svc()
	mpoller_->get(&res, svc_get_interval);
	InternalDataResult* result = getInternalDataResult();
	CCollectGpuServerTask::fillDataResult(CCacheQueuedRequest* req, InternalDataResult* result)
		req->fillGpuServerResult()
			从cache_content->idIndex[i]得到doc，更新rank相关变量 DocIdResult::dnn_rank_5000

1.13	CMergeQueryTask
---
svc()
	CCachePolicy::instance()->mergeQueryResult(req)
		CCachePolicy::mergeQueryResultFinal()
			merger->Merge(queue_req) 
	if req->shouldRequery   // 需要二查
		CCachePolicy:: resetQueuedRequest(req); // 重置req
            CSend2QueryTask::put(mb); 
	else
		put_next

1.14	CSend2SummaryTask
---
put()
	CCachePolicy::instance()->makeSummaryRequests
	sendSummaryRequest()

Summary机群组织为分环的方式，每个Summary server保存的内容是不同的，请求不同doc摘要的请求，可能被分流到不同的Summary Server上
summary::summary_request  // request 结构定义
makeSummaryRequests_type(）//生成某类summary的请求
	groupindex = CacheOptions::onlineSummaryGroup[type];
	index_in_group = CCachePolicy::getSummaryServerID(sum_Request->doc_id, ringCount, numPerRing);
	server_id = CacheOptions::summaryServerGroups[groupindex].serverids[index_in_group];


ring的含义：表示一类summary集群，normal/fast/instant…
ring内的不同server存储正的排内容不一样。
summary_server_id的作用：确定ring中的某台服务器

struct summaryServerGroup_t   // 含义：表示一种summary，一环
{
    BYTE summarytype;               //<summarytype
    uint16_t ringCount;                //<组内服务器数
    uint16_t numPerRing;             //<每环包含summary数目, 无意义，不用关心
    bool useStandby;                   // 是否用备份
    uint16_t serverids[MaxSSCountInGroup];      //<组内summary对应的id，MaxSSCountInGroup=2048
    char groupName[32];             //<组名
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

1.15	CollectSummaryTask
---
open 函数在ACE module被push进流水时调用
维护一个与summary server的连接池，包括一组MTransport 和一个Mpoller, 与Send2SummaryTask共用。

MTransport collect_clients_[s_max_server_cnt][s_max_back_cnt];  // 发送数据
Mpoller* mpoller_;    // 监听/接收

svc()
	CCachePolicy::decode_summaryresult
	将结果存储到 req->summary_result_[summary_index]  中
	if req->cacheRequest.searchtype == URLLOOKUP
		CReplyTask  // 直接返回结果
	else
		put_next       // PageMaker

1.16	CPageMaker
---
svc()
	CCachePolicy::makePage(CCacheQueuedRequest * req)
		CCacheFilter::filter

1.17	CReplyTask
svc()
	if : 首页请求，在filter_verifydaemon中过滤过多
		CSend2QueryTask::put(mb) // 进入send2Query阶段，
在过滤后就立刻判断会更好？
	下面的过滤和上面有何不同？
      if : CCachePolicy::instance()->verifyURL() == -2 // ulr黑名单过滤掉
		处理下条消息
	if ( URLLOOKUP)
		CCachePolicy::instance()->makeURLReply
		sendReplyResult(req, data);
	else
		CCachePolicy::instance()->makexmlReply
		sendReplyResult
	put_next

sendReplyResult
	pthread_rwlock_rdlock(&rwlock);  //没有写锁的地方，可以去掉此锁？
	CCacheHandler::instance()->sendResult(req->input_fd_, data, ori_len);
	pthread_rwlock_unlock(&rwlock);

1.18	CWriteBackTask
---
WriteBackTaskThreadCount=10
put
保护机制，任务队列超长时丢弃，执行CCacheQueuedRequestManager::done_request

svc()
	if 地域查询
truncateQueryResult // 截断，未细看
if req->isUpdated_     // query结果有变化，如何判定的
		WriteCacheEntry. // 更新地域无关的query cache, 
// 更新cache_id
req->entry_.cacheID = newCacheID;
CCacheDirectory::RegisterCacheID(newCacheID) 
CCacheDirectory::UnRegisterCacheID(req->cacheRequest.cacheID);
req->cacheRequest.cacheID = newCacheID;
  	if 倒正排cache全命中，hit4,hit5
		changed = false; // 不更新summary
	else
		CCachePolicy:: truncateQueryResult(req)
		if shouldSummary_ 
			CQdbAgent::UpdateSummaryContent  // 写summary cache
				CMemcacheClient::UpdateSummaryContent()
		CCachePolicy::other_update(req, db);
		CCacheDirectory::WriteCacheEntry  // 更新地域无关or相关query cache 
无条件写倒排cache, hit3也写？
写了两次cahce，内容一样，key不一样
CQdbAgent::UpdateCacheContent

2.	流水相关类
===
2.1	CCacheQueuedRequest 
---
// 模块间收发消息时共用的上下文

记录各类请求的requst/result信息，提供了各请求结果的fill/get函数
    
	CacheReqPipeStat stat_;  // 当前请求处于流水的哪个部分
	CacheEntry entry_;
	CacheContent* cache_content_; // 内含query_result

	CacheRequest  cacheRequest //存储cache查询的请求信息，存储从
search_hub收到请求消息的大部分参数。
	query::QueryRequest  queryRequest //
	CacheControlRequest controlRequest //

	InternalSummaryResult * summary_result_[MAX_SUMMARY_INDEX];
   	InternalSummaryResult * title_summary_result_[MAX_SUMMARY_INDEX];
    	InternalSummaryResult * wenda_summary_result_[MAX_SUMMARY_INDEX];

fillQueryResult() //填充cache_content中与每个group相关的字段，更新query_result

2.2	CCacheQueuedRequestManager
---
ACE_Allocator* allocator_; // 用于分配request，固定大小，为什么不用
ACE_Cached_Allocator？

request_board_   // 记录缓存的request，是个hash_map数组，大小为61，根据request_id%61，将request分配到这61个hash_map中。当收到query server的response时，用于找到对应的request

HotQuery: 热搜，由算法根据doc查询频率实施计算出某个query是否属于热搜

2.3	CacheOptions
---
// 加载config信息
loadSummaryServers  // 加载summary服务器的配置到内存中

成员变量：
summaryServerGroup_t  summaryServerGroups[MaxSummaryGroupCount];  // summary集群配置信息

2.4	CCachePolicy 
---
// 实现各级流水间request与response处理相关逻辑
std::queue<CCacheQueuedRequest*> req_queue_; // 维护一个请求池，缓存用，避免频繁申请释放

loadConentDbData()  
pb_entry->ParseFromArray  // 先解析CacheEntry
如果entry中信息显示为地域请求，且cache_id没有被更新，则返回 

      // 异步解析CacheContent，解析结果记录在fconvert_content中    
	req->fconvert_content = std::async(std::launch::async, […]() {
pb_content->ParseFromArray  // 解析CacheContent
}

2.5	CQueryMerger
---
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
	WriteMergeRes2Req    //  与排序结果copy到CacheContent::idindex中，与idindex_merged 是重复操作？

MergeFinal
	doFinalMerge(rerank_input, rerank_output, requery_doc, req)
		_rerank.reRank
		if query_from_type == QUERYFROM_WAP
			rerankQAExpertise(req);
		if 当前一查：
shouldRequery(rerank_input, rerank_output, req)
				isQueryResultGoodEnough // 判断是否需要qc or kill term二查
				isLocalizationQuery            // 分析是否需要地域二次查询
			setRequeryInput  // 保存merge结束后结果
		else 当前二查：
			setRequeryInput  // 保存merge结束后结果
			processRequery  // 应用二次查询结果，对各种二查结果排序
selectJzwdDoc(req);  // 选择精准问答的doc
执行各种排序调整策略
		addReserveTruncateDoc
	// 设置二次查询参数
if (rerank_output.requery_type != REQUERY_NULL)
		req->shouldRequery = true; //在MergeQueryTask中用于确定是否需要二查
		req->is_requery_changed = isRequeryChanged(req, rerank_output);
	makeResult. // 回填结果，填充cache_entry, cache_content
// 设置rerank过程生成的参数,填充entry_的监控、聚合相关字段
setRerankOutput(rerank_output, req);


requery_level是什么？

2.6	CCacheFilter
---
filter
	cache_content_->qc_hit_cache = qc_assess.assessQC
		getChnVec(query,queryVec);  // 获取中文字符串
		getDiffForQueryQc
	filter_baike_similar
	adjust_click_recall
	after_rerank    // 拿到summary后的后排序
	filter_bad_wenda
	adjust_ch_ofc
	adjust_result_hidden
	filter_error
	filter_verifydaemon // url检查，仅做检查，无流水逻辑更改
	filter_similar
	filter_bad_page
	filter_bad_click
	filter_more_nav
	filter_more_porn
	filter_cluster    // 聚合
	filter_site
	exchange_offical_site
	filter_url  // rerank_redirectURL

	doWebFilter
		将过滤掉的doc放到最后，插入固排
		更新web_summary_index[MaxDocIDCount]， 记录web filter之后，排序的变化情况, a[i] = j, 即第i位是idIndex中排第j的结果
		if (req->cacheRequest.begin >= DocIDCountPerPage)
			first2page[0] = CCacheQueuedRequest::fillSummaryContent() 

saveFilterDoc(req, uniqueFlag, num, "filter_url", filter_docs);
含义：uniqueFlag doc标记是否被过滤，被过滤掉的doc记录到filter_doc中

2.7	CReplyVerifier
---
check
	verifier_getstate（）// 批量校验多个url
	循环每条url结果:
		if 首页请求命中黑名单:
req->hit4Resummary = true;
req->hitCache_ = HITCACHE_DOCID;
重新进入Send2Summary
else 非首页请求命中黑名单:
重新进入Send2Query

3.	工具类
===
3.1	CCacheDirectory 
 // 管理cahce_id，防止重复访问
---
RegisterCacheID(cacheID) 的作用：以cacheID做hash map的key，查看此cacheID是否已经查询过cahce
UnRegisterCacheID ，仅在地域相关查询时才解绑？
ReadCacheEntry() 
	CQdbAgent::instance()->LoadCacheContent()
		CCachePolicy::instance()->loadConentDbData()

3.2	CQdbAgent 
// query cache模块client端
---
LoadCacheContent()
	CCachePolicy::instance()->loadConentDbData()

3.3	CMemcacheClient 
// summary cache模块client端
---

3.4	Query result buffer
---
struct maxqueryresult {
    BYTE data[MaxQueryResultSize];
};
MaxQueryResultSize = sizeof(QueryResult) + sizeof(DocIdResult)*1000;
struct QueryResult {
	DocIdResult doc[];
};
3.5	pooler
---
每个pooler对应一个epool监听线程，可以监听多个socket，并缓存消息。
poller_add(const struct poller_data *data, poller_t *poller)
poller监听data中指定的socket及事件类型

poller_start(poller_t *poller)
	pthread_create(&tid, NULL, __poller_thread_routine, poller);  // 一个poller对应一个线程

__poller_thread_routine (poller）
	epoll_wait(poller）
	__poller_handle_read
		__poller_add_result(node, poller);  // 结果存储到队列里（poller->params.result_queue）
	__poller_handle_listen

3.6	mpooler 
---
// Multi Pooler，不是类

mpoller_t *mpoller_create(const struct mpoller_params *params)
	mpoller->result_queue = poller_queue_create(params->queued_results_max); // mpoller里有缓存结果的队列
	__mpoller_create(mpoller_t *mpoller)    // 创建多个pooler
		mpoller->poller[i] = poller_create(&params);

3.7	Mpoller
---
封装了成员 mpoller_t *mpoller_;
get_message(void *buf, size_t *len, int timeout)

3.8	MTransport 
---
// 底层socket封装，代表一个链接

4.	关键数据
===
4.1	CacheEntry
---
描述了某个查询串对应的最终结果（排序、过滤后的结果）在Cache中缓存的情况。
4.2	CacheContent
---
用于缓存各路Query返回的结果，以及排序、滤重后的DocList
4.3	SummaryContent
---
用于缓存每个页面中各条Doc的摘要及其他相关信息。
4.4	Query_request
描述了Cache发给Query Server的请求

4.5	QueryResult
描述了从Query Server返回的结果

4.6	summary_request
描述了Cache发给Summary的请求。从直观的理解，所谓的Summary请求，就是拿这Query结果Doclist中docid到Summary中找到相应的网页，然后根据关键词，利用网页的内容，生成摘要的标题和摘要的内容。

4.7	summary_result
此结构描述了Summary返回的结果

4.8	cache_id
---
fillCacheIDByString()，从search_hub请求中hash字段解析得到。
hash —> cache id

4.9	cache_key
---
getKeyByCacheID()

query cache key: c + cache_id
Summary cache key:  s + cache_id + 页数

query server 查询的key ?
Summary server查询的key : doc id

4.10	request_id
---
标识一条request, 用于从respons查找到对应的CCacheQueuedRequest

4.11	summary_id
---
summary_id = request_id : summary_type : summary_index
承载了request_id (bit 33~64) ，summary_type (bit 17～32) 与summary_index (bit 1~16）

4.12	summary_index 
---
summary 在summary_result_中的存储下标
req->summary_result_[summary_index] 

4.13	queryServerGroup_t 
---
queryServerGroups[MaxQueryGroupCount]
含义：表示一个分片，按doc id划分
    BYTE updateStrategy;             //<更新策略
    uint16_t serverCount;               //<组内服务器数？？
    uint16_t serverids[MaxQSCountInGroup];      //<组内query server对应的id, 一个server id对应一台服务器
    QueryType query_type;            // normal, fast …

结构：queryServerGroups[group_id].serverids[index_in_group]  确定了
5.	功能方案
===
5.1	分页机制
search_hub request中有请求的页数，web cache会从doc序列中截取一部分
5.2	并发
每级流水都是多线程的任务，不同流水级别实现了不同的并发数。
5.3	二查机制
CQueryMerger::shouldRequery中判断。在merge后判断是否需要做二查。
优先级：qc > 去词 > 地域 > 时效
二查类型会设置到 rerank_output.requery_type 中

5.3.1	qc/去词二查
isQueryResultGoodEnough(rerank_input, rerank_output) // 分析返回结果，判断是否需要qc or kill term

5.3.2	地域二查
// 二次查询类型
REQUERY_NULL = 0;   // 无二次查询
REQUERY_QC = 1;     // qc二次
REQUERY_KILL = 2;   // 去词二次
REQUERY_REGION = 3; // 地域二次
REQUERY_TIME = 4;   // 时效性二次

地域查询词，比如：**家园，市实验二小，等隐含地域性强的查询词
在merge后判断是否需要做二查，其中包括地域二查的判断：
判断地域二查：isLocalizationQuery（）
计算出查询词的地域等级，（根据词表和summary计算，得到一个level值）
如果是地域查询，且未发生任何其他二查，则设置地域二查标记，REQUERY_REGION
 if (REQUERY_NULL == rerank_output.requery_type)
 {
      rerank_output.requery_type = REQUERY_REGION;
      return localization_level;
 }

5.4	Cache更新判断isToUpdate

5.5	cache过期(更新)判断
---
bool CCachePolicy::isExpired(CCacheQueuedRequest *req, 
int now, 
bool needQuery[], 
int reason[])

检查每组 req->entry_.updateInfo：(其中记录了过期策略及上次更新时间)
	对于手动更新策略：(manual_update_)
		if version相同：
			if 配置了UpdateAccordingQuality，
判断更新周期。
			else 
不过期；
			记录quality_group
		else if version小：
			判断更新周期；
			记录version_group
	对于自动更新策略：(auto_update_  || news_update_)
		If version小：
按更新周期判断
			记录time_group
		else if version大：
过期 ( version error)
		else if version相同，但距上次更新时间 > 3600：
			判断更新周期。
		else
			翻页情况不算过期
			判断过期时间，过期
记录time_group
			else
				计算过期概率，记录almost_expired
If manual_update_模式下过期：
	isOnlyInstantExpired = false
else if auto_uopdate 模式下过期：
	isOnlyInstantExpired = true
if 有没过期的auto_update group
	req->isInstantAllExpired = false;
else 
  	req->isInstantAllExpired = true;
等价于过期的情况：请求的页面在缓存之外，或者接近缓存尾部


// 4种cache更新策略
manual_update_ = 0,         
auto_update_ = 1,          
qc = 2,  // 什么意思
news_update_ = 3,

5.6	地域性查询
实现查询的地域相关性，即同一个query在不同地域查询，得到的结果不一样。
查cache逻辑：
先用cache_id查query cahe，在查出query cache后，判断是地域相关，则生成地域相关cache_id。后续用地域相关cache_id去生成cache key，查query cache和summary cache。
写cache逻辑：
Merge后会由rerank_output带出此次查询是否是地域性查询，并记录到req->entry_.isRegionalResult上。如果是地域性查询，重新计算cache_id。WriteBack时，使用原始cache_id写query cache，使用地域相关cache_id写query cache和summary cahce。即最终存在两种query cache 和 summary cache（一是地域无关cache，二是地域相关cache）
6.	问题
===
1 unregister_timeout_handler 什么意思
2 ReconnectManager
3 isOnlyInstantExpired

4 search_hub的请求是并行处理的么？在哪体现的？
5 固排有一个单独的query server, 在哪？
7 memdb memcache cache机制
9 hot_query_client ?
10 getSummaryFetchLevel   fetch level 什么意思？
11 服务器规格：cpu 内存
12 q1, q2什么意思？
        //标示是否为q1(对于cache实验流量，假设先到来的为q1,后到来的为q2.即未在unordered_map命中的为q1)
    bool is_first_exp_query_;

13 聚合逻辑
14 CCachePolicy::isToUpdate 逻辑
15 Summary server的备份机制？

13 含义：
enum SearchType{
    ERROR,
    NORMQUERY,      // 普通查询
    URLLOOKUP,      // Url查询
    SITEQUERY1,     // 带查询词的site查询
    SITEQUERY2,     // 不带查询词的site查询
    CACHECONTROL,       // Cache的控制
    //--link查询相关类型
    LINKLOOKUP,     // link查询
    SITELINKLOOKUP,     // sitelink查询
    RELATEDLOOKUP,      // 相似网页查询
    QUICKSHARE      //为quickshare项目单独接口
};

7.	优化点
===
1 PB arena
2 RPC框架
3 服务注册与发现
4 内存管理，tcmalloc，jemalloc（facebook推出的）
    在多线程环境使用tcmalloc和jemalloc效果非常明显。
    当线程数量固定，不会频繁创建退出的时候， 可以使用jemalloc；反之使用tcmalloc可能是更好的选择。
5 数据性能分析可视化, Grafana. Kibana. es
6 每个流水任务至少是一个线程，需要读队列的开销，直接调用会更高效
7 使用内存池代替固定size的内存分配，目前cache_content用的malloc分配，CacheQueuedRequest用的ACE_Allocator分配。






 






