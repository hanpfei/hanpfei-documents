---
title: RFC 8866 SDP：会话描述协议 II
date: 2022-01-31 12:35:49
categories: 网络协议
tags:
- 网络协议
---

# 7. 安全注意事项

SDP 经常与[会话发起协议](https://www.rfc-editor.org/rfc/rfc8866#RFC3261) [[RFC3261](https://www.rfc-editor.org/rfc/rfc8866#RFC3261)] 一起使用，使用 [提供/应答模型](https://www.rfc-editor.org/rfc/rfc8866#RFC3264) [[RFC3264](https://www.rfc-editor.org/rfc/rfc8866#RFC3264)] 来就单播会话的参数达成一致。 当以这种方式使用时，这些协议的安全考虑同样适用。

SDP 是一个描述多媒体会话的会话描述格式。接收 SDP 消息并对其采取行动的实体应该 (SHOULD) 意识到，会话描述是不可信的，除非它是通过经过身份验证和完整性保护的传输协议，从已知和可信来源获得的。许多不同的传输协议可以被用来分发会话描述，并且身份验证和完整性保护的性质也因传输而异。对于某些传输，通常不部署安全功能。如果没有以可信的方式获得会话描述，端点应该 (SHOULD) 小心，因为在其他攻击中，接收到的媒体会话可能不是预期的那个，媒体发送到的目的地可能不是预期的 ，会话的任何参数都可能不正确，或者媒体安全性可能受到损害。考虑到应用程序的安全风险和用户偏好，依赖于端点做出明智的决定 —— 端点可以决定询问用户是否接受会话。

在通过未经身份验证的传输机制或从不受信任的一方收到会话描述时，解析会话描述的软件应该采取一些预防措施。如果完整性保护没有到位，类似的问题也适用。会话描述包含了在接收者的系统上启动软件所需的信息。解析会话描述的软件不得 (MUST NOT) 启动其他软件，除非专门配置为参与多媒体会话的适当的软件。通常认为，解析会话描述的软件在用户系统上启动适合参与多媒体会话的软件，而没有首先通知用户此类软件将被启动，并征得用户同意，是不合适的。因此，通过会话公告、电子邮件、会话邀请或 WWW 页面到达的会话描述，不得 (MUST NOT) 将用户带进交互式多媒体会话中，除非用户已明确预授权此类操作。由于判断会话是否是交互式的，并不总是那么简单，因此不确定的应用程序应该假设会话是交互式的。处理会话描述中包含的 URL 的软件还应注意 [[RFC3986](https://www.rfc-editor.org/rfc/rfc8866#RFC3986)] 中确定的安全注意事项。

在本规范中，没有允许会话描述的接收者被告知，以默认传输模式启动多媒体工具的属性。在某些情况下，定义此类属性可能是合适的。如果这样做了，解析包含此类属性的会话描述的应用程序，应该忽略它们或通知用户，加入此会话将导致多媒体数据的自动传输。未知属性的默认行为是忽略它。

在某些环境中，中间系统拦截和分析包含在其他信令协议中的会话描述已变得很常见。这样做出于多种目的，包括但不限于，在防火墙中打洞以允许媒体流通过，或有选择地标记、确定优先级或阻止流量。在某些情况下，这样的中间系统可能修改会话描述，例如，使会话描述的内容与动态创建的 NAT 绑定相匹配。除非会话描述以允许中间系统进行适当检查，以建立会话描述的真实性，以及建立为建立此类通信会话的来源的权限的方式，否则不建议 (NOT RECOMMENDED) 使用这些行为。SDP 本身不包含启用这些检查的足够信息：它们依赖于封装协议（例如，SIP 或 RTSP）。使用某些程序和 SDP 扩展（例如，交互式连接建立 (ICE) [[RFC8445](https://www.rfc-editor.org/rfc/rfc8866#RFC8445)] 和 ICE-SIP-SDP [[RFC8839](https://www.rfc-editor.org/rfc/rfc8866#RFC8839)]）可以避免中间系统修改 SDP 的需要。

SDP 不得 (MUST NOT) 用于传送密钥材料（例如，使用 “a=crypto:” 属性 [[RFC4568](https://www.rfc-editor.org/rfc/rfc8866#RFC4568)]），除非可以保证，传递 SDP 的通道既是私有的又是经过身份验证的。

# 8. IANA 注意事项

## 8.1. "application/sdp" 媒体类型

[[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 中注册的一种媒体类型已更新，定义如下。

 * 类型名称：application
 * 自类型名称：sdp
 * 必须的参数：无。
 * 可选的参数：无。
 * 编码注意事项：8 位文本。SDP 文件主要是 UTF-8 格式文本。 “a=charset:” 属性可用于表示在 SDP 文件的某些部分中存在其他字符集（参见 RFC 8866 的[第 6 节](https://www.rfc-editor.org/rfc/rfc8866#attrs)）。 任意二进制内容不能在 SDP 中直接表示。
 * 安全注意事项：参见 RFC 8866 的[第 7 节](https://www.rfc-editor.org/rfc/rfc8866#security)。
 * 互操作性注意事项：参见 RFC 8866。
 * 发布的规范：参见 RFC 8866。
 * 使用这种媒体类型的应用程序：IP 语音、视频电话会议、流媒体、即时消息等。 另见 RFC 8866 的[第 3 节](https://www.rfc-editor.org/rfc/rfc8866#usage_examples)。
 * 片段标识符注意事项：无。
 * 附加信息：
     - 这个类型废弃的别名：N/A
     - 幻数：无。
     - 文件扩展名：通常使用扩展名 ".sdp"。
     - Mac 文件类型码："sdp"
 * 获取更多信息时可以联系的个人和电子邮件地址：
     - IETF MMUSIC working group
     - <mmusic@ietf.org>
 * 预期用途：一般用途 (COMMON)
 * 用途限制：无
 * 作者/修改控制者：
     - RFC 8866 的作者
     - IESG 授权的 IETF MMUSIC 工作组

## 8.2. 向 IANA 注册 SDP 参数

本文档为六个命名的 SDP 子字段指定 IANA 参数注册表。使用 SDP 规范增强巴库斯-瑙尔形式 (ABNF) 中的术语，它们是 <media>、<proto>、<attribute-name>、<bwtype>、<nettype> 和 <addrtype>。

本文档还替换和更新了先前定义在 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 中的所有这些参数的定义。

IANA 已将这些注册表中对 RFC 4566 的所有引用更改为引用本文档。

本文档中注册的所有参数的联系人姓名和电子邮件地址为：
 > IETF MMUSIC 工作组 <mmusic@ietf.org> 或由 IESG 指定的其继任者。

所有这些注册表都有如下的通用格式：
| 类型 | SDP 名称 | [其他字段] | 引用 |
|--|--|--|--|
表 3：SDP 注册表的通用格式

### 8.2.1 注册过程

为 SDP <media>、<proto>、<attribute-name>、<bwtype>、<nettype> 和 <addrtype> 等参数定义值的规范文档必须 (MUST) 包含如下信息：

 * 联系人名字
 * 联系人电子邮件地址
 * 正在定义的名称（因为它将出现在 SDP 中）
 * 名称的类型（<media>、<proto>、<attribute-name>、<bwtype>、<nettype> 或 <addrtype>）
 * 定义名称的用途描述
 * 对包含此信息和值定义的文档的稳定引用。（这通常是一个 RFC 编号。）

下面的小节指定必须为特定参数指定哪些其他信息（如果有），以及注册表中要包含哪些其他字段。

### 8.2.2 媒体类型 (<media>)

期望媒体类型集是很小的，除非在极少数情况下，否则不应 (SHOULD NOT) 扩展。 相同的规则应该适用于媒体名称以及顶级媒体类型，并且在可能的情况下，应该为 SDP 和 MIME 注册相同的名称。对于现有顶级媒体类型以外的媒体，必须 (MUST) 为要注册的新顶级媒体类型生成标准跟踪 RFC，并且注册必须 (MUST) 提供充分的理由说明为什么现有的媒体名称中没有合适的（[[RFC8126](https://www.rfc-editor.org/rfc/rfc8866#RFC8126)] 的 “标准行动” 政策）。

本备忘录注册媒体类型 "audio"、"video"、"text"、"application" 和 "message"。

注意：媒体类型 “control” 和 “data” 在本规范的早期版本 [[RFC2327](https://www.rfc-editor.org/rfc/rfc8866#RFC2327)] 中被列为有效；但是，它们的语义从未完全说明，也没有被广泛使用。这些媒体类型已在本规范中删除，尽管它们仍然是 [[RFC3840](https://www.rfc-editor.org/rfc/rfc8866#RFC3840)] 中定义的 SIP 用户代理的有效媒体类型能力。 如果这些媒体类型在未来被认为有用，则必须 (MUST) 生成标准跟踪 RFC 来记录它们的使用。在此之前，应用程序不应 (SHOULD NOT) 使用这些类型，并且不应 (SHOULD NOT) 在 SIP 能力声明中声明对它们的支持（即使它们存在于 [[RFC3840](https://www.rfc-editor.org/rfc/rfc8866#RFC3840)] 创建的注册表中）。另请注意，[[RFC6466](https://www.rfc-editor.org/rfc/rfc8866#RFC6466)] 定义了 “image” 媒体类型。

### 8.2.3. 传输协议 (<proto>)

<proto> 子字段描述所使用的传输协议。这个注册表的注册程序为 “必需的 RFC”。

本文档注册了两个值：

 * “RTP/AVP” 是对 [[RFC3550](https://www.rfc-editor.org/rfc/rfc8866#RFC3550)] 的引用，通过 UDP/IP 运行，在 [具有最小控制的音频和视频会议的 RTP 配置文件](https://www.rfc-editor.org/rfc/rfc8866#RFC3551) [[RFC3551](https://www.rfc-editor.org/rfc/rfc8866#RFC3551)] 下使用。
 * "udp" 表示直接使用 UDP。

可以 (MAY) 定义新的传输协议，并且必须 (MUST) 向 IANA 注册。 注册必须 (MUST) 引用描述协议的 RFC。这样的 RFC 可以 (MAY) 是实验性的或信息性的，尽管它最好是标准跟踪。定义新协议的 RFC 必须 (MUST) 定义管理 <fmt>（见下文）命名空间的规则。

以 “RTP/” 开头的 <proto> 名称必须 (MUST) 仅用于定义作为 RTP 配置的传输协议。 例如，短名称为 “XYZ” 的配置将由 “RTP/XYZ” 的 <proto> 子字段表示。

由 <proto> 子字段定义的每个传输协议，有一个关联的 <fmt> 命名空间，其描述了可以由该协议传递的媒体格式。格式覆盖所有可能在多媒体会话中传输的的编码。

“RTP/AVP” 和其他 “RTP/*” 配置下的 RTP 有效载荷格式必须 (MUST) 使用有效载荷类型号作为它们的 <fmt> 值。如果此会话描述动态分配有效载荷类型号，则必须 (MUST) 包含附加的 “a=rtpmap:” 属性，以指定媒体类型注册为有效负载格式定义的格式名称和参数。建议注册（与 RTP 结合）为 SDP 传输协议的其他 RTP 配置的配置，为 <fmt> 命名空间指定相同的规则。

对于 “udp” 协议，允许的 <fmt> 值是来自 IANA 媒体类型注册表的媒体子类型。媒体类型和子类型组合 <media>/<fmt> 指定 UDP 数据包主体的格式。鼓励使用格式的现有媒体子类型。如果没有合适的媒体子类型，建议 (RECOMMENDED) 通过 IETF 流程 [[RFC6838](https://www.rfc-editor.org/rfc/rfc8866#RFC6838)]，通过生成或引用定义格式的标准跟踪 RFC，来注册新的媒体子类型。

对于其他协议，可以 (MAY) 根据 <proto> 规范相关的规则注册格式。

新格式的注册必须 (MUST) 指定它们适用于哪些传输协议。

### 8.2.4. 属性名称 (<attribute-name>)

属性字段名称 (<attribute-name>) 必须 (MUST) 向 IANA 注册并记录在案，以避免由于同一名称下的属性定义冲突而导致的任何问题。（虽然 SDP 中的未知属性被简单地忽略了，但使协议碎片化的冲突属性是一个严重的问题。）

<attribute-name> 注册表的格式为：
| 类型 | SDP 名称 | 使用级别 | Mux 类别 | 引用 |
|--|--|--|--|--|
表 4：<attribute-name> 注册表的格式

例如，为会话和媒体级别定义的属性 “a=lang:” 将在新注册表中列出如下：
| 类型 | SDP 名称 | 使用级别 | Mux 类别 | 引用 |
|--|--|--|--|--|
| attribute | lang | 会话，媒体 | TRANSPORT | [RFC8866] [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] |
表 5：<attribute-name> 注册表示例

这个 <attribute-name> 注册表结合了所有以前 使用级别特定 的 “att-field” 注册表，包括 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] 所做的更新，并将 “att-field” 注册表重命名为 “attribute-name”（以前的 "att-field")" 注册表。 IANA 已完成必要的重新格式化。

本文档的 [第 6 节](https://www.rfc-editor.org/rfc/rfc8866#attrs) 替换了 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 制定的初始属性定义集。 IANA 已相应地更新了注册表。

文档可以定义新的属性，也可以扩展先前定义的属性的定义。

#### 8.2.4.1. 新属性

根据 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8866#RFC8126)] 的 “规范要求” 策略接受新属性注册，前提是规范包含以下信息：

 * 联系人姓名
 * 联系人电子邮件地址
 * 属性名称：将出现在 SDP 中的属性名称。这必须 (MUST) 符合 <attribute-name> 的定义。
 * 属性语法：对于值属性（参见 [第 5.13 节](https://www.rfc-editor.org/rfc/rfc8866#attribspec)），必须 (MUST) 提供属性值 <attribute-value> 语法（参见 [第 9 节](https://www.rfc-editor.org/rfc/rfc8866#abnf)）的 ABNF 定义。语法必须 (MUST) 遵循 [[RFC7405](https://www.rfc-editor.org/rfc/rfc8866#RFC7405)] 和 [[RFC5234](https://www.rfc-editor.org/rfc/rfc8866#RFC5234)] 的 [第 2.2 节](https://www.rfc-editor.org/rfc/rfc5234#section-2.2) 的规则形式。这应 (SHALL) 定义属性可能采用的允许值。它还可以 (MAY) 定义用于添加未来的值的扩展方法。对于属性属性，ABNF 定义被省略，因为属性属性没有值。
 * 属性语义：对于值属性，必须 (MUST) 提供该属性可能采用的值的语义描述。 属性属性的用法在下面的 目的 中进行描述。
 * 属性值：定义值的语法的 ABNF 语法规则的名称。 缺少规则名称表示该属性没有取值。 将规则名称包含在 “[” 和 “]” 中表示值是可选的。
 * 使用级别：属性的使用级别。这必须 (MUST) 是以下一项或多项：会话、媒体、源、dcsa 和 dcsa（子协议）。 有关源级属性的定义，请参阅 [[RFC5576](https://www.rfc-editor.org/rfc/rfc8866#RFC5576)]。 有关 dcsa 属性的定义，请参见 [[RFC8864](https://www.rfc-editor.org/rfc/rfc8866#RFC8864)]。
 * 字符集依赖：这必须 (MUST) 是 “是” 或 “否”，具体取决于属性值是否受制于 “a=charset:” 属性。
 * 目的：对属性的目的和用法的说明。
 * O/A 过程：在 [[RFC3264](https://www.rfc-editor.org/rfc/rfc8866#RFC3264)] 中解释的提议/应答过程。
 * Mux 类别：这必须 (MUST) 指示以下类别之一：由 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] 定义的 NORMAL、NOT RECOMMENDED、IDENTICAL、SUM、TRANSPORT、INHERIT、IDENTICAL-PER-PT、SPECIAL 或 TBD。
 * 引用：对定义属性的规范的引用。

以上是 IANA 将接受的最低要求。期望得到广泛使用和互操作性的属性，应该 (SHOULD) 用更精确地描述属性的标准跟踪 RFC 记录。

注册提交者应确保规范符合 SDP 属性的精神，最值得注意的是，属性在某种意义上是平台独立的，因为它不对操作系统做出隐含的假设，并且不会以可能抑制互操作性的方式命名特定的软件片段。

注册提交者还应谨慎选择属性的使用级别。当媒体被分开时，属性可以具有不同的值时，他们不应该只选择 “会话” 级，比如，当每个 “m =” 部分在不同的端点上都有自己的 IP 地址时。在这种情况下，选择的属性类型应该是 “会话、媒体” 或 “媒体”（取决于所需的语义）。默认规则是，对于可以同时出现在会话和媒体级别的所有新 SDP 属性，媒体级别覆盖会话级别。当新 SDP 属性不是这种情况时，必须 (MUST) 明确说明。

IANA 已使用本备忘录 [第 6 节](https://www.rfc-editor.org/rfc/rfc8866#attrs) 中的定义注册了初始属性名称集（<attribute-name> 值）（这些定义替换了 [RFC4566] 中的定义）。

#### 8.2.4.2. 对已有属性的更新

根据 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8866#RFC8126)] 的 “规范要求” 策略接受更新的属性的注册。

审查更新的指定专家，被要求评估更新是否与属性的先前意图和使用兼容，以及新文件是否具有足够的成熟度，和相对于先前文件的权威性。

更新属性的规范（例如，通过添加新值）必须 (MUST) 根据以下约束更新 [第 8.2.4.1 节](https://www.rfc-editor.org/rfc/rfc8866#newatt) 中的注册信息项：

 * 联系人姓名：必须 (MUST) 提供负责更新的实体的名称。
 * 联系人电子邮件地址：必须提供负责更新的实体的电子邮件地址。
 * 属性名称：必须 (MUST) 提供且不得 (MUST NOT) 更改。 否则它是一个新属性。
 * 属性语法：如果语法发生变化，必须 (MUST) 提供带有语法扩展的现有规则语法。对现有属性用法的修订可以 (MAY) 扩展属性的语法，但必须 (MUST) 向后兼容。
 * 属性语义：新附加属性值的语义描述，或现有值的语义扩展。现有的属性值语义必须 (MUST) 仅以向后兼容的方式扩展。
 * 使用级别：更新可以 (MAY) 只添加额外的级别。
 * 字符集依赖：不得 (MUST NOT) 更改。
 * 目的：可以 (MAY) 根据更新的用法进行扩展。
 * O/A 过程：可以 (MAY) 以向后兼容的方式更新，和/或仅适用于新的使用级别。
 * Mux 类别：除非从 “TBD” 更改为另一个值（参见 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)]），否则不更改。如果将媒体级别添加到以前不包括它的属性的定义中，它也可以 (MAY) 更改。
 * 引用：必须 (MUST) 提供新的（附加的或替换）引用。

如果属性更新对项目没有影响，则应该 (SHOULD) 省略项目。

### 8.2.5. 带宽说明符 (<bwtype>)

强烈不鼓励带宽说明符的扩散。

新带宽说明符必须 (MUST) 向 IANA 注册（<bwtype> 子字段值）。提交必须 (MUST) 引用标准跟踪 RFC，准确地指定带宽说明符的语义，并指出何时应该使用它，以及为什么现有的注册带宽说明符不够用。

RFC 必须 (MUST) 按照 [[RFC8859](https://www.rfc-editor.org/rfc/rfc8866#RFC8859)] 的定义为这个值指定 Mux 类别。

<bwtype> 注册表的格式为：
| 类型 | SDP 名称 | Mux 类别 | 引用 |
|--|--|--|--|
表 6：<bwtype> 注册表的格式

IANA 已使用本备忘录 [第 5.8 节](https://www.rfc-editor.org/rfc/rfc8866#bandwidthInfo) 中的定义，更新了带宽说明符 “CT” 和 “AS” 的 <bwtype> 注册表条目（这些定义替换了 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 中的定义）。

### 8.2.6. 网络类型 (<nettype>)

代表 Internet 的网络类型 “IN” 在本备忘录的 [第 5.2 节](https://www.rfc-editor.org/rfc/rfc8866#origin) 和 [第 5.7 节](https://www.rfc-editor.org/rfc/rfc8866#connection-information) 中定义（此定义替换 [[RFC4566](https://www.rfc-editor.org/rfc/rfc8866#RFC4566)] 中的定义）。

为了使 SDP 能够引用新的非 Internet 环境，必须 (MUST) 向 IANA 注册新的网络类型（<nettype> 子字段值）。注册受 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8866#RFC8126)] 的 “RFC 要求” 政策的约束。尽管非 Internet 环境通常不属于 IANA，但可能存在 Internet 应用程序需要与非 Internet 应用程序互操作的情况，例如将 Internet 电话呼叫网关接入公共交换电话网络 (PSTN) 时。网络类型的数量应该很少，并且应该很少扩展。新的网络类型注册必须 (MUST) 引用一个 RFC，该 RFC 提供了网络类型的详细信息，以及可能与它一起使用的地址类型。

<nettype> 注册表的格式为：
| 类型 | SDP 名称 | 可用的地址类型值 | 引用 |
|--|--|--|--|
表 7：<nettype> 注册表的格式

IANA 已将 <nettype> 注册表更新为这种新格式。以下是注册表的更新内容：
| 类型 | SDP 名称 | 可用的地址类型值 | 引用 |
|--|--|--|--|
| nettype | IN | IP4，IP6 | [RFC8866] |
| nettype | IN | RFC2543 | [[RFC2848](https://www.rfc-editor.org/rfc/rfc8866#RFC2848)] ​|
| nettype | ATM | NSAP，GWID，E164 | [[RFC3108](https://www.rfc-editor.org/rfc/rfc8866#RFC3108)] |
| nettype | ATM | E164 |   [[RFC7195](https://www.rfc-editor.org/rfc/rfc8866#RFC7195)] |
表 8：<nettype> 注册表的内容

请注意，尽管 [[RFC7195](https://www.rfc-editor.org/rfc/rfc8866#RFC7195)] 提到 “E164” 地址类型对于 ATM 和 PSTN 网络具有不同的上下文，[[RFC7195](https://www.rfc-editor.org/rfc/rfc8866#RFC7195)] 和 [[RFC3108](https://www.rfc-editor.org/rfc/rfc8866#RFC3108)] 都将 “E164” 注册为地址类型。

### 8.2.7. 地址类型 (<addrtype>)

新地址类型 (<addrtype>) 必须 (MUST) 向 IANA 注册。注册受 [[RFC8126](https://www.rfc-editor.org/rfc/rfc8866#RFC8126)] 的 “RFC 要求” 政策的约束。新的地址类型注册必须 (MUST) 引用一个 RFC，该 RFC 提供了地址类型的语法的详细信息。地址类型预计不会经常注册。

本文档的 [第 5.7 节](https://www.rfc-editor.org/rfc/rfc8866#connection-information) 给出了地址类型 “IP4” 和 “IP6” 的新定义。

## 8.3 加密密钥访问方法（已废弃）

IANA 之前维护了一个 SDP 加密密钥访问方法（“enckey”）名称表。该表已废弃，因为 “k=” 行不可扩展。 不得 (MUST NOT) 接受新的注册。

# 9. SDP 语法

本节为 SDP 提供一个增强 BNF 语法。ABNF 在 [[RFC5234](https://www.rfc-editor.org/rfc/rfc8866#RFC5234)] 和 [[RFC7405](https://www.rfc-editor.org/rfc/rfc8866#RFC7405)] 中定义。

```
; SDP Syntax
session-description = version-field
                     ​origin-field
                     ​session-name-field
                     ​[information-field]
                     ​[uri-field]
                     ​*email-field
                     ​*phone-field
                     ​[connection-field]
                     ​*bandwidth-field
                     ​1*time-description
                     ​[key-field]
                     ​*attribute-field
                     ​*media-description

version-field =       %s"v" "=" 1*DIGIT CRLF
                         ​;this memo describes version 0

origin-field =       %s"o" "=" username SP sess-id SP sess-version SP
                        ​nettype SP addrtype SP unicast-address CRLF

session-name-field =  %s"s" "=" text CRLF

information-field =   %s"i" "=" text CRLF

uri-field =           %s"u" "=" uri CRLF

email-field =         %s"e" "=" email-address CRLF

phone-field =         %s"p" "=" phone-number CRLF

connection-field =    %s"c" "=" nettype SP addrtype SP
                         ​connection-address CRLF
                         ​;a connection field must be present
                         ​;in every media description or at the
                         ​;session level

bandwidth-field =     %s"b" "=" bwtype ":" bandwidth CRLF

time-description =    time-field
                         ​[repeat-description]

repeat-description =  1*repeat-field
                         ​[zone-field]

time-field =          %s"t" "=" start-time SP stop-time CRLF

repeat-field =        %s"r" "=" repeat-interval SP typed-time
                         ​1*(SP typed-time) CRLF

zone-field =          %s"z" "=" time SP ["-"] typed-time
                         ​*(SP time SP ["-"] typed-time) CRLF

key-field =           %s"k" "=" key-type CRLF

attribute-field =     %s"a" "=" attribute CRLF

media-description =   media-field
                     ​[information-field]
                     ​*connection-field
                     ​*bandwidth-field
                     ​[key-field]
                     ​*attribute-field

media-field =         %s"m" "=" media SP port ["/" integer]
                         ​SP proto 1*(SP fmt) CRLF

; sub-rules of 'o='
username =            non-ws-string
                     ​;pretty wide definition, but doesn't
                     ​;include space

sess-id =             1*DIGIT
                     ​;should be unique for this username/host

sess-version =        1*DIGIT

nettype =             token
                     ​;typically "IN"

addrtype =            token
                     ​;typically "IP4" or "IP6"

; sub-rules of 'u='
uri =                 URI-reference
                     ​; see RFC 3986

; sub-rules of 'e=', see RFC 5322 for definitions
email-address        = address-and-comment / dispname-and-address
                      ​/ addr-spec
address-and-comment  = addr-spec 1*SP "(" 1*email-safe ")"
dispname-and-address = 1*email-safe 1*SP "<" addr-spec ">"

; sub-rules of 'p='
phone-number =        phone *SP "(" 1*email-safe ")" /
                     ​1*email-safe "<" phone ">" /
                     ​phone

phone =               ["+"] DIGIT 1*(SP / "-" / DIGIT)

; sub-rules of 'c='
connection-address =  multicast-address / unicast-address

; sub-rules of 'b='
bwtype =              token

bandwidth =           1*DIGIT

; sub-rules of 't='
start-time =          time / "0"

stop-time =           time / "0"

time =                POS-DIGIT 9*DIGIT
                     ​; Decimal representation of time in
                     ​; seconds since January 1, 1900 UTC.
                     ​; The representation is an unbounded
                     ​; length field containing at least
                     ​; 10 digits. Unlike some representations
                     ​; used elsewhere, time in SDP does not
                     ​; wrap in the year 2036.

; sub-rules of 'r=' and 'z='
repeat-interval =     POS-DIGIT *DIGIT [fixed-len-time-unit]

typed-time =          1*DIGIT [fixed-len-time-unit]

fixed-len-time-unit = %s"d" / %s"h" / %s"m" / %s"s"
; NOTE: These units are case-sensitive.

; sub-rules of 'k='
key-type =            %s"prompt" /
                     ​%s"clear:" text /
                     ​%s"base64:" base64 /
                     ​%s"uri:" uri
                     ​; NOTE: These names are case-sensitive.

base64      =         *base64-unit [base64-pad]
base64-unit =         4base64-char
base64-pad  =         2base64-char "==" / 3base64-char "="
base64-char =         ALPHA / DIGIT / "+" / "/"

; sub-rules of 'a='
attribute =           (attribute-name ":" attribute-value) /
                     ​attribute-name

attribute-name =      token

attribute-value =     byte-string

att-field =           attribute-name ; for backward compatibility

; sub-rules of 'm='
media =               token
                     ​;typically "audio", "video", "text", "image"
                     ​;or "application"

fmt =                 token
                     ​;typically an RTP payload type for audio
                     ​;and video media

proto  =              token *("/" token)
                     ​;typically "RTP/AVP", "RTP/SAVP", "udp",
                     ​;or "RTP/SAVPF"

port =                1*DIGIT

; generic sub-rules: addressing
unicast-address =     IP4-address / IP6-address / FQDN / extn-addr

multicast-address =   IP4-multicast / IP6-multicast / FQDN
                     ​/ extn-addr

IP4-multicast =       m1 3( "." decimal-uchar )
                     ​"/" ttl [ "/" numaddr ]
                     ​; IP4 multicast addresses may be in the
                     ​; range 224.0.0.0 to 239.255.255.255

m1 =                  ("22" ("4"/"5"/"6"/"7"/"8"/"9")) /
                     ​("23" DIGIT )

IP6-multicast =       IP6-address [ "/" numaddr ]
                     ​; IP6 address starting with FF

numaddr =             integer

ttl =                 (POS-DIGIT *2DIGIT) / "0"

FQDN =                4*(alpha-numeric / "-" / ".")
                     ​; fully qualified domain name as specified
                     ​; in RFC 1035 (and updates)

IP4-address =         b1 3("." decimal-uchar)

b1 =                  decimal-uchar
                     ​; less than "224"

IP6-address =                                      6( h16 ":" ) ls32
                     ​/                       "::" 5( h16 ":" ) ls32
                     ​/ [               h16 ] "::" 4( h16 ":" ) ls32
                     ​/ [ *1( h16 ":" ) h16 ] "::" 3( h16 ":" ) ls32
                     ​/ [ *2( h16 ":" ) h16 ] "::" 2( h16 ":" ) ls32
                     ​/ [ *3( h16 ":" ) h16 ] "::"    h16 ":"   ls32
                     ​/ [ *4( h16 ":" ) h16 ] "::"              ls32
                     ​/ [ *5( h16 ":" ) h16 ] "::"              h16
                     ​/ [ *6( h16 ":" ) h16 ] "::"

h16 =                 1*4HEXDIG

ls32 =                ( h16 ":" h16 ) / IP4-address

; Generic for other address families
extn-addr =      non-ws-string

; generic sub-rules: datatypes
text =                byte-string
                     ​;default is to interpret this as UTF8 text.
                     ​;ISO 8859-1 requires "a=charset:ISO-8859-1"
                     ​;session-level attribute to be used

byte-string =         1*(%x01-09/%x0B-0C/%x0E-FF)
                     ​;any byte except NUL, CR, or LF

non-ws-string =       1*(VCHAR/%x80-FF)
                     ​;string of visible characters

token-char =          ALPHA / DIGIT
                             ​/ "!" / "#" / "$" / "%" / "&"
                             ​/ "'" ; (single quote)
                             ​/ "*" / "+" / "-" / "." / "^" / "_"
                             ​/ "`" ; (Grave accent)
                             ​/ "{" / "|" / "}" / "~"

token =               1*(token-char)

email-safe =          %x01-09/%x0B-0C/%x0E-27/%x2A-3B/%x3D/%x3F-FF
                     ​;any byte except NUL, CR, LF, or the quoting
                     ​;characters ()<>

integer =             POS-DIGIT *DIGIT

zero-based-integer = "0" / integer

non-zero-int-or-real = integer / non-zero-real

non-zero-real = zero-based-integer "." *DIGIT POS-DIGIT

; generic sub-rules: primitives
alpha-numeric =       ALPHA / DIGIT

POS-DIGIT =           %x31-39 ; 1 - 9

decimal-uchar =       DIGIT
                     ​/ POS-DIGIT DIGIT
                     ​/ ("1" 2(DIGIT))
                     ​/ ("2" ("0"/"1"/"2"/"3"/"4") DIGIT)
                     ​/ ("2" "5" ("0"/"1"/"2"/"3"/"4"/"5"))

; external references:
ALPHA =               <ALPHA definition from RFC 5234>
DIGIT =               <DIGIT definition from RFC 5234>
CRLF =                <CRLF definition from RFC 5234>
HEXDIG =              <HEXDIG definition from RFC 5234>
SP =                  <SP definition from RFC 5234>
VCHAR =               <VCHAR definition from RFC 5234>
URI-reference =       <URI-reference definition from RFC 3986>
addr-spec =           <addr-spec definition from RFC 5322>
```

# 10. 自 RFC 4566 以来的变更摘要

​* 总体上澄清和完善的术语。文本中使用的术语与 ABNF 对齐。术语 <attribute>、<att-field> 和 “att-field” 现在是 <attribute-name>。术语 <value> 和 <att-value> 现在是 <attribute-value>。 术语 "media" 现在是 <media>。

​* 已确定现已过时的项目：“a=cat:”（第 6.1 节）、“a=keywds:”（第 6.2 节）和 “k=”（第 5.12 节）。

​* 更新了规范性和信息性参考，并添加了对其他相关 RFC 的参考。

​* 重新格式化 SDP 属性一节（第 6 节）以提高可读性。 属性值的语法现在以 ABNF 给出。

​* 强制发送带有非活动媒体流的 RTCP（第 6.7.4 节）。

​* 删除了 “私有会话” 小节。 该小节可以追溯到 SDP 主要与 SAP（会话公告协议）一起使用的时候，后者已不再使用。现在 SDP 的绝大多数用途是建立私人会话。第 7 节介绍了对此的注意事项。

​* 扩展并阐明了 “a=lang:”（第 6.12 节）和 “a=sdplang:”（第 6.11 节）属性的规范。

​* 删除了对 SAP 的一些引用，因为它不再广泛使用。

​* 更改了注册 UDP 传输的 <fmt> 值的方式（第 8.2.3 节）。

​* 更改了注册新属性所需的机制和文档（第 8.2.4.1 节）。

​* 收紧了 IANA 扩展注册程序。

​* 删除了电话号码和长格式名称（第 8.2 节）。

​* 扩展了 IANA <nettype> 注册表以标识有效的 <addrtype> 子字段（第 8.2.6 节）。

​* 将多个 IANA “att-field” 注册表重组为单个 <attribute-name> 注册表（第 8.2.4 节）。

​* 修订 ABNF 语法（第 9 节），以提高清晰度并与文本对齐。保持向后兼容性，但有少数例外。

    ​- 修订了时间描述的语法（“t=”、“r=”、“z=”）以消除歧义。澄清 “z=” 只修改前面的 “r=” 行。使没有前面的 “r=” 的 “z=” 成为语法错误（第 5.11 节）。（这与某些异常用法不兼容。）

    ​- 更新了 “IP6-address” 和 “IP6-multicast” 规则，与 [RFC3986] 中的语法一致，反映了 [RFC5954] 对 [RFC3261] 所做的错误修复。删除了由于此更改而未使用的规则。

    ​- “att-field” 规则已重命名为 “attribute-name”，因为在其他地方 “*-field” 总是指完整的行。但是，规则名称 “att-field” 仍然被定义为一个同义词，以与其他 RFC 的引用向后兼容。

​* “att-value” 规则已重命名为 “attribute-value”。

​* 修订了以 ABNF 语法来说冗余的规范性声明，使文本变为非规范性的。

​* 根据 [RFC5735] 和 [RFC5771] 修改了示例 SDP 描述中的 IPv4 单播和多播地址。

​* 更改了一些示例以使用 IPv6 地址，并添加了使用 IPv6 的其他示例。

​* 合并了来自 [RFC4855] 的不区分大小写规则。

​* 修订了错误引用 NTP 的部分（第 5.2 节、第 5.9 节、第 5.10 节和第 5.11 节）。

​* 澄清了对 “a=charset:” 属性的影响和使用的解释（第 6.10 节）。

​* 修订了 “a=type:” 属性的描述，以消除它有时会将默认媒体方向更改为 “a=sendrecv” 以外的其他内容的暗示（第 6.9 节）。

# 11. 参考 (References)

## 11.1. 规范性参考 (Normative References)

```
  ​[E164]     International Telecommunication Union, "E.164 : The
             ​international public telecommunication numbering plan",
             ​ITU Recommendation E.164, November 2010,
             ​<https://www.itu.int/rec/T-REC-E.164-201011-I/en>.

  ​[ISO.8859-1.1998]
             ​International Organization for Standardization,
             ​"Information technology - 8-bit single byte coded graphic
             ​- character sets - Part 1: Latin alphabet No. 1, JTC1/
             ​SC2", ISO/IEC Standard 8859-1, 1998.

  ​[RFC1034]  Mockapetris, P., "Domain names - concepts and facilities",
             ​STD 13, RFC 1034, DOI 10.17487/RFC1034, November 1987,
             ​<https://www.rfc-editor.org/info/rfc1034>.

  ​[RFC1035]  Mockapetris, P., "Domain names - implementation and
             ​specification", STD 13, RFC 1035, DOI 10.17487/RFC1035,
             ​November 1987, <https://www.rfc-editor.org/info/rfc1035>.

  ​[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
             ​Requirement Levels", BCP 14, RFC 2119,
             ​DOI 10.17487/RFC2119, March 1997,
             ​<https://www.rfc-editor.org/info/rfc2119>.

  ​[RFC2848]  Petrack, S. and L. Conroy, "The PINT Service Protocol:
             ​Extensions to SIP and SDP for IP Access to Telephone Call
             ​Services", RFC 2848, DOI 10.17487/RFC2848, June 2000,
             ​<https://www.rfc-editor.org/info/rfc2848>.

  ​[RFC2978]  Freed, N. and J. Postel, "IANA Charset Registration
             ​Procedures", BCP 19, RFC 2978, DOI 10.17487/RFC2978,
             ​October 2000, <https://www.rfc-editor.org/info/rfc2978>.

  ​[RFC3108]  Kumar, R. and M. Mostafa, "Conventions for the use of the
             ​Session Description Protocol (SDP) for ATM Bearer
             ​Connections", RFC 3108, DOI 10.17487/RFC3108, May 2001,
             ​<https://www.rfc-editor.org/info/rfc3108>.

  ​[RFC3629]  Yergeau, F., "UTF-8, a transformation format of ISO
             ​10646", STD 63, RFC 3629, DOI 10.17487/RFC3629, November
             ​2003, <https://www.rfc-editor.org/info/rfc3629>.

  ​[RFC3986]  Berners-Lee, T., Fielding, R., and L. Masinter, "Uniform
             ​Resource Identifier (URI): Generic Syntax", STD 66,
             ​RFC 3986, DOI 10.17487/RFC3986, January 2005,
             ​<https://www.rfc-editor.org/info/rfc3986>.

  ​[RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session
             ​Description Protocol", RFC 4566, DOI 10.17487/RFC4566,
             ​July 2006, <https://www.rfc-editor.org/info/rfc4566>.

  ​[RFC5234]  Crocker, D., Ed. and P. Overell, "Augmented BNF for Syntax
             ​Specifications: ABNF", STD 68, RFC 5234,
             ​DOI 10.17487/RFC5234, January 2008,
             ​<https://www.rfc-editor.org/info/rfc5234>.

  ​[RFC5576]  Lennox, J., Ott, J., and T. Schierl, "Source-Specific
             ​Media Attributes in the Session Description Protocol
             ​(SDP)", RFC 5576, DOI 10.17487/RFC5576, June 2009,
             ​<https://www.rfc-editor.org/info/rfc5576>.

  ​[RFC5646]  Phillips, A., Ed. and M. Davis, Ed., "Tags for Identifying
             ​Languages", BCP 47, RFC 5646, DOI 10.17487/RFC5646,
             ​September 2009, <https://www.rfc-editor.org/info/rfc5646>.

  ​[RFC5890]  Klensin, J., "Internationalized Domain Names for
             ​Applications (IDNA): Definitions and Document Framework",
             ​RFC 5890, DOI 10.17487/RFC5890, August 2010,
             ​<https://www.rfc-editor.org/info/rfc5890>.

  ​[RFC5952]  Kawamura, S. and M. Kawashima, "A Recommendation for IPv6
             ​Address Text Representation", RFC 5952,
             ​DOI 10.17487/RFC5952, August 2010,
             ​<https://www.rfc-editor.org/info/rfc5952>.

  ​[RFC7195]  Garcia-Martin, M. and S. Veikkolainen, "Session
             ​Description Protocol (SDP) Extension for Setting Audio and
             ​Video Media Streams over Circuit-Switched Bearers in the
             ​Public Switched Telephone Network (PSTN)", RFC 7195,
             ​DOI 10.17487/RFC7195, May 2014,
             ​<https://www.rfc-editor.org/info/rfc7195>.

  ​[RFC8126]  Cotton, M., Leiba, B., and T. Narten, "Guidelines for
             ​Writing an IANA Considerations Section in RFCs", BCP 26,
             ​RFC 8126, DOI 10.17487/RFC8126, June 2017,
             ​<https://www.rfc-editor.org/info/rfc8126>.

  ​[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
             ​2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
             ​May 2017, <https://www.rfc-editor.org/info/rfc8174>.

  ​[RFC8859]  Nandakumar, S., "A Framework for Session Description
             ​Protocol (SDP) Attributes When Multiplexing", RFC 8859,
             ​DOI 10.17487/RFC8859, January 2021,
             ​<https://www.rfc-editor.org/info/rfc8859>.

  ​[RFC8864]  Drage, K., Makaraju, M., Ejzak, R., Marcon, J., and R.
             ​Even, Ed., "Negotiation Data Channels Using the Session
             ​Description Protocol (SDP)", RFC 8864,
             ​DOI 10.17487/RFC8864, January 2021,
             ​<https://www.rfc-editor.org/info/rfc8864>.
```

## 11.2. 参考资料 (Informative References)

```
  ​[ITU.H332.1998]
             ​International Telecommunication Union, "H.332 : H.323
             ​extended for loosely coupled conferences", ITU
             ​Recommendation H.332, September 1998,
             ​<https://www.itu.int/rec/T-REC-H.332-199809-I/en>.

  ​[RFC2045]  Freed, N. and N. Borenstein, "Multipurpose Internet Mail
             ​Extensions (MIME) Part One: Format of Internet Message
             ​Bodies", RFC 2045, DOI 10.17487/RFC2045, November 1996,
             ​<https://www.rfc-editor.org/info/rfc2045>.

  ​[RFC2327]  Handley, M. and V. Jacobson, "SDP: Session Description
             ​Protocol", RFC 2327, DOI 10.17487/RFC2327, April 1998,
             ​<https://www.rfc-editor.org/info/rfc2327>.

  ​[RFC2974]  Handley, M., Perkins, C., and E. Whelan, "Session
             ​Announcement Protocol", RFC 2974, DOI 10.17487/RFC2974,
             ​October 2000, <https://www.rfc-editor.org/info/rfc2974>.

  ​[RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston,
             ​A., Peterson, J., Sparks, R., Handley, M., and E.
             ​Schooler, "SIP: Session Initiation Protocol", RFC 3261,
             ​DOI 10.17487/RFC3261, June 2002,
             ​<https://www.rfc-editor.org/info/rfc3261>.

  ​[RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model
             ​with Session Description Protocol (SDP)", RFC 3264,
             ​DOI 10.17487/RFC3264, June 2002,
             ​<https://www.rfc-editor.org/info/rfc3264>.

  ​[RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
             ​Jacobson, "RTP: A Transport Protocol for Real-Time
             ​Applications", STD 64, RFC 3550, DOI 10.17487/RFC3550,
             ​July 2003, <https://www.rfc-editor.org/info/rfc3550>.

  ​[RFC3551]  Schulzrinne, H. and S. Casner, "RTP Profile for Audio and
             ​Video Conferences with Minimal Control", STD 65, RFC 3551,
             ​DOI 10.17487/RFC3551, July 2003,
             ​<https://www.rfc-editor.org/info/rfc3551>.

  ​[RFC3556]  Casner, S., "Session Description Protocol (SDP) Bandwidth
             ​Modifiers for RTP Control Protocol (RTCP) Bandwidth",
             ​RFC 3556, DOI 10.17487/RFC3556, July 2003,
             ​<https://www.rfc-editor.org/info/rfc3556>.

  ​[RFC3605]  Huitema, C., "Real Time Control Protocol (RTCP) attribute
             ​in Session Description Protocol (SDP)", RFC 3605,
             ​DOI 10.17487/RFC3605, October 2003,
             ​<https://www.rfc-editor.org/info/rfc3605>.

  ​[RFC3711]  Baugher, M., McGrew, D., Naslund, M., Carrara, E., and K.
             ​Norrman, "The Secure Real-time Transport Protocol (SRTP)",
             ​RFC 3711, DOI 10.17487/RFC3711, March 2004,
             ​<https://www.rfc-editor.org/info/rfc3711>.

  ​[RFC3840]  Rosenberg, J., Schulzrinne, H., and P. Kyzivat,
             ​"Indicating User Agent Capabilities in the Session
             ​Initiation Protocol (SIP)", RFC 3840,
             ​DOI 10.17487/RFC3840, August 2004,
             ​<https://www.rfc-editor.org/info/rfc3840>.

  ​[RFC3890]  Westerlund, M., "A Transport Independent Bandwidth
             ​Modifier for the Session Description Protocol (SDP)",
             ​RFC 3890, DOI 10.17487/RFC3890, September 2004,
             ​<https://www.rfc-editor.org/info/rfc3890>.

  ​[RFC4568]  Andreasen, F., Baugher, M., and D. Wing, "Session
             ​Description Protocol (SDP) Security Descriptions for Media
             ​Streams", RFC 4568, DOI 10.17487/RFC4568, July 2006,
             ​<https://www.rfc-editor.org/info/rfc4568>.

  ​[RFC4855]  Casner, S., "Media Type Registration of RTP Payload
             ​Formats", RFC 4855, DOI 10.17487/RFC4855, February 2007,
             ​<https://www.rfc-editor.org/info/rfc4855>.

  ​[RFC5124]  Ott, J. and E. Carrara, "Extended Secure RTP Profile for
             ​Real-time Transport Control Protocol (RTCP)-Based Feedback
             ​(RTP/SAVPF)", RFC 5124, DOI 10.17487/RFC5124, February
             ​2008, <https://www.rfc-editor.org/info/rfc5124>.

  ​[RFC5322]  Resnick, P., Ed., "Internet Message Format", RFC 5322,
             ​DOI 10.17487/RFC5322, October 2008,
             ​<https://www.rfc-editor.org/info/rfc5322>.

  ​[RFC5735]  Cotton, M. and L. Vegoda, "Special Use IPv4 Addresses",
             ​RFC 5735, DOI 10.17487/RFC5735, January 2010,
             ​<https://www.rfc-editor.org/info/rfc5735>.

  ​[RFC5771]  Cotton, M., Vegoda, L., and D. Meyer, "IANA Guidelines for
             ​IPv4 Multicast Address Assignments", BCP 51, RFC 5771,
             ​DOI 10.17487/RFC5771, March 2010,
             ​<https://www.rfc-editor.org/info/rfc5771>.

  ​[RFC5888]  Camarillo, G. and H. Schulzrinne, "The Session Description
             ​Protocol (SDP) Grouping Framework", RFC 5888,
             ​DOI 10.17487/RFC5888, June 2010,
             ​<https://www.rfc-editor.org/info/rfc5888>.

  ​[RFC5954]  Gurbani, V., Ed., Carpenter, B., Ed., and B. Tate, Ed.,
             ​"Essential Correction for IPv6 ABNF and URI Comparison in
             ​RFC 3261", RFC 5954, DOI 10.17487/RFC5954, August 2010,
             ​<https://www.rfc-editor.org/info/rfc5954>.

  ​[RFC6466]  Salgueiro, G., "IANA Registration of the 'image' Media
             ​Type for the Session Description Protocol (SDP)",
             ​RFC 6466, DOI 10.17487/RFC6466, December 2011,
             ​<https://www.rfc-editor.org/info/rfc6466>.

  ​[RFC6838]  Freed, N., Klensin, J., and T. Hansen, "Media Type
             ​Specifications and Registration Procedures", BCP 13,
             ​RFC 6838, DOI 10.17487/RFC6838, January 2013,
             ​<https://www.rfc-editor.org/info/rfc6838>.

  ​[RFC7230]  Fielding, R., Ed. and J. Reschke, Ed., "Hypertext Transfer
             ​Protocol (HTTP/1.1): Message Syntax and Routing",
             ​RFC 7230, DOI 10.17487/RFC7230, June 2014,
             ​<https://www.rfc-editor.org/info/rfc7230>.

  ​[RFC7405]  Kyzivat, P., "Case-Sensitive String Support in ABNF",
             ​RFC 7405, DOI 10.17487/RFC7405, December 2014,
             ​<https://www.rfc-editor.org/info/rfc7405>.

  ​[RFC7656]  Lennox, J., Gross, K., Nandakumar, S., Salgueiro, G., and
             ​B. Burman, Ed., "A Taxonomy of Semantics and Mechanisms
             ​for Real-Time Transport Protocol (RTP) Sources", RFC 7656,
             ​DOI 10.17487/RFC7656, November 2015,
             ​<https://www.rfc-editor.org/info/rfc7656>.

  ​[RFC7826]  Schulzrinne, H., Rao, A., Lanphier, R., Westerlund, M.,
             ​and M. Stiemerling, Ed., "Real-Time Streaming Protocol
             ​Version 2.0", RFC 7826, DOI 10.17487/RFC7826, December
             ​2016, <https://www.rfc-editor.org/info/rfc7826>.

  ​[RFC8445]  Keranen, A., Holmberg, C., and J. Rosenberg, "Interactive
             ​Connectivity Establishment (ICE): A Protocol for Network
             ​Address Translator (NAT) Traversal", RFC 8445,
             ​DOI 10.17487/RFC8445, July 2018,
             ​<https://www.rfc-editor.org/info/rfc8445>.

  ​[RFC8839]  Petit-Huguenin, M., Nandakumar, S., Holmberg, C., Keränen,
             ​A., and R. Shpount, "Session Description Protocol (SDP)
             ​Offer/Answer Procedures for Interactive Connectivity
             ​Establishment (ICE)", RFC 8839, DOI 10.17487/RFC8839,
             ​January 2021, <https://www.rfc-editor.org/info/rfc8839>.

  ​[RFC8843]  Holmberg, C., Alvestrand, H., and C. Jennings,
             ​"Negotiating Media Multiplexing Using the Session
             ​Description Protocol (SDP)", RFC 8843,
             ​DOI 10.17487/RFC8843, January 2021,
             ​<https://www.rfc-editor.org/info/rfc8843>.
```

# 致谢

IETF 多方多媒体会话控制 (MMUSIC) 工作组中的许多人通过提出意见和建议对本文档做出了贡献。

我们要特别感谢为创建本文档或其前身文档之一做出贡献的以下人员：Adam Roach、Allison Mankin、Bernie Hoeneisen、Bill Fenner、Carsten Bormann、Eve Schooler、Flemming Andreasen、Gonzalo Camarillo、Jörg Ott、John Elwell、Jon Peterson、Jonathan Lennox、Jonathan Rosenberg、Keith Drage、Peter Parnes、Rob Lanphier、Ross Finlayson、Sean Olson、Spencer Dawkins、Steve Casner、Steve Hanna、Van Jacobson。

# 作者的地址

**Ali Begen**
Networked Media
Turkey
Email: ali.begen@networked.media

**Paul Kyzivat**
United States of America
Email: pkyzivat@alum.mit.edu

**Colin Perkins**
University of Glasgow
School of Computing Science
Glasgow
G12 8QQ
United Kingdom
Email: csp@csperkins.org

**Mark Handley**
University College London
Department of Computer Science
London
WC1E 6BT
United Kingdom
Email: M.Handley@cs.ucl.ac.uk


[原文](https://www.rfc-editor.org/rfc/rfc8866
