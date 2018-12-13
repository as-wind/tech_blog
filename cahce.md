
1.	web cache相关服务
search_hub
qo：search_hub请求的服务，query优化
qa：search_hub请求的服务,query意图识别
qc：web_cache请求的服务。query纠错

2.	Pipeline类
===
2.1	main()
---
WebCacheLoader.cpp中
main()
	assemblePipeline（）             // 组装pipeline
		CCachePipeline::open（）// 注册15个ACE_Module
	csvc->open() 	                    // 启动cache serverice 
 	thrman.spawn_n(threadsnumber, &loader_event_loop, NULL);   //  loader_event_loop不重要，只是等待信号，不处理请求。

2.2	CCacheService
---
无线程，普通类
open()
	配置服务端口，log server，
	连接redis服务         //  百度问问结果替换为搜狗问问，存储对应关系
qdbAgent->open()  // 初始化
directory->open()

	CCacheHandler::instance()->open()  主要的服务启动动作，接收请求

2.3	CCacheHandler
---
ACE task类，4线程
相当于web_cache于search_hub网络通信的server端，负责监听连接，接收request。
open()
	创建Mpoller // 内含两个pooler，即两个线程 
	activate()  // 激活自己, CacheHandlerTaskThreadCount=4,  多线程会并行执行
svc，但只有一个Mpoller, Mpoller是多线程安全的吗？貌似是

svc()	// 这里不应有业务处理逻辑，尽快分发，否则会有瓶颈
	handleSuccessRequest(CCacheQueuedRequest* req, std::string* request)
		req->request.swap(*request); // 原始请求存到CCacheQueuedRequest的
request成员中。
		enqueue_request（）// 登记到 CacheQueuedRequestManager:: 
request_board_ 上
		LocalSeachTask::put

2.4	CLocalSearchTask
---
ACE task类，10线程，LocalSearchTaskThreadCount = 10

cacheID 是search_hub计算后传过来的
svc()
req->parseRequest() // 将search_hub的参数经过解析，包装成
// CacheRequest结构体类型的对象，
存到req-> cacheRequest中
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

2.5	CSend2QueryTask
---
普通类，非task

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

CCachePolicy:: makeQueryRequests(req, requests)
If (updateStrategy == qc)
	构造qc request // 没有协议定义，直接硬编码的
	Else
		构造query request
sendQueryRequest（）  // 其中有qps保护机制，类似令牌筒
循环所有request
If (updateStrategy == qc)
getQCRequestServerIdx // 得到server_id
	CollectQueryTask::send_request // 真正实现的发送动作
		重连机制待细化

同时请求多种query server，包括qc server，
2.6	MCollectQueryTask
---
ACE_Task类，6线程（硬编码）
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
deserialization()  // 初步解析返回结果，临时存储，为什么不直接解析到
cache_content_->query_results中？
		CCachePolicy::decode_queryresult()
			cache_result::decode() //解析到InternalQueryResult::_query_result中
	fillInternalQueryResult  // deferred task记录到 req->ffill_query_result中
if (flag == QC_FLAG) 
// 存储到cache_content_->qc_string中
CCachePolicy::instance()->parseQCdata 
else if (isNewsTrendQueryGroup)
if isToUpdate()
		req->fillHotResult
else if (isNewClickQueryGroup)
if isToUpdate()
		req->fillClickResult
else
if isToUpdate()
				req->fillQueryResult  // 最终解析到 req->cache_content_中
	put_next  // ACE函数，传递到下级流水

CCachePolicy::isToUpdate (CCacheQueuedRequest*, InternalQueryResult *)
得到query result后在fillInternalQueryResult中执行此函数，检查缓存的req->cache内容是否需要更新，需要则设置req->isUpdated_字段。
手动更新模式：则判断cache_content中的doc_num、版本等信息；
自动更新模式：则判断doc_num、版本、crc检验码。
更新req->cache_entry中的超时周期(仅影响本次查询)，
原则是：cache内容有更新，缩短超时周期（以2/3倍递减），
反之增大超时周期（以3/2倍递增）
在CCachePolicy::isExpired中会比较更新后的超时周期，判断cache是否超时，但是isExpired仅在LocalSearch阶段被调用。对于本次查询，req->entry中的超时周期不再用了？

2.7	CPreMergeQueryTask
---
ACE task，10线程
svc()
	CCachePolicy:: mergeQueryResult(req)
		CCachePolicy::mergeQueryResultFirst（）
			CQueryMerger::Merge
				CQueryMerger::MergeFirst

merge、排序后的结果在cache_content->idIndex中，
Send2TitleSummary使用的req->cache_content_.query_results[]

（1）	通过堆排序对多路query server返回的结果做合并、排序、去重。
保留前2000个doc。
堆排序作用：减小reRank的负担
（2）	重复的非固排doc按summary_type优先级决定是否组合放入新query组
（3）	通过_rerank.reRank做排序，后面请求summary title时只用了前1000个，这里rerank可以优化，不排序，仅挑选出前1000个优质doc
reRank的作用：减少请求summary title、请求GPU server、精排的负担

2.8	CSend2TitleSummaryTask
---
普通类，非task
由summary server提供title 查询服务，web cache与每台 summury server有1个连接
这个连接是多余的，CSend2SummaryTask中已经有4个连接了 

put()
	CCachePolicy::makeTitleSummaryRequests  // 
		min(MaxDnnCnnCalcNum, cache_content_->docNum);
 		最多请求1000个doc，MaxDnnCnnCalcNum = 1000;

		循环取出cache_content_->idIndex[i]：//遍历cache_content中的docId，
设置summary_request字
		{
		query::DocIdResult *result = get_docid_result()
		
		只有normal和fast有title结果
从query server中拿到title结果的，不取title

		sum_Request .doc_id = result->gdocid;
		sum_Request .request_type = summary::TITLE_REQUERST_;
		sum_Request .summary_id = summary_id; 
makeSummaryRequests_type（）
}
	sendTitleSummaryRequest()

2.9	CollectTitleSummaryTask  
---
ACE task，4线程
title summary server 与summary server有何不同？返回结果一样么？
title是summary生成的？根据summary_request中的query_string生成的？
为什么通过GetUtfSummaryResultOtherExtra()获得title?

svc()
	result = CCachePolicy::getInternalSummaryResult()  // 申请result存储空间
	CCachePolicy::decode_summaryresult ()  // 解析title summary结果到result中，与summary的解析重复了？
	req->title_summary_result_[summary_index]  = result  // 记录到context中
	put_next(). //  传给下级流水：Send2GpuServer

2.10	CSend2GpuServerTask
---
ACE task，4线程
与GpuServer的通信采用protobuf序列化和反序列化 

通过基类CSyncSendTaskBase::svc() 实现gpu request发送
CSyncSendTaskBase::svc()
	CSyncSendTaskBase::sendSyncRequest()
		makeServerRequests
			CCachePolicy::makeGpuServerRequests()
getFeaturesDnnCnn() // 4线程并行计算
	          			getTitleWordFromQsAddSite或
getTitleWordFromSummaryAddSite 
		collect_base_->sendRequest  //  collect_base_指向了
CCollectGpuServerTask 类对象

2.11	CCollectGpuServerTask
---
ACE task，4线程，mpoller两个线程，
gpu server: 79台     // 每个cache维护79个client，分别连接每一台server
connectionPerGpuServer：16 // 对每个gpu server，每个client维护16个连接
（16个MTransport）

主要逻辑在基类CSyncCollectTaskBase中实现
open()
	指定poller_thread_num = 2; 即Mpoller中含有两个pooler
	CSyncCollectTaskBase::initTask  // 创建clinets, 为每个client创建tansport_list

从GPU拿到排序特征，作为LTR模型的输入，类似doc中的dnn_rank_5000等参数
req->idx_mapping是做什么的？结果记录到了idx_mapping确定的doc中一份

2.12	CMergeQueryTask
---
ACE task，10线程

svc()
	CCachePolicy::instance()->mergeQueryResult(req)
		CCachePolicy::mergeQueryResultFinal()
			merger->Merge(queue_req)  // shouldRequery会被设置
	if req->shouldRequery   // 需要二查
		CCachePolicy:: resetQueuedRequest(req);  // 重置req
            CSend2QueryTask::put(mb); 
	else
		put_next

2.13	CSend2SummaryTask
---
普通类，无独立线程
每台web cache与每台summary server有4个连接

put()
	CCachePolicy::makeSummaryRequests  // 
		// 请求多少doc?
		If URLLOOKUP
			makeURLSummaryRequests()
				makeSummaryRequests_type()
		else 
makeSummaryRequests_i()
	makeSummaryRequests_type()
req->filter_->makeExtraSummaryRequests();
makeSummaryRequests_qalistlong(req, data, indexes, types);
makeSummaryRequests_qaview(req, data, indexes, types);
makeSummaryRequests_qashort(req, data, indexes, types);
	sendSummaryRequest()

Summary机群组织为分环的方式，每个Summary server保存的内容是不同的，请求不同doc摘要的请求，可能被分流到不同的Summary Server上
summary::summary_request  // request 结构定义

makeSummaryRequests_type(）//生成某类summary的请求
	groupindex = CacheOptions::onlineSummaryGroup[type];
	index_in_group = CCachePolicy::getSummaryServerID(sum_Request->doc_id, ringCount, numPerRing);
	server_id = CacheOptions::summaryServerGroups[groupindex].serverids[index_in_group];


ring的含义：表示一类summary集群，normal/fast/instant…
ring内的不同server存储的正排内容不一样。
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
    "BackupSummary",  // 没配置
    "QaListLongSummary",
    "QaViewSummary",
}; 

2.14	CollectSummaryTask
---
ACE task，4线程，
open 函数在ACE module被push进流水时调用
维护一个与summary server的连接池，包括一个Mpoller和一组client，clinet与Send2SummaryTask共用。

MTransport collect_clients_[s_max_server_cnt][s_max_back_cnt];  // 发送数据
Mpoller* mpoller_;    // 监听/接收

svc()
	CCachePolicy::decode_summaryresult
	将结果存储到 req->summary_result_[summary_index]  中
	if req->cacheRequest.searchtype == URLLOOKUP
		CReplyTask  // 直接返回结果
	else
		put_next       // PageMaker

2.15	CPageMaker
---
ACE task，6线程

svc()
	CCachePolicy::makePage(CCacheQueuedRequest * req)
if (req->filter_->need_filter)
        		req->filter_->filter(req);
				doWebFilter() // 填充过滤结果，判断是否需要resummary
		req->fillSummaryContent()
first2page ？？
			req->fillSummaryContent(sumContent, begin, end, false, true)
				按cache_content_->web_summary_index中记录的顺序
取docid，填充sumContent
			initial_summary_content_[i] = sumContent; //没用了
			summary_content_[i] = sumContent;
qalistlong_summary_content_ = sumContent;
qaview_summary_content_ = sumContent;
qashort_summary_content_ = sumContent;
		req->fillExtraSummaryContent()  
	if req->shouldResummary
		重新请求summary
	else 
		put_next
2.16	CReplyTask
---
ACE task，4线程

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
		compressResult
		sendReplyResult
	put_next

sendReplyResult
	pthread_rwlock_rdlock(&rwlock);  //没有写锁的地方，可以去掉此锁？
	CCacheHandler::instance()->sendResult(req->input_fd_, data, ori_len);
	pthread_rwlock_unlock(&rwlock);

2.17	CWriteBackTask
---
AEC task，10线程，WriteBackTaskThreadCount=10
put
保护机制，任务队列超长时丢弃，执行CCacheQueuedRequestManager::done_request

svc()
	if 地域查询
truncateQueryResult // 截断，未细看
if req->isUpdated_     // query结果有变化，如何判定的
		WriteCacheEntry. // 更新地域无关的query cache, 
// 以下更新cache_id
req->entry_.cacheID = newCacheID;
CCacheDirectory::RegisterCacheID(newCacheID) 
CCacheDirectory::UnRegisterCacheID(req->cacheRequest.cacheID);
req->cacheRequest.cacheID = newCacheID;
  	if 倒正排cache全命中，hit4,hit5
		changed = false; // 不更新summary
	else
		CCachePolicy:: truncateQueryResult(req) // 截断
		if shouldSummary_ 
			CQdbAgent::UpdateSummaryContent  // 写summary cache
				CMemcacheClient::UpdateSummaryContent() // memcache
		CCachePolicy::other_update(req, db);

		// 更新地域无关or相关query cache 无条件写倒排cache, 
hit3也写? 写了两次cahce，内容一样，key不一样
CCacheDirectory::WriteCacheEntry
CQdbAgent::UpdateCacheContent
		createContentDbData()  // 序列化cache entry和content
				CQdbClient::WriteContentToDB    // memdb
				如果写query cache失败，则删除summary cache，
保证一致性，即summary cache依赖于query cache

3.	流水相关类
===
3.1	CCacheQueuedRequest 
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

IdIndex idx_mapping[MaxMergeMappingSize]; // 什么作用？
IdIndex idindex_merged[2 * MaxDocIDCount]; // 一查Merge后的IdIndex
IdIndex idindex_merged_requery[2 * MaxDocIDCount]; //二查Merge后的IdIndex
IdIndex idindex_reset_rerank[MaxDocIDCount];  //ResetRerank后的IdIndex


CCachePolicy维护了 req_queue_队列，起到内存池的作用，队列size 50，此对象从队列上分配，对象调用了clear做初始化 
 // 相关函数CCachePolicy::getCCacheQueuedRequest（）


fillQueryResult() //填充cache_content中与每个group相关的字段，更新query_result
//此函数的执行保证在convert_content执行后才执行，//cache_content互斥吧

3.2	CCacheQueuedRequestManager
---
ACE_Allocator* allocator_; // 用于分配request，固定大小，为什么不用
ACE_Cached_Allocator？

request_board_   // 缓存接收到的request，是个hash_map数组，大小为61，根据request_id%61，将request指针分配到这61个hash_map中。当收到query server的response时，用于找到对应的request

HotQuery: 热搜，由算法根据doc查询频率实施计算出某个query是否属于热搜

3.3	CacheOptions
---
// 加载config信息
loadSummaryServers  // 加载summary服务器的配置到内存中

成员变量：
summaryServerGroup_t  summaryServerGroups[MaxSummaryGroupCount];  // summary集群配置信息

3.4	CCachePolicy 
---
实现各级流水间request与response处理相关逻辑

std::queue<CCacheQueuedRequest*> req_queue_; // 维护一个请求池，缓存用，初始化时批量new出50个。避免频繁申请释放
CacheHandler收到请求后会通过getCCacheQueuedRequest()从队列上pop一个node

loadConentDbData()  
pb_entry->ParseFromArray  // 先解析CacheEntry
如果entry中信息显示为地域请求，且cache_id没有被更新，则返回 

      // 异步解析CacheContent，解析结果记录在fconvert_content中    
	req->fconvert_content = std::async(std::launch::async, […]() {
pb_content->ParseFromArray  // 解析CacheContent
}

3.5	CQueryMerger
---
query::DocIdResult * ids_[MaxQueryGroupCount];  // 二维数组，
记录所有query组的docid结果
IdIndex* idindex_; // 记录排序结果

MergeFirst
	if 一查：
initMerger（）// 初始化rerank_input结构，待排序的数据
      else 二查：
		resetMerger() // 重置rerank_input结构
	doFirstMerge
		多路结果中会有相同doc么？
		_pdc->setMerge  // 初始化CRerankDataCenter的成员变量
		mergeResult  // 合并多路Query的结果，去重
			getGroupSpan  // 根据一查/二查类型，选取待merge的group
			heapSorter::push_i(hNode) // 根据 rank数值(query_server给的排序？)
维护一个关于doc id的堆，排序、去重
		req->setIdIndexMerged() // 将merge、排序后的结果idindex_，copy一份到
req->idindex_merged 中，什么作用？ 
		_ rerank::reRank()  // 对idindex_预排序，前面的堆排序与此处什么关系
	WriteMergeRes2Req    //  排序结果copy到CacheContent::idindex中，
idindex记录了doc的排序顺序

MergeFinal
	doFinalMerge(rerank_input, rerank_output, requery_doc, req)
		_rerank.reRank() 
		if query_from_type == QUERYFROM_WAP
			rerankQAExpertise(req);
		if 当前一查：
shouldRequery(rerank_input, rerank_output, req)
				isQueryResultGoodEnough() // 判断是否需要qc or kill term二查
				isLocalizationQuery()            // 分析是否需要地域二次查询
			{
setRequeryInput  // 保存merge结果到req->requery_doc_中,无效
}
		else 当前二查：
			setRequeryInput  // 保存merge结果到req->requery_doc_中，无效？
			processRequery  // 应用二次查询结果，对各种二查结果排序
				qcRequeryRank or
				killTermRequeryRank or
				localizationRequeryRank or
				timeRequeryRank
selectJzwdDoc(req);  // 选择精准问答的doc
执行各种调整、打压、人工干预策略（本地站点提前、低质量打压…）
		addReserveTruncateDoc
	// 设置二次查询参数
if (rerank_output.requery_type != REQUERY_NULL)
		req->shouldRequery = true; //在MergeQueryTask中用于确定是否需要二查
		req->is_requery_changed = isRequeryChanged(req, rerank_output);
	makeResult. // 回填结果，填充req->cache_entry, req->cache_content
// 设置rerank过程生成的参数,填充entry_的监控、聚合相关字段
setRerankOutput(rerank_output, req);

requery_level是什么？

3.6	CCacheFilter
---
filter
	cache_content_->qc_hit_cache = qc_assess. assessQC
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
	filter_similar // 相似结果过滤
	filter_bad_page
	filter_bad_click
	filter_more_nav
	filter_more_porn
	filter_cluster    // 聚合（只首页展示聚合），新闻、问答，
	filter_site
	exchange_offical_site
	filter_url  // rerank_redirectURL

	doWebFilter
		if 过滤后结果够多
			将过滤掉的doc放到最后，插入固排.
填充cache_content_->web_summary_index
// filter之后，最终排序情况记录在此字段中，
排序的变化情况, a[i] = j, 即第i位是idIndex中排第j的结果
			if (req->cacheRequest. begin >= DocIDCountPerPage)
				first2page[0] = CCacheQueuedRequest::fillSummaryContent() 
		else
			req->shouldResummary = true;

saveFilterDoc(req, uniqueFlag, num, "filter_url", filter_docs);
含义：uniqueFlag doc标记是否被过滤，被过滤掉的doc记录到filter_doc中

3.7	CReplyVerifier
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

4.	工具类
===
4.1	CCacheDirectory 
 // 管理cahce_id，防止重复访问
---
RegisterCacheID(cacheID) 的作用：以cacheID做hash map的key，查看此cacheID是否已经查询过cahce
UnRegisterCacheID ，仅在地域相关查询时才解绑？
ReadCacheEntry() 
	CQdbAgent::instance()->LoadCacheContent()
		CCachePolicy::instance()->loadConentDbData()

4.2	CQdbAgent 
// query cache模块client端
---
LoadCacheContent()
	CCachePolicy::instance()->loadConentDbData()

4.3	CMemcacheClient 
// summary cache模块client端
---

4.4	Query result buffer
---
struct maxqueryresult {
    BYTE data[MaxQueryResultSize];
};
MaxQueryResultSize = sizeof(QueryResult) + sizeof(DocIdResult)*1000;
struct QueryResult {
	DocIdResult doc[];
};

4.5	CSyncCollectTaskBase
---
// 与GpuServer的通信采用protobuf序列化和反序列化 
svc()
	mpoller_->get(&res, svc_get_interval);
	InternalDataResult* result = getInternalDataResult();
      // 将临时结果解析到req中，具体逻辑由子类实现
	fillDataResult(CCacheQueuedRequest* req, InternalDataResult* result) 
		req->fillGpuServerResult()
			从cache_content->idIndex[i]得到doc，更新rank相关变量 DocIdResult::dnn_rank_5000

4.6	pooler
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

4.7	mpooler 
---
// Multi Pooler，不是类

mpoller_t *mpoller_create(const struct mpoller_params *params)
	mpoller->result_queue = poller_queue_create(params->queued_results_max); // mpoller里有缓存结果的队列
	__mpoller_create(mpoller_t *mpoller)    // 创建多个pooler
		mpoller->poller[i] = poller_create(&params);

4.8	Mpoller
---
封装了成员 mpoller_t *mpoller_;
get_message(void *buf, size_t *len, int timeout)

4.9	MTransport 
---
// 底层socket封装，代表一个链接

5.	关键数据
===
5.1	CacheRequest
---
记录web cache接收到的查询请求, 从search_hub收到请求消息被解析后，大部分参数存储于此结构中。
5.2	cache_request
---
web cache请求本地cache数据库的消息结构
5.3	cache_result
---
本地cache数据库返回的结果
主要字段：
    uint16_t doc_size;
    uint16_t doc_num;
    uint16_t group_id;
    union {
        uint16_t info_asn;
        struct
        {
            uint8_t more_result : 1;
            uint8_t invalid : 1;
        };
    };
    uint32_t request_id;
    uint32_t total_items;
    uint32_t crc_32;
    asn1::array_length_t title_terms_count;
    doc_title_terms title_terms[query::MAX_TITLE_COUNT];
    cache_result_doc doc[MaxDocsInResult]; // doc结果信息

5.4	CacheEntry
---
对应于query entry的读写，与PbCacheEntry结构对应。

描述了某个查询串对应的最终结果（排序、过滤后的结果）在Cache中缓存的情况。

summaryImage // 记录summary db 中是否存储了某页的summary数据，有责读db，否则不读，提高性能
initial_summary_content_[i] = sumContent; //没用了
5.5	CacheContent
---
对应于query cache的读写结构，与PbCacheContent结构对应，达到对query cache的读写。

缓存各路Query返回的结果，以及排序、滤重后的DocList

Query相关：
    gchar_t query_string[MaxQueryStringLength];
    BYTE terms_count;
    TermRange term_range[MaxTermsCount];
    uint8_t term_weights[MaxTermsCount];
Qc相关：
    gchar_t qc_query_string[MaxQueryStringLength];      
    BYTE qc_terms_count;                                
    TermRange qc_term_range[MaxTermsCount];
    uint8_t qc_term_weights[MaxTermsCount];
    gchar_t qc_string[3][MaxQueryStringLength]; // qc串
doc id相关：
    uint16_t docNum; // 总doc id条数
    uint32_t totalItems;
    IdIndex idIndex[2*MaxDocIDCount]; // 记录merge rank后的排序情况
uint16_t web_summary_index[MaxDocIDCount]; //记录web filter之后
排序的变化情况, a[i] = j, 即第i位是idIndex中排第j的结果
uint16_t qq_summary_index[MaxDocIDCount]; // 记录qq filter之后，
排序的变化情况
    InteractiveEntry interactive_entry[MAX_INTERACTIVE_ENTRY_NUM];
    int  KeywordValue[2*DocIDCountPerPage];
    uint16_t doc_num[MaxQueryGroupCount]; //每组query server的doc数
    uint32_t total_items[MaxQueryGroupCount];
query原始结果：
char query_results[];     //<存储所有query的结果,包括
QueryResult,final_rank,sim_page_count

5.6	QueryRequest
---
描述了Cache发给Query Server的请求


5.7	QueryResult
---
描述了从Query Server返回的结果
    struct QueryResult : public QueryResultBase
    {
        uint16_t doc_size;
        uint16_t doc_num;
        uint8_t group_id;     // query类型
        uint8_t more_result : 1;
        uint8_t invalid : 1;
        uint8_t : 0;
        uint32_t request_id;  // unique id from cache
        uint32_t total_items; // 对于这个查询最多能返回的结果数(滤重前)
        uint16_t annotation_terms_count;
        uint16_t _unused;
        TermRange term_range[MaxTermsCount];
        uint32_t crc_32;
        uint32_t vertical_attr;
        uint8_t term_weights[MaxTermsCount];
        uint128_t needQueryBit;
        int title_terms_count;
        doc_title_terms title_terms[MAX_TITLE_COUNT]; doc title信息
        int news_doc_rank_info_num;
        rank_t news_doc_keep_rank;
        t_DocRankInfo news_doc_rank_info[MAX_NEWS_RANK_INFO_NUM];
        DocIdResult doc[]; // 依次列出结果
    };

5.8	summary_request
---
描述了Cache发给Summary的请求。从直观的理解，所谓的Summary请求，就是拿这Query结果Doclist中docid到Summary中找到相应的网页，然后根据关键词，利用网页的内容，生成摘要的标题和摘要的内容。

5.9	summary_result
---
此结构描述了Summary返回的结果
5.10	SummaryContent
---
用于缓存页面中各条Doc的摘要及其他相关信息。
对应summary cache的内容，写summary cache时将此结构写入memcache

struct SummaryContent {
    unsigned int size;
    BYTE itemCount;  
    BYTE summaryType[DocIDCountPerPage];
    uint32_t notValid;

    gchar_t query_string[MaxQueryStringLength];     
    BYTE terms_count;                               
    TermRange term_range[MaxTermsCount];
    
    gchar_t qc_query_string[MaxQueryStringLength];      
    BYTE qc_terms_count;                                
    TermRange qc_term_range[MaxTermsCount];

    int interactive_num;
    time_t makeTime;

    gchar_t qc_string[3][MaxQueryStringLength]; // qc串
    int  KeywordValue[2*DocIDCountPerPage];

    query::DocIdResult docid[DocIDCountPerPage];
    rank_t final_rank[DocIDCountPerPage];
    uint16_t sim_page_count[DocIDCountPerPage];
    bool from_requery[DocIDCountPerPage];
    char doc_debug_info[DocIDCountPerPage][MAX_DEBUG_INFO_LENGTH]; // doc的debug信息
    t_ClickData click_data[MAX_CLICK_DATA_NUM]; // 保存点击数据
    t_FilterDoc filter_doc[MAX_FILTER_DEBUG_DOC_NUM]; // 保存在filter阶段被过滤的doc
    /*新闻时效性数据*/
    uint32_t newsTrendDocNum_1[NewsDayScope]; // hotQuery新闻热点的统计情况
    uint32_t newsTrendDocNum_2[NewsDayScope*2]; // fast/norm天级新闻的统计情况
    uint32_t newsTrendDocNum_3[NewsHourScope]; // instant小时级新闻的统计情况
    uint32_t newsTrendDocNum_4[NewsWeekScope]; // fast/norm周级新闻的统计情况
    int      newsTrendFlag_1;
    int      newsTrendFlag_2;
    int      newsTrendFlag_3;
    int      newsTrendFlag_4;
    int      newsTrendFlag;
    int      newsVrPos;
    int summaryOffset[DocIDCountPerPage];
    char summaryData[0]; 
};

5.11	cache_key
---
getKeyByCacheID()

query cache key: c + cache_id
Summary cache key:  s + cache_id + 页数

query server 查询的key ?
Summary server查询的key : doc id

5.12	各种id
5.12.1	cache_id
fillCacheIDByString()，从search_hub请求中hash字段解析得到。
hash —> cache id

5.12.2	request_id
标识一条request, 用于从respons查找到对应的CCacheQueuedRequest

5.12.3	summary_id
summary_id = request_id : summary_type : summary_index
承载了request_id (bit 33~64) ，summary_type (bit 17～32) 与summary_index (bit 1~16）

5.12.4	summary_index 
summary 在summary_result_中的存储下标
req->summary_result_[summary_index] 

5.13	t_rerank_output
struct t_rerank_output // rerank输出的各种信息，供其它使用
{
    uint32_t requery_type; // null, qc, kill term, loc , etc
    BYTE requery_level;
    int qcTag; // qc使用策略
    bool shouldHintRequery;
    short localization_level; // 地域查询强度级别
    bool is_potential_province_query;
    bool isRegionalResult;
    bool shouldTimeRequery;
    uint32_t fromtime;
    uint8_t oldResultNum;
    float time_requery_ltr_threshold; // 用于判断二查结果的相关性
    uint16_t time_requery_time_threshold;
    bool badTimeResults;
    uint8_t requery_used; // 是否使用requery结果
    uint16_t docnum0; // 一次结果数

    uint8_t click_rerank_stat; // 点击反馈使用情况
    uint8_t click_model_stat; // 点击模型使用情况
    uint8_t click_ltr_stat; // 点击LTR使用情况

    bool is_forbid_click;
    
    uint32_t ltrMd5;
    uint32_t ltrMd5Requery;
    uint32_t ltrVersion;

    // 聚合相关
    int news_vr_position;
    int newest_news_index;
    int total_find_news;
    int first_find_news;
    bool is_news_strict;
    int news_relevance_level;
    int first_wenda_position;
    int first_wenwen_wenda_position;
    int first_wenku_position;
    int first_scholar_position;
    int first_bbs_position;
    int first_standard_position;
    int first_medical_position;
    int second_medical_position;
    int zhidao_num;
    int wenwen_num;
    int mingyi_vr_total;
    int mingyi_vr_wenda;
    hostID_t cluster_bbs_site;
    int first_soft_position;
    int medical_cluster_type;
    int med_intent_level;

5.14	queryServerGroup_t 
---
queryServerGroups[MaxQueryGroupCount]

含义：表示一个分片，按doc id划分
结构：queryServerGroups[group_id].serverids[index_in_group]  确定了

对应配置文件中配置的服务集群
struct queryServerGroup_t
{
    BYTE updateStrategy;               // <更新策略
    uint16_t serverCount;               // <组内服务器数
uint16_t serverids[MaxQSCountInGroup];      //<组内query对应的id，
一个server id对应一台服务器
    char groupName[32];                  //<组名
    bool isClickQueryGroup;              //<点击信息Query组
    bool isNewClickQueryGroup;        //<新点击信息Query组
    bool isNewsTrendQueryGroup;     //<新闻趋势Query组
    bool stillUseTCP;
    QueryType query_type; 			 // normal, fast …
    bool isQaViewQueryGroup;            //QaView Query组
    bool isQaExpertiseQueryGroup;
};

更新策略
enum updateStrategyType
{
    manual_update_ = 0,         
    auto_update_ = 1,          
    qc = 2,
    news_update_ = 3,
};

QueryServers：
Normal：21组(group1~group21)，每组24台，共504台
	"updateStrategy"="manual"
Fast：12组（group22~group33），每组42台，共504台
	"updateStrategy"="manual"
Inst：4组 (group34~group37)，每组12台，共台48
"updateStrategy"="auto"
Manual:	1组（group38），2台. // 什么意思
	"updateStrategy"="manual"  
ClickQueryGroup: 1组（group39），2台
	"updateStrategy"="manual"
Qc:	1组（group40），7台
"updateStrategy"="qc"
NewsTrendQueryGroup:	1组（group41），4台
"updateStrategy"="auto"
Inst(News): 2组（group42~group43）,每组24台，共48台
"updateStrategy"="auto"
NewClickQueryGroup：1组（group44），5台
"updateStrategy"="auto"
QaViewQueryGroup：group45，2台 // 立知
"updateStrategy"="manual"
BaikeQueryGroup：group46，4台
"updateStrategy"="manual"
QaExpertiseQueryGroup：group47，2台
"updateStrategy"="manual"

5.15	summaryServerGroup_t
struct summaryServerGroup_t 
{
    BYTE summarytype;             // 正排类型，最大值13，
    uint16_t ringCount;              // 环是怎么划分的？按内容？环之间内容不同？
    uint16_t numPerRing;           // 环内服务器数 
    bool useStandby;
    uint16_t serverids[MaxSSCountInGroup];      //<组内summary对应的id
    char groupName[32];            //组名
};

SummaryServers:
NormalSummary	1008台，主
FastSummary		1008台，主
InstantSummary	18台，主
"ringCount"=dword:0000001 // ring内的数据都是一样的？
"numPerRing"=dword:00000018
QaViewSummary	4台，主备
HumanSummary	2台，主备
MetaSummary		24台，主

6.	功能方案
===
6.1	分页机制
---
search_hub request中有请求的页数，web cache会从doc序列中截取一部分
6.2	并发
---
每级流水都是多线程的任务，不同流水级别实现了不同的并发数。
6.3	二查机制
---
CQueryMerger::shouldRequery中判断。在merge后判断是否需要做二查。
优先级：qc > 去词 > 地域 > 时效，只做一种二查。
二查类型会设置到 rerank_output .requery_type 中

CQueryMerger::shouldRequery
	if isQueryResultGoodEnough(…)    // 判断是否需要qc or kill term二查
else if isLocalizationQuery()            // 分析是否需要地域二查
else if rerank_output. shouldTimeRequery  // 判断是否做实效二查，
有时效性的query，需要做二查，并按实效排序。在merge阶段的
CRerankManager::reRank()函数中，依次调用rerank策略时实现的。

6.3.1	QC/去词二查
isQueryResultGoodEnough(rerank_input, rerank_output)
 // 分析返回结果，判断是否需要qc or kill term
	if rerank_input. qc_level >= 0 && rerank_input. qc_string非空，
requery_type = REQUERY_QC
	else 根docNum, top_title_rank均值等判断 
requery_type = REQUERY_KILL

parseQCdata() 会解析qc server的返回结果，qc_level == 0表示无法做qc
struct t_qc_data1
{
    uint32_t request_id; // 对应请求的Id
    size_t group_id;     // qc server的组
    int qc_count;
    int qc_len;
    char qc_result[0]; // 变长字符串,t_qc_data2 结构
};
struct t_qc_data2
{
    int qc_level; // qc级别
    int freq[2];
};

6.3.2	地域二查
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

6.4	Query cache过期判断
bool CCachePolicy::isExpired(CCacheQueuedRequest *req, 
int now, 
bool needQuery[], 
int reason[])
作用：读出query cache后判断内容是否过期，过期则需要请求query server

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


struct queryServerGroup_t
{
    BYTE updateStrategy;               //<更新策略
    uint16_t serverCount;               //<组内服务器数
    uint16_t serverids[MaxQSCountInGroup];      //<组内query对应的id
    char groupName[32];             //<组名
    bool isClickQueryGroup;             //<点击信息Query组
    bool isNewClickQueryGroup;          //<新点击信息Query组
    bool isNewsTrendQueryGroup;         //<新闻趋势Query组
    bool stillUseTCP;
    QueryType query_type;
    bool isQaViewQueryGroup;            //QaView Query组
    bool isQaExpertiseQueryGroup;
};

// queryServerGroup_t. updateStrategy 定义了query server 4种更新策略
manual_update_ = 0,         
auto_update_ = 1,          
qc = 2,  // 什么意思，表明是qc server?
news_update_ = 3,

6.5	地域性查询
---
实现查询的地域相关性，即同一个query在不同地域查询，得到的结果不一样。
查cache逻辑：
先用cache_id查query cahe，在查出query cache后，判断是地域相关，则生成地域相关cache_id。后续用地域相关cache_id去生成cache key，查query cache和summary cache。
写cache逻辑：
Merge后会由rerank_output带出此次查询是否是地域性查询，并记录到req->entry_.isRegionalResult上。如果是地域性查询，重新计算cache_id。WriteBack时，使用原始cache_id写query cache，使用地域相关cache_id写query cache和summary cahce。即最终存在两种query cache 和 summary cache（一是地域无关cache，二是地域相关cache）
6.6	分流机制
---
1.	为了提高cache命中率，前端按照查询串的散列码把请求分发到一台cache上，确保相同查询肯定落到同一台cache机器。与用户无关。
2.	

6.7	服务器部署
6个机房：cache-web-djt，cache-web-djt1，cache-web-gd，cache-web-gd1，
cache-web-js，cache-web-tc1，
  		   6个机房，每个机房45台，共270台
机器配置：32c, 128g，
状态：cpu，日内波动，峰值在27%左右
内存，稳定在98G左右，其中cache 34G，memdb 32G，memcached 24G

SiteVerify：	2台
WapSiteVerify：2台
GpuServer:		79台
WsGpuServer:	20台
QaShortServer:	60台
RedisServer：	1台
7.	问题
===
1 unregister_timeout_handler 什么意思
2 ReconnectManager
3 isOnlyInstantExpired
4 shouldRequestQaShort
4 GetHotQueryResult

5 
    SITEQUERY1,     // 带查询词的site查询
SITEQUERY2,     // 不带查询词的site查询

6 query server也返回title了？ QueryServer会回传DocList中前20条的title信息??
   doc_title_terms
5 固排有一个单独的query server, 在哪？

6 cache_click_request ？
9 hot_query_client ?
10 getSummaryFetchLevel   fetch level 什么意思？控制每页请求数量？
12 q1, q2什么意思？
        //标示是否为q1(对于cache实验流量，假设先到来的为q1,后到来的为q2.即未在unordered_map命中的为q1)
    bool is_first_exp_query_;

13 聚合逻辑
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

8.	优化点
===
1 PB arena，createContentDbData中pb的new/delete 耗时, 已经用了jemalloc
   makeGpuServerRequests中的PB delete
2 InternalSummaryResult构造函数中使用了malloc，可用内存池优化
2 cache content 压缩解压缩耗时，ZSTD_compress
2 RPC框架
3 服务注册与发现
5 数据性能分析可视化, Grafana. Kibana. es
6 每个流水任务至少是一个线程，需要读队列的开销，直接调用会更高效
7 使用内存池代替固定size的内存分配，目前cache_content用的malloc分配，CacheQueuedRequest用的ACE_Allocator分配。
8 过滤器按效率排序：效率 = 过滤掉数量 / 总耗时

