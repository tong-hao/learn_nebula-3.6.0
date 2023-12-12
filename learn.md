
# 代码逻辑

## src/kvstore 
- 主要被 `src/storage` 使用
- `src/kvstore/listener` 主要作为 learn 角色。
- `src/kvstore/raftex` 主要实现了raft协议
- `src/kvstore/wal` 

整体流程：
StorageServer->NebulaStore->RaftPart

RocksEngine
KVEngine
KVStore



## src/storage 
- http 主要是http接口
- admin 是admin相关的Processor，如：ReIndex,GetLeader,ClearSpace
- exec 里面定义了各种Node，被其他Processor调用
- index 只有一个 LookupProcessor
- mutate 点和边相关的Processor
- query 查询相关的Processor
- transaction 应该是为了一致性做的

storaged 启动流程：
```cpp
src/daemons/StorageDaemon.cpp
 StorageServer::start()
  MetaClient
  StorageServer::getStoreInstance()
   NebulaStore::init()
    RaftexService::createService
  StorageServer::getStorageServer()
   GraphStorageServiceHandler

```

添加顶点流程：
```cpp
GraphStorageServiceHandler::future_addVertices
 AddVerticesProcessor::process
  AddVerticesProcessor::doProcess
   // 调用下面两个函数转成kv格式
   NebulaKeyUtils::tagKey
   encodeRowVal
   doPut
    NebulaStore::asyncMultiPut
     Part::asyncMultiPut
      // 走raft协议
      RaftPart::appendAsync 
```


## src/parser
- 下面定义了各种 Sentence
- 会在`parse.yy`中调用 Sentence

```cpp
Validator::makeValidator
 ShowHostsValidator
  ShowHostsSentence
```

## src/graph

主要子目录：
- validator
- planner
- scheduler
- executor
- optimizer

graphd 启动流程：
```cpp
GraphDeamon.cpp
 GraphServer::start()
```

gql执行的整体流程：
GraphService->QueryEngine->QueryInstance->GQLParser->xxxValidator->xxxPlanner->Scheduler->xxxExecutor

`insert vertex` gql的执行流程：
```cpp
GraphService::future_execute
 GraphService::future_executeWithParameter
  QueryEngine::execute
   QueryInstance::execute
    QueryInstance::validateAndOptimize()
     // gql经过parse后的结果是各种Sentence，也就是Tree
     GQLParser(qctx()).parse(rctx->query())
     Validator::validate(各种Sentence)
      // 每个Sentence会对应一个Validator
      InsertVerticesValidator::validateImpl()
      // 生成 Planner
      Validator::toPlan()
       Planner::toPlan
        SequentialPlanner::transform
    // 优化器
    QueryInstance::findBestPlan()
     Optimizer::findBestPlan
    AsyncMsgNotifyBasedScheduler::schedule()
     Executor::create
     // BFS
     Executor::doSchedule
     AsyncMsgNotifyBasedScheduler::scheduleExecutor
      runSelect
      runLoop
      runExecutor
       InsertVerticesExecutor::execute()
        StorageClient::addVertices
```

## 疑惑
- RocksEngine、KVEngine、KVStore 之间的关系
