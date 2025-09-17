# autoStubSDK
udp通信的自动化桩

## 自动化桩背景
应用于自动化框架，可以通过服务实例的启动模拟其他服务对象，还可以通过脚本调用的方式模拟依赖服务的请求和返回行为，并且可以把结果同步到用例自动化case当中；同时兼容桩和mock的能力，并且能进行自动化脚本的转化，还可以减轻开发可测试性保障的成本。

价值：
• 可以满足服务端测试用例自动化脚本100%的转换
    ◦ 依赖服务返回异常状态码
    ◦ 依赖服务返回异常返回体
    ◦ 依赖服务返回超时
    ◦ 依赖服务异常下线
    ◦ 多实例容灾场景
    ◦ 负载均衡场景模拟
• 可以满足更细粒度测试场景的模拟方式
• 可以满足被测服务的可测试性
• 轻量级的应用，不需要开发介入
• 可以满足各类型协议的mock场景
• 可以在单个testCase单元上控制mock（直接定义桩返回协议、桩启动、桩下线、桩返回）

## 自动化桩交互设计
<img width="3126" height="5172" alt="image" src="https://github.com/user-attachments/assets/f3bc89f6-f39b-43bc-aefe-723157ee0d67" />


## 自动桩的使用方法
基于httpStubOperator模块提供的方法使用桩
1. 使用前需要进行实例化
```python
from stubSDK.httpStubOperator import StubOperator

file_app = StubOperator(stub_port=8771)
# stub_port: 定义桩服务启动时的端口号
```
2. 启动桩
```python
file_app.start_stub(channel_recv_timeout=2)
# channel_recv_timeout: 桩消息接收超时时间的定义，如果桩消息接收超时会回调500状态码给被测服务，选填字段，默认值为2
```
3. 运行测试用例，如果是请求被测服务的http接口
使用到http异步请求方法
```python
    hc = HttpCommon()
    hc.http_requests(url=url, method='post', headers=headers, json=body)
```
4. 桩接到被测服务的请求后会打包成消息包返回给python客户端，可以使用桩实例化后的receive方法接收消息
data = file_app.receive()接收消息的协议：
```json
{
    "path": "str",
    "method": "str",
    "headers": "dict",
    "cookies": "dict",
    "params": "dict",
    "data": "json str"
}
```
可以接受消息时进行被测服务请求三方服务的协议校验：
```python
        data = file_app.receive()
        self.assertIn('file_id', data['params'].keys())
        self.assertEqual('110', data['params']['file_id'])
```
5. 定义桩回调给被测服务的消息

```python
{
    "response": 定义返回体,
    "status_code": 定义返回状态码
}
response_data = {
            "response": {'msg': 'success'},
            "status_code": 403
        }
        file_app.send(response_data)
```
testCase使用桩完整代码的demo：
```python
import unittest
from common.outputLog import info, class_case_decoration
import requests
from common.checkCommon import json_output_check
from stubSDK.httpStubOperator import StubOperator
from stubSDK.httpCommon import HttpCommon


@class_case_decoration
class DocteamEditHandle(unittest.TestCase):
    def setUp(self) -> None:
        self.file_app = StubOperator(stub_port=8771)
        self.file_app.start_stub()

    def testCase01(self):
        """用户编辑云文档，请求编辑接口，依赖服务返回403状态码"""
        url = 'http://127.0.0.1:8530/edit'
        headers = {
            'Content-Type': 'application/json',
            'Cookie': 'user_id=userB'
        }
        body = {
            "file_id": "110",
            "status": "edit"
        }
        hc = HttpCommon()
        hc.http_requests(url=url, method='post', headers=headers, json=body)

        data = self.file_app.receive()
        self.assertIn('file_id', data['params'].keys())
        self.assertEqual('110', data['params']['file_id'])

        response_data = {
            "response": {'msg': 'success'},
            "status_code": 403
        }
        self.file_app.send(response_data)

        hc.http_handle_end_check()
        self.assertEqual(501, hc.status_code)
        self.assertEqual('SERVER ERROR', hc.res_json['msg'])

    def tearDown(self) -> None:
        self.file_app.stub_stop()
```

自动化桩的拓展
• 支持其他通信协议的桩，如果三方服务是grpc协议，需要额外实现grpc服务实例的ServerStub
• 桩其他能力的实现，比方说桩返回超时、桩实例重启
