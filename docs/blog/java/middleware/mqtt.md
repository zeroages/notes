# MQTT在线连接数API

1. 获取 API 认证凭证 
访问地址: http://ip:18083/#/api-key
在 EMQX Dashboard 中创建应用以获取认证凭证：


2. 接口文档
访问地址: http://ip:18083/api-docs/index.html#/


1. Java
* Maven 依赖配置
```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
```
* 示例代码

```java
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import java.io.IOException;
import java.util.Base64;
public class EmqxRestApiTest {
    
    /**
     * 主方法 - 获取客户端连接信息
     */
    public static void main(String[] args) {
        // 配置参数
        String apiKey = "key";
        String apiSecret = "Secret";
        String clientId = "clientId";
        String emqxUrl = "http://ip:18083/api/v5/clients/" + clientId;
        // 创建 HTTP 客户端
        OkHttpClient client = new OkHttpClient();
        // 创建 Basic Auth 认证头
        String credentials = apiKey + ":" + apiSecret;
        String authHeader = "Basic " + Base64.getEncoder().encodeToString(credentials.getBytes());
        // 构建 HTTP 请求
        Request request = new Request.Builder()
                .url(emqxUrl)
                .header("Authorization", authHeader)
                .build();
        try {
            // 执行请求
            Response response = client.newCall(request).execute();
            
            // 检查响应状态
            if (!response.isSuccessful()) {
                System.err.println("请求失败: " + response.code() + " - " + response.message());
                return;
            }
            // 获取响应体
            String responseBody = response.body() != null ? response.body().string() : "";
            System.out.println("API 响应: " + responseBody);
            // 解析连接状态
            if (responseBody.contains("\"connected\":true")) {
                System.out.println("✅ 客户端 " + clientId + " 在线");
            } else {
                System.out.println("❌ 客户端 " + clientId + " 离线");
            }
        } catch (IOException e) {
            System.err.println("请求异常: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```
*  输出结果示例
```json
{
  "auth_expire_at": null,
  "mqueue_dropped": 0,
  "send_msg.qos1": 651,
  "proto_ver": 4,
  "recv_msg.qos0": 0,
  "subscriptions_max": "infinity",
  "awaiting_rel_max": 100,
  "send_pkt": 736,
  "mailbox_len": 1,
  "clientid": "32",
  "clean_start": false,
  "awaiting_rel_cnt": 0,
  "recv_pkt": 736,
  "send_msg.dropped.expired": 0,
  "username": "shihuikj",
  "expiry_interval": 7200,
  "client_attrs": {},
  "inflight_cnt": 0,
  "ip_address": "ip",
  "send_msg.dropped.too_large": 0,
  "recv_msg.dropped": 0,
  "peersni": null,
  "recv_cnt": 728,
  "recv_msg.qos1": 16,
  "send_oct": 43572,
  "send_msg.dropped.queue_full": 0,
  "heap_size": 610,
  "reductions": 804998,
  "durable": false,
  "recv_msg.qos2": 0,
  "enable_authn": true,
  "mqueue_len": 0,
  "proto_name": "MQTT",
  "node": "emqx@127.0.0.1",
  "is_expired": false,
  "send_msg": 651,
  "is_persistent": true,
  "subscriptions_cnt": 1,
  "send_cnt": 734,
  "listener": "tcp:default",
  "send_msg.qos2": 0,
  "connected_at": "2025-08-25T10:38:45.675+08:00",
  "inflight_max": 32,
  "port": 56386,
  "recv_msg": 16,
  "recv_oct": 4162,
  "mqueue_max": 1000,
  "is_bridge": false,
  "connected": true,
  "send_msg.qos0": 0,
  "recv_msg.dropped.await_pubrel_timeout": 0,
  "created_at": "2025-08-25T06:00:44.077+08:00",
  "keepalive": 60,
  "send_msg.dropped": 0,
  "mountpoint": null
}
```
* 关键参数说明

🔍 核心连接参数

| 参数名 | 类型 | 说明 | 示例值 |
|--------|------|------|--------|
| ip_address | string | 客户端的 IP 地址 | "ip" |
| connected | boolean | 客户端连接状态 | true (在线) |
| keepalive | integer | 心跳间隔时间(秒) | 60 |
| clientid | string | 客户端连接 ID | "32" |
| username | string | 客户端用户名 | "shihuikj" |

📊 连接详情参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| connected_at | string | 连接建立时间 |
| created_at | string | 客户端创建时间 |
| port | integer | 客户端连接端口号 |
| node | string | 所在的 EMQX 节点名称 |
| proto_ver | integer | MQTT 协议版本 |
| clean_start | boolean | 是否清理会话 |

* 相关 API 接口

📡 客户端管理接口

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | /api/v5/clients | 获取所有客户端列表 |
| GET | /api/v5/clients/{clientid} | 获取指定客户端信息 |
| DELETE | /api/v5/clients/{clientid} | 踢除指定客户端 |
| GET | /api/v5/nodes/{node}/clients | 获取指定节点的客户端列表 |

🔧 其他相关接口

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | /api/v5/nodes | 获取集群节点信息 |
| GET | /api/v5/metrics | 获取系统监控指标 |
| POST | /api/v5/publish | 发布消息到主题 |