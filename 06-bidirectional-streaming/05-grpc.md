# gRPC ストリーミング

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/grpc/

## 目次

- [概要](#概要)
- [gRPC の基本](#grpc-の基本)
- [Protocol Buffers 定義](#protocol-buffers-定義)
- [サーバー実装](#サーバー実装)
- [クライアント実装](#クライアント実装)
- [双方向ストリーミング](#双方向ストリーミング)

---

## 概要

gRPC は Google が開発した高性能な RPC フレームワーク。HTTP/2 ベースで、効率的なバイナリシリアライゼーション（Protocol Buffers）を使用。

### gRPC の利点

- **高パフォーマンス**: バイナリプロトコルと HTTP/2 による効率
- **型安全**: Protocol Buffers による厳密な型定義
- **双方向ストリーミング**: ネイティブサポート
- **言語間互換性**: 多言語クライアント生成

## gRPC の基本

### ストリーミングパターン

1. **Unary**: 単一リクエスト → 単一レスポンス
2. **Server Streaming**: 単一リクエスト → 複数レスポンス
3. **Client Streaming**: 複数リクエスト → 単一レスポンス
4. **Bidirectional Streaming**: 複数リクエスト ↔ 複数レスポンス

## Protocol Buffers 定義

### agent.proto

```protobuf
syntax = "proto3";

package agent;

// Agent サービス定義
service AgentService {
    // 単方向ストリーミング
    rpc StreamChat(ChatRequest) returns (stream ChatResponse);
    
    // 双方向ストリーミング
    rpc BidirectionalChat(stream ChatRequest) returns (stream ChatResponse);
}

// リクエストメッセージ
message ChatRequest {
    string content = 1;
    map<string, string> metadata = 2;
}

// レスポンスメッセージ
message ChatResponse {
    oneof response {
        ContentChunk content = 1;
        ToolUse tool_use = 2;
        ToolResult tool_result = 3;
        Error error = 4;
    }
    string event_id = 5;
}

message ContentChunk {
    string text = 1;
}

message ToolUse {
    string tool_name = 1;
    string tool_id = 2;
    string input = 3;
}

message ToolResult {
    string tool_id = 1;
    string output = 2;
}

message Error {
    string code = 1;
    string message = 2;
}
```

### コード生成

```bash
# Python
python -m grpc_tools.protoc \
    -I. \
    --python_out=. \
    --grpc_python_out=. \
    agent.proto

# Go
protoc --go_out=. --go-grpc_out=. agent.proto
```

## サーバー実装

### Python サーバー

```python
import grpc
from concurrent import futures
import agent_pb2
import agent_pb2_grpc
from strands import Agent
import asyncio

class AgentServicer(agent_pb2_grpc.AgentServiceServicer):
    def __init__(self):
        self.agent = Agent(
            system_prompt="あなたは親切なアシスタントです。"
        )
    
    def StreamChat(self, request, context):
        """単方向ストリーミング"""
        # Agent を実行
        for event in self.agent.stream(request.content):
            if "data" in event:
                response = agent_pb2.ChatResponse(
                    content=agent_pb2.ContentChunk(text=event["data"])
                )
                yield response
            
            if "tool_use" in event:
                tool_use = event["tool_use"]
                response = agent_pb2.ChatResponse(
                    tool_use=agent_pb2.ToolUse(
                        tool_name=tool_use["name"],
                        tool_id=tool_use["id"],
                        input=str(tool_use["input"])
                    )
                )
                yield response
    
    def BidirectionalChat(self, request_iterator, context):
        """双方向ストリーミング"""
        for request in request_iterator:
            # 各リクエストに対して Agent を実行
            for event in self.agent.stream(request.content):
                if "data" in event:
                    yield agent_pb2.ChatResponse(
                        content=agent_pb2.ContentChunk(text=event["data"])
                    )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    agent_pb2_grpc.add_AgentServiceServicer_to_server(
        AgentServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    print("サーバーを起動しました（ポート 50051）")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### 非同期サーバー

```python
import grpc.aio
import agent_pb2
import agent_pb2_grpc
from strands import Agent
import asyncio

class AsyncAgentServicer(agent_pb2_grpc.AgentServiceServicer):
    def __init__(self):
        self.agent = Agent(
            system_prompt="あなたは親切なアシスタントです。"
        )
    
    async def StreamChat(self, request, context):
        """非同期単方向ストリーミング"""
        async for event in self.agent.stream_async(request.content):
            if "data" in event:
                yield agent_pb2.ChatResponse(
                    content=agent_pb2.ContentChunk(text=event["data"])
                )
    
    async def BidirectionalChat(self, request_iterator, context):
        """非同期双方向ストリーミング"""
        async for request in request_iterator:
            async for event in self.agent.stream_async(request.content):
                if "data" in event:
                    yield agent_pb2.ChatResponse(
                        content=agent_pb2.ContentChunk(text=event["data"])
                    )

async def serve():
    server = grpc.aio.server()
    agent_pb2_grpc.add_AgentServiceServicer_to_server(
        AsyncAgentServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    await server.start()
    print("非同期サーバーを起動しました（ポート 50051）")
    await server.wait_for_termination()

if __name__ == '__main__':
    asyncio.run(serve())
```

## クライアント実装

### Python クライアント

```python
import grpc
import agent_pb2
import agent_pb2_grpc

def stream_chat(prompt: str):
    """単方向ストリーミングクライアント"""
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = agent_pb2_grpc.AgentServiceStub(channel)
        
        request = agent_pb2.ChatRequest(content=prompt)
        
        for response in stub.StreamChat(request):
            if response.HasField('content'):
                print(response.content.text, end='', flush=True)
            elif response.HasField('tool_use'):
                print(f"\n[Tool: {response.tool_use.tool_name}]")
        
        print()

# 使用例
stream_chat("こんにちは、今日の天気を教えて")
```

### 非同期クライアント

```python
import grpc.aio
import agent_pb2
import agent_pb2_grpc
import asyncio

async def async_stream_chat(prompt: str):
    """非同期ストリーミングクライアント"""
    async with grpc.aio.insecure_channel('localhost:50051') as channel:
        stub = agent_pb2_grpc.AgentServiceStub(channel)
        
        request = agent_pb2.ChatRequest(content=prompt)
        
        async for response in stub.StreamChat(request):
            if response.HasField('content'):
                print(response.content.text, end='', flush=True)
        
        print()

asyncio.run(async_stream_chat("こんにちは"))
```

## 双方向ストリーミング

### クライアント実装

```python
import grpc.aio
import agent_pb2
import agent_pb2_grpc
import asyncio

async def bidirectional_chat():
    """双方向ストリーミングクライアント"""
    async with grpc.aio.insecure_channel('localhost:50051') as channel:
        stub = agent_pb2_grpc.AgentServiceStub(channel)
        
        # リクエストを生成するジェネレータ
        async def request_generator():
            while True:
                user_input = await asyncio.get_event_loop().run_in_executor(
                    None, input, "You: "
                )
                
                if user_input.lower() == 'quit':
                    break
                
                yield agent_pb2.ChatRequest(content=user_input)
        
        # 双方向ストリーミング
        responses = stub.BidirectionalChat(request_generator())
        
        print("Agent: ", end='', flush=True)
        async for response in responses:
            if response.HasField('content'):
                print(response.content.text, end='', flush=True)

asyncio.run(bidirectional_chat())
```

### 並列処理

```python
import grpc.aio
import agent_pb2
import agent_pb2_grpc
import asyncio

async def parallel_bidirectional():
    """並列双方向ストリーミング"""
    async with grpc.aio.insecure_channel('localhost:50051') as channel:
        stub = agent_pb2_grpc.AgentServiceStub(channel)
        
        request_queue = asyncio.Queue()
        
        async def request_generator():
            while True:
                request = await request_queue.get()
                if request is None:
                    break
                yield request
        
        async def send_messages():
            while True:
                user_input = await asyncio.get_event_loop().run_in_executor(
                    None, input, "You: "
                )
                
                if user_input.lower() == 'quit':
                    await request_queue.put(None)
                    break
                
                await request_queue.put(
                    agent_pb2.ChatRequest(content=user_input)
                )
        
        async def receive_responses(stream):
            async for response in stream:
                if response.HasField('content'):
                    print(f"\rAgent: {response.content.text}", end='', flush=True)
            print()
        
        stream = stub.BidirectionalChat(request_generator())
        
        await asyncio.gather(
            send_messages(),
            receive_responses(stream)
        )

asyncio.run(parallel_bidirectional())
```

### エラーハンドリング

```python
import grpc

async def stream_with_error_handling(prompt: str):
    try:
        async with grpc.aio.insecure_channel('localhost:50051') as channel:
            stub = agent_pb2_grpc.AgentServiceStub(channel)
            
            request = agent_pb2.ChatRequest(content=prompt)
            
            async for response in stub.StreamChat(request):
                if response.HasField('error'):
                    raise Exception(f"Server error: {response.error.message}")
                
                if response.HasField('content'):
                    print(response.content.text, end='')
    
    except grpc.RpcError as e:
        if e.code() == grpc.StatusCode.UNAVAILABLE:
            print("サーバーに接続できません")
        elif e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
            print("タイムアウトしました")
        else:
            print(f"gRPC エラー: {e.code()} - {e.details()}")
```
