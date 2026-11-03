一、Agent项目深挖
	
1.项目解决什么业务问题，用户Query到结果返回的完整链路：解决的是业务人员不熟悉表结构和口径、难将自然语言问题转为可靠 SQL、并需手工分析汇总的问题；数据链路是用户选择数据库并提问，前端通过 A2A 流式发送；后端依次完成知识与表列召回、关系扩展、可行性评估、计划及人工确认，再由 Supervisor 循环调度 SQL 执行和可选 Python 分析，最终流式返回 Markdown 报告，并实时展示各节点结果。

2.多Agent入口和最终输出谁负责，任务路由用规则还是模型决策。
- **多 Agent 入口**：`A2AController` 接收 `message/stream`，`GraphAgentExecutor` 创建任务、初始化 State 并启动 Graph。
- **最终输出**：`ReportGeneratorNode` 负责生成 Markdown 报告；`GraphAgentExecutor` 将其转为 A2A Artifact，经 SSE 返回前端展示。
- **任务路由**：混合决策。Supervisor 每轮由 LLM 决定 SQL、Python、Report 或 END，属于模型决策；可行性评估、 SupervisorEdge 根据状态字段规则分流，并设 12 轮熔断。
- ![[Pasted image 20261103031216.png]]
	
3.一个Query多需求怎么拆，串行并行怎么选；Plan和Execute各管什么，简单任务要不要规划。
按“子目标”拆，而不是按句子拆。每个子任务应明确：目标、输入、工具、依赖、预期输出。严格串行。
目前没有简单任务快路径
	
4.简单任务为何不直接用ReAct；Multi-Agent优势、额外成本；为何不做一个Agent配多个Skill。
	
5.Agent间上下文怎么隔离、怎么通信，传完整对话还是结构化结果，怎么控长防重复。没有隔离，没有控长防重复
	
6.执行失败、工具选错、反复调同一工具怎么处理；任务完成怎么判断、是否校验；Loop终止条件和防死循环；换底层模型要改哪些。
	
二、RAG
	
1.文档上传到最终检索的完整流程。
	
2.切片怎么设计，Chunk Size和Overlap怎么定。
	
3.检索前是否Query Rewrite，怎么防改写偏离原意。
	
4.BM25和Dense各自解决什么，为什么混合检索。
	
5.召回后为什么上Reranker，怎么判断检索效果。
	
三、后端工程
	
1.Agent流式输出怎么实现，为什么选SSE。
	
2.用户取消或断网，怎么停后台任务。
	
3.多Tool同时跑，Python怎么异步并发；FastAPI里阻塞操作有啥影响。
	
4.长任务的长连接和状态怎么处理；Session和对话状态存哪，并发怎么防冲突。
	
5.外部模型超时失败怎么重试降级；服务怎么做限流、并发控制和权限校验。
	
四、手撕代码
	
1.链表倒数第K个节点：双指针，O(n)、O(1)。
	
2.合并两个有序数组：双指针，类似LeetCode 88。