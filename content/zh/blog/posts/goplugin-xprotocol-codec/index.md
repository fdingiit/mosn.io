---
title: "基于goplugin技术的编解码器动态加载实现"
linkTitle: "基于goplugin技术的编解码器插件实现"
date: 2021-02-03
author: "[fd](https://github.com/fdingiit)"
description: "本文会重点介绍基于goplugin技术的mosn编解码器实现。"
aliases: “/zh/blog/posts/goplugin-xprotocol-codec"
---

**M**odular**OSN**的模块扩展能力，不仅体现在其灵活的系统架构设计层面，同时体现在独立于MOSN的插件式扩展开发层面。本文依据蚂蚁集团围绕「协议」这个常见扩展点进行的探索，介绍基于goplugin技术的编解码器插件实现。
在阅读本文之前，推荐浏览以下两篇前置文章，以帮助您理解本文内容。

- 《[MOSN 多协议机制解析](https://mosn.io/blog/posts/multi-protocol-deep-dive/)》
- 《[MOSN 扩展机制解析](https://mosn.io/blog/posts/mosn-extensions/)》



### why
mosn在设计之初便提供了一套公共API框架——XProtocol——用以支持多协议及其扩展，开发者通过实现几个简单的接口，便可以将一个标准或私有协议接入MOSN，赋予其微服务+Mesh场景下的各种强大能力。
xprotocol协议框架虽然足够灵活，但依旧略有不足：协议实现与mosn强耦合，即协议对mosn代码的侵入性过大。从长久的架构设计的角度来看，协议的实现不应是sidecar的「组成部分」，而应该是sidecar的「上层业务」；从实践的角度来看，协议的新增、变更，不应该影响到mosn核心功能的CR、发布、生产。换句话说，协议作为一种Domain Specific Language，mosn应该对其实现一无所知。

因此，mosn开源及蚂蚁中间件团队针对这个痛点进行了一系列探索，一方面升级了xprotocol框架，一方面落地了基于goplguin技术的codec动态加载解决方案。

### what

#### 什么是xprotocol
xprotocol是mosn专门针对扩展协议提供的一套通用API，通过实现这套接口，任何协议都可以被mosn所接管。事实上，蚂蚁集团内部最多使用的bolt协议就是基于这套接口实现的。

Xprotocol接口定义
```go
type XProtocol interface {
	Protocol

	Heartbeater

	Hijacker

	PoolMode() PoolMode // configure this to use which connpool

	EnableWorkerPool() bool // same meaning as EnableWorkerPool in types.StreamConnection

	// generate a request id for stream to combine stream request && response
	// use connection param as base
	GenerateRequestID(*uint64) uint64
}
```

更多关于xprotocol的信息，请阅读《[MOSN 多协议机制解析](https://mosn.io/blog/posts/multi-protocol-deep-dive/)》

#### 什么是goplugin
goplugin机制是golang在1.8版本发布的一个语言级feature。其目的是提供一种方式，使程序的某些功能以插件的形式（*.so文件）与主程序在时间和空间上解耦：时间上，功能将会在程序运行时，而不是启动时被加载；空间上，无论从代码角度，还是物理角度，功能将独立于主程序存在。

这里给个图！！！！

### how

#### 新架构api

#### plugin的开发
新架构下plugin的开发与老架构基本完全一致，本质为实现xprotocol协议。这部分内容请参考《[MOSN 多协议机制解析](https://mosn.io/blog/posts/multi-protocol-deep-dive/)》。
除此之外，需要额外开发实现一个新增接口和一个新增方法：

新增接口：`XProtocolCodec`
```go
// XProtocolCodec provides the ability to load a codec plugin at runtime
type XProtocolCodec interface {
	// ProtocolName returns the name of the protocol
	ProtocolName() ProtocolName

	// XProtocol returns the implementation for XProtocol of the protocol
	XProtocol() XProtocol

	// ProtocolMatxch returns the matcher function of the protocol
	ProtocolMatch() ProtocolMatch

	// HTTPMapping returns the implementation for HTTPMapping of the protocol
	HTTPMapping() HTTPMapping
}
```

1. `ProtocolName() ProtocolName`
此方法需要返回编解码器实现的协议名，例如假设协议名为`bolt`：

```go
func (b boltCodec) ProtocolName() api.ProtocolName {
	return "bolt"
}
```

2. `XProtocol() XProtocol`
此方法需要返回编解码器对于xprotocol协议的具体实现；假设`type boltCodec struct{}`已经实现了所有api，则此函数可以是：

```go
func (b boltCodec) XProtocol() api.XProtocol {
	return &boltProtocol{}
}
```

3. `ProtocolMatch() ProtocolMatch`
此方法需要返回编解码器的协议嗅探方法，用于协议的自动识别。对于bolt协议来说，嗅探方法的实现可能是：
```go
func boltMatcher(data []byte) api.MatchResult {
	length := len(data)
	if length == 0 {
		return api.MatchAgain
	}

	if data[0] == ProtocolCode {
		return api.MatchSuccess
	}

	return api.MatchFailed
}
```

则本方法的实现可以是：
```go
func (b boltCodec) ProtocolMatch() api.ProtocolMatch {
	return boltMatcher
}
```


4. `HTTPMapping() HTTPMapping`
此方法需要返回此协议状态码与http标准状态码之间的映射关系。bolt的映射关系可以是：
```go
type boltStatusMapping struct{}

func (m *boltStatusMapping) MappingHeaderStatusCode(ctx context.Context, headers api.HeaderMap) (int, error) {
	cmd, ok := headers.(api.XRespFrame)
	if !ok {
		return 0, errors.New("no response status in headers")
	}
	code := uint16(cmd.GetStatusCode())
	// TODO: more accurate mapping
	switch code {
	case ResponseStatusSuccess:
		return http.StatusOK, nil
	case ResponseStatusServerThreadpoolBusy:
		return http.StatusServiceUnavailable, nil
	case ResponseStatusTimeout:
		return http.StatusGatewayTimeout, nil
		//case RESPONSE_STATUS_CLIENT_SEND_ERROR: // CLIENT_SEND_ERROR maybe triggered by network problem, 404 is not match
		//	return http.StatusNotFound, nil
	case ResponseStatusConnectionClosed:
		return http.StatusBadGateway, nil
	default:
		return http.StatusInternalServerError, nil
	}
}
```

则本方法的实现可以是：
```go
func (b boltCodec) HTTPMapping() api.HTTPMapping {
	return &boltStatusMapping{}
}
```

新增方法：　`func() XProtocolCodec`
新架构下，mosn通过符合以上这个函数签名的方法拉起协议插件。这个函数协议名可以由开发者决定，并在配置文件里指明；若不指明，其默认函数名为`LoadCodec`。
此函数只需要把具体XProtocolCodec接口的实现返回即可，例如：
```go
func LoadCodec() api.XProtocolCodec {
	return &boltCodec{}
}
```

如上，对接新架构动态加载codec的工作便全部完成了。如果开发者已经实现了xprotocol协议框架，则为了对接goplugin模式所新增的工作量不过是把已经实现的接口和方法作为新增接口各个方法的返回值返回罢了。代码量不超过20行，耗时不超过3分钟。


#### plugin的使用
为了使用动态codec，用户需要在mosn的配置文件里增添一些内容，举个例子：
```json
  "third_part_codec": {
    "codecs": [
      {
        "enable": true,
        "type": "go-plugin",
        "path": "path/to/bolt.so"
      }
    ]
  }
  ```

这里配置了第三方编解码器的主要信息，包括
- 启用标记位
- 编解码器类型，暂时只支持go-plugin
- 编解码器文件路径

配置完这些信息之后，mosn便可以在初始化时尝试识别并拉起对应的编解码器了

```bash
2021-02-02 20:07:14,76 [INFO] [mosn] [init codec] loading protocol [bolt] from third part codec
2021-02-02 20:07:14,76 [INFO] [mosn] [init codec] load go plugin codec succeed: path/to/bolt.so

```

### misc



### about author


