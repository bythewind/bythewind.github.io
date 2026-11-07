---
title: 排查跨国服务访问产生的无权限问题
description: 一次 Feign 跨国调用从超时、无权限到 Header 中文内容被截取导致 JSON 解析失败的排查过程。
date: 2026-11-07 10:00:00 +0800
categories:
  - 服务治理
tags:
  - Feign
  - Gateway
  - Spring Cloud
  - HTTP Header
  - 故障排查
status: 已解决
---

## 现象：跨国调用先表现为超时

国外服务通过 Feign 调取国内服务时，接口没有返回明确的业务错误，而是在等待一段时间后超时。最初很容易把方向放在网络延迟、连接池或者国内服务处理慢上。

但这类跨地域调用不能只用“距离远”解释。请求已经进入 Feign 客户端，却长期没有拿到有效响应，说明需要同时检查三层问题：

1. 国外服务实际请求的是哪个地址；
2. 请求是否经过允许访问国内服务的入口；
3. 请求有没有携带该入口要求的认证信息。

继续追踪调用链后，根因逐渐明确：国外服务没有直接访问国内服务的权限。此时的超时只是表面现象，并不能证明国内业务接口执行缓慢。

## 第一处修复：改为从网关进入

原来的 Feign 调用没有走预期的网关入口，而是直接指向国内服务。这样既绕过了统一鉴权，也可能被网络访问控制拦截。

修复分成了两部分。

第一，在 Feign 客户端上明确指定网关地址，避免服务发现或默认地址把请求送到国内服务的实例：

```java
@FeignClient(
    name = "domestic-service",
    url = "${gateway.domestic-service-url}",
    configuration = DomesticServiceClient.AuthRelayConfiguration.class
)
public interface DomesticServiceClient {
    // ...
}
```

第二，在 Feign 的 `RequestInterceptor` 中补充认证 Token 和必要的请求 Header，使请求经过网关时仍能保留调用方身份：

```java
static class AuthRelayConfiguration {
    @Bean
    RequestInterceptor authRelayInterceptor() {
        return template -> {
            String token = currentRequest().getHeader(HttpHeaders.AUTHORIZATION);
            if (StringUtils.hasText(token)) {
                template.header(HttpHeaders.AUTHORIZATION, token);
            }

            String tenantId = currentRequest().getHeader("X-Tenant-Id");
            if (StringUtils.hasText(tenantId)) {
                template.header("X-Tenant-Id", tenantId);
            }

            String requestId = currentRequest().getHeader("X-Request-Id");
            if (StringUtils.hasText(requestId)) {
                template.header("X-Request-Id", requestId);
            }
        };
    }
}
```

这段配置要求请求 Header 只做白名单转发。不能直接把入口请求的 Header 全量复制给下游，否则 `Host`、`Content-Length`、`Cookie` 等字段也可能被带到另一个请求中，造成额外的安全和协议问题。

修改后，国外服务能够通过网关调用国内服务，无权限问题消失。到这里，第一层故障已经处理完成。

## 第二处故障：国内服务无法解析 JSON

链路打通后，国内服务开始收到请求，但很快出现了新的报错。日志中的关键现象是 JSON 解析失败：接收端拿到了内容，却无法把它还原成预期的对象。

此时最容易误判为对象字段不一致、Jackson 配置变化或者前后端序列化格式不同。实际排查时，应该先把“发送前的内容”“网关收到的内容”和“国内服务收到的内容”逐段做对比。

对比后发现，Feign 确实发送了完整的结构化数据，但请求经过 Header 透传后，部分字段在到达国内服务时已经被截取。最终进入 JSON 解析器的字符串是不完整的，于是出现类似下面的异常：

```text
JsonParseException: Unexpected end-of-input
```

### Header 中的中文部分被截取

Header 中存在中文内容，传输后部分字段被截取，导致国内服务拿到的 JSON 不完整，最终解析失败。

## Header 的解决方案

header 的解决方案是把 header 中中文的部分删除掉，只传有用的字段。

## 验证

修复后按调用链逐段验证，而不是只看最终接口是否返回成功：

1. 国外服务的 Feign 请求地址确实指向网关，而不是国内服务的直连地址；
2. 网关能够识别 Token，并允许国外服务访问目标路由；
3. Header 中的中文部分已全部删除，只传递有用的字段；
4. 国内服务收到的 JSON 内容完整并能够正常解析；
5. 正常请求、无 Token 请求和 Header 缺失请求分别返回预期结果。

## 这次排查留下的判断顺序

这次问题有两个容易被表面信息带偏的地方。

第一阶段看到的是超时，但不能直接把责任归到国内服务性能。跨国调用发生异常时，要确认请求是否经过正确入口，以及请求是否具备入口要求的权限。

第二阶段看到的是 JSON 解析失败，但对象映射并没有变化。需要比较每一跳实际收到的字段，确认 Header 是否完整，而不是立刻修改模型或序列化配置。

最后得到的修复方式是：Feign 明确走网关并透传认证信息，Header 中删除中文部分，只传有用的字段。这样既解决了无权限访问，也解决了中文内容被截取后导致的 JSON 解析失败。
