---
title: 交互式连接建立 (ICE)：一种用于网络地址转换 (NAT) 穿越的协议 III
date: 2022-01-08 14:37:49
categories: 网络协议
tags:
- 网络协议
---

# 18. IAB 注意事项

IAB 研究了 “单边自地址修复”（UNSAF）问题，这是 ICE 代理尝试通过协作协议反射机制确定在 NAT 另一侧的另一个域其地址的一般过程 [ [RFC3424](https://www.rfc-editor.org/rfc/rfc3424) ]。ICE 是执行此类功能的协议的一个示例。有趣的是，ICE 的过程不是单边的，而是双边的，并且这种差异对 IAB 提出的问题有重大影响。实际上，ICE 可以被视为一个 双边自地址修复 (B-SAF) 协议，而不是 UNSAF 协议。无论如何，IAB 已强制要求为此目的开发的任何协议都记录一组特定的注意事项。本节满足这些要求。

## 18.1. 问题定义

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 精确定义将通过 UNSAF 提案解决的特定、范围有限的问题。不应将短期解决方案推广到解决其它问题。这种泛化导致对提议的短期修复的长期依赖和使用 —— 这意味着将其称为 “短期” 不再准确。
 
ICE正在解决的具体问题是：

> 为两个对等方提供一种确定可用于通信的传输地址集的方法。

> 为代理提供一种方法来确定它希望与之通信的另一个对等方可到达的地址。

## 18.2. 退出策略

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 退出策略/过渡计划的描述。更好的短期修复是那些随着适当技术的部署自然地越来越少使用的修复。

ICE 本身并不容易被淘汰。但是，即使在全球连接的互联网中，它也很有用，例如，它可以用作检测路由器故障是否暂时中断连接的手段。ICE 还有助于防止某些与 NAT 无关的安全攻击。然而，ICE 所做的是帮助逐步淘汰其它 UNSAF 机制。ICE 有效地在这些机制中进行挑选，提高更好的机制的优先级，而降低更差的机制的优先级。随着 IPv6 的引入，NAT 开始消散，服务器自反和中继候选地址（两种形式的 UNSAF 地址）根本不会被使用，因为本地主机候选地址存在更高优先级的连接。因此，服务器的使用越来越少，最终可以在它们的使用量变为零时被删除。

事实上，ICE 可以帮助从 IPv4 过渡到 IPv6。当两台双栈主机与 SIP 通信（使用 IPv6）时，它可用于确定是使用 IPv6 还是使用 IPv4。它还可以允许具有 6to4 和本机 v6 连接的网络确定在与对等方通信时使用哪个地址。

## 18.3. ICE 引入的脆弱性

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 讨论可能使系统更加 “脆弱” 的具体问题。例如，涉及在多个网络层使用数据的方法会产生更多的依赖，增加调试挑战，并使其更难转换。

ICE 实际上消除了现有 UNSAF 机制的脆弱性。特别是，经典 STUN（如 [RFC 3489](https://www.rfc-editor.org/rfc/rfc3489) [[RFC3489](https://www.rfc-editor.org/rfc/rfc3489)] 中所述）有几个脆弱点。其中一个是发现过程，该过程需要 ICE 代理尝试对其身处其后的 NAT 的类型进行分类。这个过程很容易出错。使用 ICE，根本不使用该发现过程。不是单方面评估地址的有效性，而是通过测量与对等方的连接性来动态确定其有效性。确定连接性的过程非常健壮。

经典 STUN 和任何其它单边机制的另一个脆弱点是它对附加服务器的绝对依赖。ICE 使用服务器来分配单边地址，但如果可能，它允许代理直接连接。因此，在某些情况下，即使 STUN 服务器发生故障，使用 ICE 时仍然允许进行通话。

经典 STUN 的另一个脆弱点是它假设 STUN 服务器位于公共互联网上。有趣的是，对于 ICE，这是没有必要的。在各种地址领域中可以有大量的 STUN 服务器。ICE 会发现提供了可用地址的那个。

经典 STUN 中最令人不安的脆弱点是它不适用于所有网络拓扑。在各个代理与 STUN 服务器之间存在共享的 NAT 的情况下，传统的 STUN 可能无法工作。有了 ICE，这个限制就被清除了。

经典 STUN 还引入了一些安全注意事项。幸运的是，ICE 也减轻了这些安全注意事项。

因此，ICE 用于修复经典 STUN 中引入的脆性，并且不会在系统中引入任何额外的脆性。

这些改进的代价是 ICE 增加了会话建立时间。

## 18.4. 长期解决方案的要求

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 确定长期、健全的技术解决方案的需求； 帮助找到正确的长期解决方案的过程。

我们从 [RFC 3489](https://www.rfc-editor.org/rfc/rfc3489) 得出的结论保持不变。然而，我们认为 ICE 确实有帮助，因为我们相信它可以成为长期解决方案的一部分。

## 18.5. 与现有 NAPT 设备一起使用时的问题

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 讨论指出的实际问题对现有、已部署的 NA[P]T 和经验报告的影响。

许多 NAT 设备现在正被部署到市场中，试图提供 “通用” ALG 功能。这些通用 ALG 在数据包中以文本或二进制形式搜索 IP 地址，并在它们与绑定匹配时重写它们。这会干扰经典的 STUN。然而，对 STUN [[RFC5389](https://www.rfc-editor.org/rfc/rfc5389)] 的更新使用了一种编码，该编码对通用 ALG 隐藏了这些二进制地址。

对于基于 UDP 的绑定，现有的 NAPT 设备具有不确定且通常较短的到期时间。这需要实现发送周期性的保活来维护这些绑定。ICE 使用 15 秒的默认值，这是一个非常保守的估计。最终，随着时间的推移，随着 NAT 设备的行为变得符合 [[RFC4787](https://www.rfc-editor.org/rfc/rfc4787)]，这个最小保活将变得确定和众所周知，并且可以调整 ICE 计时器。有一种方法来发现和控制最小保活间隔会更好。

# 19. 安全注意事项

## 19.1. IP 地址隐私

探测候选地址的过程向任何网络上监听的攻击者揭示了客户端及其对等方的源地址，而交换候选地址的过程向任何能够看到协商的攻击者揭示了地址。某些地址，例如通过 VPN 用户的本地接口收集的服务器自反地址，可能是敏感信息。如果无法缓解这些潜在的攻击，ICE 使用可以定义控制哪些地址向协商和/或探测过程显示的机制。单独的实现也可以具有用于控制显示哪些地址的特定于实现的规则。例如，[[WebRTC-IP-HANDLING](https://www.rfc-editor.org/rfc/rfc8445.html#ref-WebRTC-IP-HANDLING)] 提供了有关通过 ICE 为 WebRTC 应用程序揭示 IP 地址的隐私方面的额外信息。建议 (RECOMMENDED) 在可能出现此类问题的 ICE 实现中提供编程或用户接口，以控制使用哪些网络接口来生成候选地址。

根据对等方提供的候选地址类型，以及针对这些候选地址执行的连接测试的结果，对等方可能能够确定本地网络的特征，例如，如果对等方具有明显不同的时间。在限制范围内，对等方可能能够探测本地网络。

ICE 系统中可能存在多种类型的攻击。这些小节考虑了这些攻击及其对策。

## 19.2. 对连接性检查的攻击

攻击者可能会尝试破坏 STUN 连接检查。最终，所有这些攻击都会欺骗 ICE 代理，使其对连接检查的结果有不正确的认知。根据攻击的类型，攻击者需要具备不同的能力。在某些情况下，攻击者需要在连接检查的路径上。在其它情况下，攻击者不需要在路径上，只要它能够生成 STUN 连接检查就可以。尽管对连接检查的攻击通常由网络实体执行，但如果攻击者能够控制端点，它可能能够触发连接检查攻击。攻击者可以尝试并导致的可能错误结论是：

错误的无效：攻击者可以欺骗一个代理对使其认为一个候选地址对是无效的，而实际上并非如此。这可被用于使代理优先一个不同的候选地址（例如由攻击者注入的候选地址）或通过强制所有候选地址失败来中断呼叫。

错误的有效：攻击者可以欺骗一个代理对使其认为一个候选地址对是有效的，而实际上并非如此。这可能会导致代理继续进行会话，但随后无法接收任何数据。

错误的对等方自反候选地址：攻击者可以使代理发现一个新的对等方自反候选地址，而这是不期望的。这可用于将数据流重定向到 DoS 目标或攻击者，用于窃听或其它目的。

错误的候选地址之上的错误有效：攻击者已经让代理相信有一个候选地址具有一个实际上不路由到该代理的地址（例如，通过注入虚假的对等方自反候选地址或虚假的服务器自反候选地址）。然后攻击者发起攻击，迫使代理相信这个候选地址是有效的。

如果攻击者可以导致错误的对等方自反候选地址或错误的候选地址之上的错误的有效，它可以发起 [[RFC5389](https://www.rfc-editor.org/rfc/rfc5389)] 中描述的任何攻击。

为了强制错误的无效结果，攻击者必须等待来自其中一个代理的连接检查被发送。如果是这样，攻击者需要注入一个带有不可恢复错误响应（例如 400）的虚假响应，或者丢弃响应以使其永远不会到达代理。但是，由于候选地址实际上是有效的，原始请求可能会到达对等方代理并导致成功响应。攻击者需要通过 DoS 攻击、第 2 层网络中断或其它技术强制丢弃此数据包或其响应。如果它不这样做，成功响应也将到达发起者，提醒它可能的攻击。攻击者生成虚假响应的能力通过 STUN 短期凭证机制得到缓解。为了使这个响应被处理，攻击者需要密码。如果候选地址交换信令是安全的，则攻击者将无法拥有密码，且它的响应将被丢弃。

欺骗性 ICMP 硬错误（类型 3，代码 2 - 4）也可用于创建错误的无效结果。如果 ICE 代理对这些 ICMP 错误做出响应，则攻击者能够生成 ICMP 消息，该消息将传递给发送连接检查的代理。代理对 ICMP 错误消息的验证是其唯一的防御措施。对于类型 3 代码=4，外部 IP 标头不提供验证，除非连接检查发送时设置 DF=0。对于由主机发起的代码 2 或 3，其地址应为远程代理的主机、自反或中继候选 IP 地址中的任何一个。ICMP 消息包括触发错误的消息的 IP 头和 UDP 头。这些字段也需要验证。IP 目标和 UDP 目标端口需要匹配目标候选地址和端口或候选地址的基地址。源 IP 地址和端口可以是发送连接检查的代理的相同基地址的任何候选地址。因此，任何有权访问候选地址交换的攻击者都将拥有必要的信息。因此，验证是一个薄弱的防御，并且没有源地址验证的话，来自网络中的节点的非路径攻击者发送欺骗性的 ICMP 攻击也是可能的。

强制伪造的有效结果以类似的方式工作。攻击者需要等待来自每个代理的绑定请求并注入一个虚假的成功响应。同样，由于 STUN 短期凭证机制，为了让攻击者注入有效的成功响应，攻击者需要密码。或者，攻击者可以将通常会被网络丢弃或拒绝的有效成功响应路由（例如，使用隧道机制）到代理。

可以通过虚假请求或响应或重放来强制错误的对等方自反候选地址结果。我们首先考虑虚假请求和响应的情况。它要求攻击者向一个代理发送绑定请求，其中包含虚假候选地址的源 IP 地址和端口。此外，攻击者需要等待来自其它代理的绑定请求，并生成一个带有包含虚假候选地址的 XOR-MAPPED-ADDRESS 属性的虚假响应。与此处描述的其它攻击一样，STUN 消息完整性机制和安全的候选地址交换可以缓解这种攻击。

使用数据包重放强制错误的对等方自反候选地址结果是不同的。攻击者一直等到其中一个代理发送检查。它拦截此请求并以伪造的源 IP 地址将其重放给其它代理。它还需要防止原始请求到达远程代理，方法是发起 DoS 攻击以导致数据包被丢弃或使用第 2 层机制强制丢弃它。由于完整性检查通过（完整性检查不能也不会覆盖源 IP 地址和端口），因此重放的数据包在另一个代理处被接收并被接受。然后对其进行响应。该响应将包含一个带有错误候选地址的 XOR-MAPPED-ADDRESS，并将发送给该错误候选地址。然后，攻击者需要接收它并将其重放给发起者。

然后，另一个代理将启动对该错误候选地址的连接检查。 此验证需要成功。 这要求攻击者对错误的候选地址强制进行错误的验证。使用 STUN 和候选地址交换的完整性机制可以防止注入伪造的请求或响应来达到此目的。这样，这种攻击只能通过重放来启动。为此，攻击者需要拦截到这个虚假候选地址的检查，并将其重放给另一个代理。然后，它需要拦截响应并将其重放回去。

除非攻击者由假候选地址标识，否则这种攻击很难发起。这是因为它要求攻击者拦截并重放两个不同主机发送的数据包。如果两个代理都在不同的网络上（例如，通过公共互联网），这种攻击可能很难协调，因为它需要同时针对网络不同部分的两个不同端点发生。

如果攻击者本身由假候选地址标识，则攻击更容易协调。但是，如果数据路径是安全的（例如，使用安全实时传输协议 (SRTP) [[RFC3711](https://www.rfc-editor.org/rfc/rfc3711)]），攻击者将无法处理数据包，而只能丢弃它们，从而有效地禁用数据流。但是，此攻击需要代理中断数据包以阻止连接检查到达目标。在这种情况下，如果目标是破坏数据流，那么用相同的机制破坏数据流本身比攻击 ICE 要容易得多。

## 19.3. 对服务器自反地址收集的攻击

ICE 端点使用 STUN 绑定请求从一个 STUN 服务器收集服务器自反候选地址。这些请求没有以任何方式认证。因此，攻击者有大量技术可以用来给客户端提供一个错误的服务器自反候选地址：

 * 攻击者可以破坏 DNS，导致 DNS 查询返回恶意 STUN 服务器地址。该服务器可以为客户端提供虚假的服务器自反候选地址。DNS 安全可以缓解这种攻击，尽管不需要 DNSSEC 来解决它。

 * 可以观察 STUN 消息的攻击者（例如共享网段上的攻击者，如 Wi-Fi）可以注入一个有效的虚假响应，并将被客户端接受。

 * 攻击者可以破坏 STUN 服务器并导致它发送带有不正确映射地址的响应。

通过这些攻击学习到的映射地址将在 ICE 会话的建立中被用作服务器自反候选地址。为了让这个候选地址真正被用于数据，攻击者还需要攻击连接检查，特别是强制一个错误的候选地址上的错误的有效。如果错误地址标识了第四方（既不是发起者、响应者也不是攻击者），这种攻击很难发起，因为它需要攻击会话中每个 ICE 代理生成的检查，并且如果它标识攻击者本身，则 SRTP 会阻止它。

如果攻击者选择不攻击连接检查，它所能做的最坏的事情就是阻止使用服务器自反候选地址。但是，如果对等方代理至少有一个受攻击的代理可以访问的候选地址，则 STUN 连接检查本身将提供可用于数据交换的对等方自反候选地址。对等方自反候选地址通常比服务器自反候选地址优先级更高。因此，近针对 STUN 地址收集的攻击通常不会对会话产生任何影响。

## 19.4. 对中继候选地址收集的攻击

攻击者可能会试图破坏中继候选地址的收集，迫使客户端相信它有一个错误的中继候选地址。与 TURN 服务器的交换使用长期凭证进行身份验证。因此，注入虚假响应或请求将不生效。此外，与绑定 (Binding) 请求不同，分配 (Allocate) 请求不易受到修改源 IP 地址和端口的重放攻击的影响，因为源 IP 地址和端口不用于为客户端提供其中继候选地址。

即使攻击者使客户端相信了错误的中继候选地址，连接性检查也会导致此类候选地址仅在成功时才被使用。因此，攻击者需要启动一个错误的候选地址上的错误的有效，如上所述，这是一个非常难以协调的攻击。

## 19.5. 内部攻击

除了攻击者是试图插入虚假候选地址信息或 STUN 消息的第三方的攻击之外，当攻击者是 ICE 交换中经过身份验证且有效的参与者时，ICE 还有一些可能的攻击。

### 19.5.1. STUN 放大攻击

STUN 放大攻击类似于 “语音锤” 攻击，其中攻击者使其它代理将语音数据包定向到攻击目标。但是，不是将语音数据包定向到目标，而是将 STUN 连接检查定向到目标。攻击者发送大量候选地址，例如，50 个。响应代理接收候选地址信息并启动它的被定向到目标的检查，因此，永远不会产生响应。在 WebRTC 的情况下，用户甚至可能不知道这种攻击正在进行，因为它可能是由用户获取的恶意 JavaScript 代码在后台触发的。应答者将每隔 Ta ms 开始一次新的连接检查（例如，Ta=50ms）。然而，由于大量的候选地址，重传定时器被设置为很大的数字。因此，数据包将以每 Ta 毫秒一个的间隔发送，然后间隔增加。因此，STUN 不会以比数据发送速度更快的速率发送数据包，并且 STUN 数据包只会短暂持续，直到 ICE 会话失败。尽管如此，这是一种放大机制。

消除放大是不可能的，但可以通过各种启发式方法来减小数量。ICE 代理应该 (SHOULD) 将它们执行的连接检查的总数限制为 100。此外，代理可以 (MAY) 限制它们接受的候选地址的数量。

通常，希望避免此类攻击的协议会强制发起者在发送下一条消息之前等待响应。然而，在 ICE 的情况下，这是不可能的。无法区分以下两种情况：

 * 因为发起者正被用来对一个毫无戒心的目标发起 DoS 攻击而没有响应，该目标不会响应。

 * 因为发起者无法访问 IP 地址和端口而没有响应。

在第二种情况下，下次有机会发送另一个检查，而在前一种情况下，将不再发送检查。

# 20. IANA 注意事项

最初的 ICE 规范注册了四个 STUN 属性和一个新的 STUN 错误响应。STUN 属性和错误响应在此处重现。此外，该规范还注册了一个新的 ICE 选项。

## 20.1. STUN 属性

IANA 注册了四个 STUN 属性：
```
      0x0024 PRIORITY
      0x0025 USE-CANDIDATE
      0x8029 ICE-CONTROLLED
      0x802A ICE-CONTROLLING
```

## 20.2. STUN 错误响应

IANA 已注册以下 STUN 错误响应码：

487    角色冲突 (Role Conflict)：客户端断言与服务器角色冲突的 ICE 角色（控制或受控）。

## 20.3. ICE 选项

IANA 已按照 [[RFC6336](https://www.rfc-editor.org/rfc/rfc6336)] 中定义的程序，在 “交互式连接建立 (ICE)” 注册表的 “ICE 选项” 子注册表中注册了以下 ICE 选项。

ICE 选项名称：
```
      ice2
```

联系人：
```
      名称:    IESG
      Email:   iesg@ietf.org
```

修改控制人：
```
      IESG
```

描述：
ICE 选项表示使用 ICE 选项的 ICE 代理是根据 [RFC 8445](https://www.rfc-editor.org/rfc/rfc8445) 实现的。

参考：
[RFC 8445](https://www.rfc-editor.org/rfc/rfc8445)

# 21. 对 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245) 的更改

此更新的 ICE 规范的目的是：

 * 阐明 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245) 中的过程。

 * 由于在 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245) 中发现的缺陷，以及来自已基于 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245) 实现和部署 ICE 应用程序的社区的反馈，进行技术更改。

 * 通过删除 SIP 和 SDP 过程，使过程独立于信令协议。特定于信令协议的过程将在单独的使用文档中定义。[[ICE-SIP-SDP](https://www.rfc-editor.org/rfc/rfc8445.html#ref-ICE-SIP-SDP)] 使用 SIP 和 SDP 定义 ICE 使用。

进行了以下技术更改：

 * 积极提名被移除。

 * 计算候选地址对状态和调度连接性检查的过程已修改。

 * 计算 Ta 和 RTO 的过程已修改。

 * 活跃检查列表和冻结检查列表定义已移除。

 * 'ice2' ICE 选项已添加。

 * IPv6 注意事项已修改。

 * 对 keepalive 的无操作使用，以及与非 ICE 对等方的 keepalive 已移除。

# 22. 参考

## 22.1. 规范性参考

```
   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <https://www.rfc-editor.org/info/rfc2119>.

   [RFC4941]  Narten, T., Draves, R., and S. Krishnan, "Privacy
              Extensions for Stateless Address Autoconfiguration in
              IPv6", RFC 4941, DOI 10.17487/RFC4941, September 2007,
              <https://www.rfc-editor.org/info/rfc4941>.

   [RFC5389]  Rosenberg, J., Mahy, R., Matthews, P., and D. Wing,
              "Session Traversal Utilities for NAT (STUN)", RFC 5389,
              DOI 10.17487/RFC5389, October 2008,
              <https://www.rfc-editor.org/info/rfc5389>.

   [RFC5766]  Mahy, R., Matthews, P., and J. Rosenberg, "Traversal Using
              Relays around NAT (TURN): Relay Extensions to Session
              Traversal Utilities for NAT (STUN)", RFC 5766,
              DOI 10.17487/RFC5766, April 2010,
              <https://www.rfc-editor.org/info/rfc5766>.

   [RFC6336]  Westerlund, M. and C. Perkins, "IANA Registry for
              Interactive Connectivity Establishment (ICE) Options",
              RFC 6336, DOI 10.17487/RFC6336, July 2011,
              <https://www.rfc-editor.org/info/rfc6336>.

   [RFC6724]  Thaler, D., Ed., Draves, R., Matsumoto, A., and T. Chown,
              "Default Address Selection for Internet Protocol Version 6
              (IPv6)", RFC 6724, DOI 10.17487/RFC6724, September 2012,
              <https://www.rfc-editor.org/info/rfc6724>.

   [RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
              2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
              May 2017, <https://www.rfc-editor.org/info/rfc8174>.
```

## 22.2. 参考资料

```
   [ICE-SIP-SDP]
              Petit-Huguenin, M., Nandakumar, S., and A. Keranen,
              "Session Description Protocol (SDP) Offer/Answer
              procedures for Interactive Connectivity Establishment
              (ICE)", Work in Progress,
              draft-ietf-mmusic-ice-sip-sdp-21, June 2018.

   [RFC1918]  Rekhter, Y., Moskowitz, B., Karrenberg, D., de Groot, G.,
              and E. Lear, "Address Allocation for Private Internets",
              BCP 5, RFC 1918, DOI 10.17487/RFC1918, February 1996,
              <https://www.rfc-editor.org/info/rfc1918>.

   [RFC2475]  Blake, S., Black, D., Carlson, M., Davies, E., Wang, Z.,
              and W. Weiss, "An Architecture for Differentiated
              Services", RFC 2475, DOI 10.17487/RFC2475, December 1998,
              <https://www.rfc-editor.org/info/rfc2475>.

   [RFC3102]  Borella, M., Lo, J., Grabelsky, D., and G. Montenegro,
              "Realm Specific IP: Framework", RFC 3102,
              DOI 10.17487/RFC3102, October 2001,
              <https://www.rfc-editor.org/info/rfc3102>.

   [RFC3103]  Borella, M., Grabelsky, D., Lo, J., and K. Taniguchi,
              "Realm Specific IP: Protocol Specification", RFC 3103,
              DOI 10.17487/RFC3103, October 2001,
              <https://www.rfc-editor.org/info/rfc3103>.

   [RFC3235]  Senie, D., "Network Address Translator (NAT)-Friendly
              Application Design Guidelines", RFC 3235,
              DOI 10.17487/RFC3235, January 2002,
              <https://www.rfc-editor.org/info/rfc3235>.

   [RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston,
              A., Peterson, J., Sparks, R., Handley, M., and E.
              Schooler, "SIP: Session Initiation Protocol", RFC 3261,
              DOI 10.17487/RFC3261, June 2002,
              <https://www.rfc-editor.org/info/rfc3261>.

   [RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model
              with Session Description Protocol (SDP)", RFC 3264,
              DOI 10.17487/RFC3264, June 2002,
              <https://www.rfc-editor.org/info/rfc3264>.

   [RFC3303]  Srisuresh, P., Kuthan, J., Rosenberg, J., Molitor, A., and
              A. Rayhan, "Middlebox communication architecture and
              framework", RFC 3303, DOI 10.17487/RFC3303, August 2002,
              <https://www.rfc-editor.org/info/rfc3303>.

   [RFC3424]  Daigle, L., Ed. and IAB, "IAB Considerations for
              UNilateral Self-Address Fixing (UNSAF) Across Network
              Address Translation", RFC 3424, DOI 10.17487/RFC3424,
              November 2002, <https://www.rfc-editor.org/info/rfc3424>.

   [RFC3489]  Rosenberg, J., Weinberger, J., Huitema, C., and R. Mahy,
              "STUN - Simple Traversal of User Datagram Protocol (UDP)
              Through Network Address Translators (NATs)", RFC 3489,
              DOI 10.17487/RFC3489, March 2003,
              <https://www.rfc-editor.org/info/rfc3489>.

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
              Jacobson, "RTP: A Transport Protocol for Real-Time
              Applications", STD 64, RFC 3550, DOI 10.17487/RFC3550,
              July 2003, <https://www.rfc-editor.org/info/rfc3550>.

   [RFC3605]  Huitema, C., "Real Time Control Protocol (RTCP) attribute
              in Session Description Protocol (SDP)", RFC 3605,
              DOI 10.17487/RFC3605, October 2003,
              <https://www.rfc-editor.org/info/rfc3605>.

   [RFC3711]  Baugher, M., McGrew, D., Naslund, M., Carrara, E., and K.
              Norrman, "The Secure Real-time Transport Protocol (SRTP)",
              RFC 3711, DOI 10.17487/RFC3711, March 2004,
              <https://www.rfc-editor.org/info/rfc3711>.

   [RFC3725]  Rosenberg, J., Peterson, J., Schulzrinne, H., and G.
              Camarillo, "Best Current Practices for Third Party Call
              Control (3pcc) in the Session Initiation Protocol (SIP)",
              BCP 85, RFC 3725, DOI 10.17487/RFC3725, April 2004,
              <https://www.rfc-editor.org/info/rfc3725>.

   [RFC3879]  Huitema, C. and B. Carpenter, "Deprecating Site Local
              Addresses", RFC 3879, DOI 10.17487/RFC3879, September
              2004, <https://www.rfc-editor.org/info/rfc3879>.

   [RFC4038]  Shin, M-K., Ed., Hong, Y-G., Hagino, J., Savola, P., and
              E. Castro, "Application Aspects of IPv6 Transition",
              RFC 4038, DOI 10.17487/RFC4038, March 2005,
              <https://www.rfc-editor.org/info/rfc4038>.

   [RFC4091]  Camarillo, G. and J. Rosenberg, "The Alternative Network
              Address Types (ANAT) Semantics for the Session Description
              Protocol (SDP) Grouping Framework", RFC 4091,
              DOI 10.17487/RFC4091, June 2005,
              <https://www.rfc-editor.org/info/rfc4091>.

   [RFC4092]  Camarillo, G. and J. Rosenberg, "Usage of the Session
              Description Protocol (SDP) Alternative Network Address
              Types (ANAT) Semantics in the Session Initiation Protocol
              (SIP)", RFC 4092, DOI 10.17487/RFC4092, June 2005,
              <https://www.rfc-editor.org/info/rfc4092>.

   [RFC4103]  Hellstrom, G. and P. Jones, "RTP Payload for Text
              Conversation", RFC 4103, DOI 10.17487/RFC4103, June 2005,
              <https://www.rfc-editor.org/info/rfc4103>.

   [RFC4291]  Hinden, R. and S. Deering, "IP Version 6 Addressing
              Architecture", RFC 4291, DOI 10.17487/RFC4291, February
              2006, <https://www.rfc-editor.org/info/rfc4291>.

   [RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session
              Description Protocol", RFC 4566, DOI 10.17487/RFC4566,
              July 2006, <https://www.rfc-editor.org/info/rfc4566>.

   [RFC4787]  Audet, F., Ed. and C. Jennings, "Network Address
              Translation (NAT) Behavioral Requirements for Unicast
              UDP", BCP 127, RFC 4787, DOI 10.17487/RFC4787, January
              2007, <https://www.rfc-editor.org/info/rfc4787>.

   [RFC5245]  Rosenberg, J., "Interactive Connectivity Establishment
              (ICE): A Protocol for Network Address Translator (NAT)
              Traversal for Offer/Answer Protocols", RFC 5245,
              DOI 10.17487/RFC5245, April 2010,
              <https://www.rfc-editor.org/info/rfc5245>.

   [RFC5382]  Guha, S., Ed., Biswas, K., Ford, B., Sivakumar, S., and P.
              Srisuresh, "NAT Behavioral Requirements for TCP", BCP 142,
              RFC 5382, DOI 10.17487/RFC5382, October 2008,
              <https://www.rfc-editor.org/info/rfc5382>.

   [RFC5761]  Perkins, C. and M. Westerlund, "Multiplexing RTP Data and
              Control Packets on a Single Port", RFC 5761,
              DOI 10.17487/RFC5761, April 2010,
              <https://www.rfc-editor.org/info/rfc5761>.

   [RFC6080]  Petrie, D. and S. Channabasappa, Ed., "A Framework for
              Session Initiation Protocol User Agent Profile Delivery",
              RFC 6080, DOI 10.17487/RFC6080, March 2011,
              <https://www.rfc-editor.org/info/rfc6080>.

   [RFC6146]  Bagnulo, M., Matthews, P., and I. van Beijnum, "Stateful
              NAT64: Network Address and Protocol Translation from IPv6
              Clients to IPv4 Servers", RFC 6146, DOI 10.17487/RFC6146,
              April 2011, <https://www.rfc-editor.org/info/rfc6146>.

   [RFC6147]  Bagnulo, M., Sullivan, A., Matthews, P., and I. van
              Beijnum, "DNS64: DNS Extensions for Network Address
              Translation from IPv6 Clients to IPv4 Servers", RFC 6147,
              DOI 10.17487/RFC6147, April 2011,
              <https://www.rfc-editor.org/info/rfc6147>.

   [RFC6298]  Paxson, V., Allman, M., Chu, J., and M. Sargent,
              "Computing TCP's Retransmission Timer", RFC 6298,
              DOI 10.17487/RFC6298, June 2011,
              <https://www.rfc-editor.org/info/rfc6298>.

   [RFC6544]  Rosenberg, J., Keranen, A., Lowekamp, B., and A. Roach,
              "TCP Candidates with Interactive Connectivity
              Establishment (ICE)", RFC 6544, DOI 10.17487/RFC6544,
              March 2012, <https://www.rfc-editor.org/info/rfc6544>.

   [RFC6928]  Chu, J., Dukkipati, N., Cheng, Y., and M. Mathis,
              "Increasing TCP's Initial Window", RFC 6928,
              DOI 10.17487/RFC6928, April 2013,
              <https://www.rfc-editor.org/info/rfc6928>.

   [RFC7050]  Savolainen, T., Korhonen, J., and D. Wing, "Discovery of
              the IPv6 Prefix Used for IPv6 Address Synthesis",
              RFC 7050, DOI 10.17487/RFC7050, November 2013,
              <https://www.rfc-editor.org/info/rfc7050>.

   [RFC7721]  Cooper, A., Gont, F., and D. Thaler, "Security and Privacy
              Considerations for IPv6 Address Generation Mechanisms",
              RFC 7721, DOI 10.17487/RFC7721, March 2016,
              <https://www.rfc-editor.org/info/rfc7721>.

   [RFC7825]  Goldberg, J., Westerlund, M., and T. Zeng, "A Network
              Address Translator (NAT) Traversal Mechanism for Media
              Controlled by the Real-Time Streaming Protocol (RTSP)",
              RFC 7825, DOI 10.17487/RFC7825, December 2016,
              <https://www.rfc-editor.org/info/rfc7825>.

   [RFC8421]  Martinsen, P., Reddy, T., and P. Patil, "Interactive
              Connectivity Establishment (ICE) Multihomed and IPv4/IPv6
              Dual-Stack Guidelines", RFC 8421, DOI 10.17487/RFC8421,
              July 2018, <https://www.rfc-editor.org/info/rfc8421>.

   [WebRTC-IP-HANDLING]
              Uberti, J. and G. Shieh, "WebRTC IP Address Handling
              Requirements", Work in Progress, draft-ietf-rtcweb-ip-
              handling-09, June 2018.
```

# 附录 A. 精简和完整实现

ICE 允许两种类型的实现。完整实现支持会话中的控制和受控角色，还可以执行地址收集。相比之下，精简实现是一个极简实现，它做的很少，但响应 STUN 检查，它只支持会话中的受控角色。

因为 ICE 需要两个端点都支持它才能为任一端点带来好处，所以在网络中增量部署 ICE 更加复杂。许多会话都包括一个端点，该端点本身并不位于 NAT 之后，也不会担心 NAT 穿越。一个非常常见的情况是让一个需要 NAT 穿越的端点（例如 VoIP 硬电话或软电话）呼叫这些设备之一。即使手机支持完整的 ICE 实现，如果其它设备不支持，则根本不会使用 ICE。精简版实现为这些设备提供了一个低成本的入口点。一旦它们支持精简实现，完整的实现就可以连接到它们并获得 ICE 的全部好处。

因此，精简实现仅适用于将 *始终* 被连接到公共 Internet 并具有公网 IP 地址的设备，它可以在该地址处接收来自任何通信方的数据包。当精简实现置于 NAT 之后时，ICE 将不起作用。

ICE 允许精简实现具有单个 IPv4 主机候选地址和多个 IPv6 地址。在这种情况下，候选地址对由控制代理使用静态算法选择，例如本规范推荐的 [RFC 6724](https://www.rfc-editor.org/rfc/rfc6724) 中的算法。然而，地址选择的静态机制总是容易出错，因为它们永远不能反映实际的拓扑结构或提供连接性的实际保证。它们总是启发式的。 因此，如果 ICE 代理实现的 ICE 只是在其 IPv4 和 IPv6 地址之间进行选择，并且其 IP 地址均不位于 NAT 之后，则仍建议 (RECOMMENDED) 使用完整的 ICE 以提供可能的最健壮的地址选择形式。

重要的是要注意，精简版实现已添加到本规范中，以提供到完整实现的垫脚石。即使对于始终仅通过一个 IPv4 地址连接到公共 Internet 的设备，如果可以，完整实现也是更可取的。完整实现也获得了与 NAT 穿越无关的 ICE 的安全优势。最后，很常见的情况是，一个设备今天发现自己具有公网地址，明天就将被放置在 NAT 后面的网络中。在设备或产品的整个生命周期内，很难明确知道它是否会一直在公共互联网上使用。全面实现可确保通信始终可以工作。

# 附录 B. 设计动机

ICE 包含许多规范行为，这些行为本身可能很简单，但源自复杂或不明显的想法或值得进一步讨论的用例。由于这些设计动机不是为了实现的目的而必须理解的，因此在这里对其进行讨论。本附录是非规范性的。

## B.1. STUN 事务的步调

用于收集候选地址和验证连接性的 STUN 事务以大约每 Ta 毫秒一个新事务的速率进行。反过来，每个事务都有一个重传计时器 RTO，它也是 Ta 的函数。为什么这些事务是有节奏的，为什么要使用这些公式？

发送这些 STUN 请求通常会在客户端和 STUN 服务器之间的 NAT 设备上创建绑定。经验表明，许多 NAT 设备对它们创建新绑定的速率都有上限。IETF ICE WG 在本规范工作期间的讨论得出结论，每 5 ms 一次得到了很好的支持。这就是 Ta 的下限为 5 ms 的原因。此外，这些数据包在网络上的传输会占用带宽，并且需要由 ICE 代理进行速率限制。基于 [[RFC5245](https://www.rfc-editor.org/rfc/rfc5245)] 早期草案版本的部署往往会使速率受限的接入链路过载，并且总体性能不佳，此外还会对网络产生负面影响。因此，调步可确保 NAT 设备不会过载，并将流量保持在合理的速率。

“合理的” 速率的定义是，STUN 使用的带宽不得 (MUST NOT) 超过，数据开始流动之后 RTP 本身使用的带宽。Ta 的公式是这样设计的，如果每 Ta 秒发送一个 STUN 数据包，它将消耗与 RTP 数据包相同数量的带宽，所有数据流的总和。当然，STUN 有重传，并且希望也能调整它们的速度。出于这个原因，设置 RTO 使得第一个事务上的第一次重传发生在最后一个事务上的第一个 STUN 请求发生时。 图示：
```
              First Packets              Retransmits



                    |                        |
                    |                        |
             -------+------           -------+------
            /               \        /               \
           /                 \      /                 \

           +--+    +--+    +--+    +--+    +--+    +--+
           |A1|    |B1|    |C1|    |A2|    |B2|    |C2|
           +--+    +--+    +--+    +--+    +--+    +--+

        ---+-------+-------+-------+-------+-------+------------ Time
           0       Ta      2Ta     3Ta     4Ta     5Ta
```

在这张图中，将发送三个事务（例如，在候选地址收集的情况下，有三个主机候选地址/STUN 服务器对）。它们是事务 A、B 和 C。重传定时器被设置为，在时间 3Ta 处发送第一个事务的第一次重传 (数据包 A2) 。由于 STUN 在其重传上使用指数退避，在第一次重传之后的后续重传发生的频率甚至低于每Ta 毫秒一次。

这种全局最小步调间隔为 5 ms 的机制一般不适用于传输协议，但基于以下原因，它适用于 ICE。

 * 以如下通常适用于传输协议的规则开始：

     1. 令 MaxBytes 为启动时允许的网络在途最大字节数，这应该 (SHOULD) 为 14600，如 [[RFC6928] 第 2 节](https://www.rfc-editor.org/rfc/rfc6928#section-2) 中所定义。

     2. 令 HTO 为事务超时，如果 RTT 已知，这应该 (SHOULD) 为 2*RTT，否则为 500 ms。这基于来自 [[RFC5389](https://www.rfc-editor.org/rfc/rfc5389)] 的 STUN 消息的 RTO 和 TCP 初始 RTO，在 [[RFC6298](https://www.rfc-editor.org/rfc/rfc6298)] 中为 1 秒。

     3. 令 MinPacing 为事务间最小的步调间隔，即 5 ms （见上文）。

 * 请注意，代理通常不知道 ICE 事务的 RTT（特别是连接检查），这意味着 HTO 几乎总是 500 ms。

 * 请注意，5 ms 的 MinPacing 和 500 ms 的 HTO 最多提供 100 个数据包/HTO，对于小于 120 字节的典型 ICE 检查，这意味着网络中最多有 12000 个在途字节，小于由规则 1 表示的最大值。

 * 因此，对于 ICE，规则集简化为仅有 MinPacing 规则，这相当于具有一个全局 Ta 值。

## B.2. 具有多个基的候选地址

第 5.1.3 节讨论了消除具有相同传输地址和基的候选地址。然而，具有相同传输地址但不同基的候选地址不是冗余的。ICE 代理何时可以有两个具有相同 IP 地址和端口但基不同的候选地址呢？考虑图 11 的拓扑：
```
          +----------+
          | STUN Srvr|
          +----------+
               |
               |
             -----
           //     \\
          |         |
         |  B:net10  |
          |         |
           \\     //
             -----
               |
               |
          +----------+
          |   NAT    |
          +----------+
               |
               |
             -----
           //     \\
          |    A    |
         |192.168/16 |
          |         |
           \\     //
             -----
               |
               |
               |192.168.1.100      -----
          +----------+           //     \\             +----------+
          |          |          |         |            |          |
          | Initiator|---------|  C:net10  |-----------| Responder|
          |          |10.0.1.100|         | 10.0.1.101 |          |
          +----------+           \\     //             +----------+
                                   -----
```
图 11：具有不同基的相同候选地址

在这种情况下，发起代理是多主的。它在网络 C 上有一个 IP 地址 10.0.1.100，该网络是一个 net 10 专用网络。响应代理位于相同的网络上。发起代理还连接到网络 A，即 192.168/16，IP 地址为 192.168.1.100。这个网络上有一个 NAT，natting 到网络 B，这是另一个 net 10 私有网络，但它没有连接到网络 C。网络 B 上有一个 STUN 服务器。

发起代理在网络 C 上的 IP 地址上获得一个候选地址 (10.0.1.100:2498)，在网络 A 上的 IP 地址上获得一个候选地址(192.168.1.100:3344) 。它从 192.168.1.100:3344 对其配置的 STUN 服务器执行 STUN 查询。此查询通过 NAT，它恰好分配了绑定 10.0.1.100:2498。STUN 服务器在 STUN 绑定响应中反映了这一点。现在，发起代理已经获得了一个服务器自反候选地址，其传输地址与主机候选地址 (10.0.1.100:2498) 相同。但是，服务器自反候选地址的基为 192.168.1.100:3344，而主机候选地址的基为 10.0.1.100:2498。

## B.3. 相关地址和相关端口属性的目的

候选地址属性包含两个 ICE 本身根本不使用的值 —— 相关地址和相关端口。它们为什么会出现？

包含它们有两个动机。第一个是诊断。了解不同类型候选地址之间的关系很有用。通过包含它，ICE 代理可以知道哪个中继候选地址与哪个自反候选地址相关联，而哪个自反候选地址又与特定的主机候选地址相关联。当对一个候选地址的检查成功了，但对其它候选地址的没有成功时，这提供了网络中所发生的事情有用的诊断。

第二个原因与非路径服务质量 (QoS) 机制有关。在 PacketCable 2.0 等环境中使用 ICE 时，代理除了执行正常的 SIP 操作外，还会检查 SIP 消息中的 SDP 并提取 IP 地址和端口以进行数据流量传输。然后，它们可以通过策略服务器与网络中的接入路由器交互，为数据流建立有保证的 QoS。此 QoS 是通过基于 5 元组对 RTP 流量进行分类，然后为其提供保证速率或适当标记其 DSCP 来提供的。当住宅 NAT 存在，并且中继候选地址被选中用于数据时，这个中继候选地址将是实际 TURN 服务器上的传输地址。该地址没有说明接入路由器中用于对数据包进行分类以进行 QoS 处理的实际传输地址。相反，需要针对 TURN 服务器的服务器自反候选地址。通过在 SDP 中传递转换信息，代理可以使用该传输地址向接入路由器请求 QoS。

## B.4. STUN 用户名的重要性

ICE 需要通过 STUN 使用其短期凭证功能来使用消息完整性。实际的短期凭据是通过在候选地址交换中交换用户名片段形成的。对这种机制的需求不仅仅是为了安全性；它实际上首先是正确操作 ICE 需要。

考虑 ICE 代理 L、R 和 Z。L 和 R 在使用 10.0.0.0/8 的私有企业 1 中。Z 位于也在使用 10.0.0.0/8 的私有企业 2 内。事实证明，R 和 Z 的 IP 地址都是 10.0.1.1。L 发送候选地址给 Z。Z 以它的主机候选地址对 L 做出响应。在这种情况下，这些候选地址是 10.0.1.1:8866 和 10.0.1.1:8877。事实证明，R 同时在一个会话中，并且也使用了 10.0.1.1:8866 和 10.0.1.1:8877 作为主机候选者。这意味着 R 准备好在这些端口上接受 STUN 消息，就像 Z 一样。L 将向 10.0.1.1:8866 发送 STUN 请求，向 10.0.1.1:8877 发送另一个请求。然而，这些并没有像预期的那样到达 Z。相反，它们都去了 R！如果 R 只是回复它们，L 将会认为它与 Z 建立了连接，而实际上它与完全不同的用户 R 建立了连接。为了解决这个问题，使用了 STUN 短期凭证机制。用户名片段足够随机； 因此，R 极不可能使用与 Z 相同的值。因此，R 将因为凭证无效而拒绝 STUN 请求。本质上，STUN 用户名片段提供了一种临时主机标识符，绑定到作为候选地址交换的一部分而建立的特定会话。

IP 地址不唯一性的一个不幸后果是，在上面的示例中，R 甚至可能不是 ICE 代理。它可能是任何主机，STUN 数据包指向的端口可以是该主机上的任何临时端口。如果有应用程序在此套接字上监听数据包，并且它不准备为正在使用的随便什么协议处理格式错误的数据包，则该应用程序的操作可能会受到影响。幸运的是，由于交换的端口是临时的，并且通常来自动态的或注册的范围，因此该端口很有可能不用于在主机 R 上运行服务器，而是用于某些协议的代理端。由于这个范围内的端口使用的瞬态特性，这降低了命中已分配端口的可能性。但是，出问题的可能性确实存在，网络部署人员需要为此做好准备。请注意，这不是 ICE 特有的问题； 对于任何类型的协议，尤其是公共 Internet 上的协议，乱入的数据包随时可能到达端口。因此，此要求只是重申了 Internet 应用程序的一般设计指南——为任何端口上的未知数据包做好准备。

## B.5. 候选地址对优先级公式

候选对的优先级具有奇数形式。它是：
```
      pair priority = 2^32*MIN(G,D) + 2*MAX(G,D) + (G>D?1:0)
```

为什么会是这个？当基于这个值给候选地址对排序时，生成的排序具有 MAX/MIN 属性。这意味着首先基于两个优先级中最小的那个对候选地址对降序排序。对于具有相同最小优先级值的对，使用最大优先级在它们之间进行排序。如果最大和最小优先级相同，则控制代理的优先级用作表达式最后部分的决定性值。使用 2*32 的因子是因为单个候选地址的优先级总是小于 2*32，导致候选地址对的优先级是两个组件优先级的“串联”。这将创建 MAX/MIN 排序。 MAX/MIN 确保对于特定的 ICE 代理，在尝试所有较高优先级的候选地址之前，永远不会使用较低优先级的候选地址。

## B.6. 为什么需要保活？

一旦数据开始在一个候选地址对上流动，在会话期间，依然需要在中间 NAT 上保持绑定有效。通常，数据流包本身（例如，RTP）就可以满足这个目标。然而，有些情况值得进一步讨论。首先，在某些 RTP 应用中，例如 SIP，数据流可以 “暂停”。这是通过使用 [RFC 3264](https://www.rfc-editor.org/rfc/rfc3264) [[RFC3264](https://www.rfc-editor.org/rfc/rfc3264)] 中定义的 SDP "sendonly" 或 "inactive" 属性来完成的。[RFC 3264](https://www.rfc-editor.org/rfc/rfc3264) 指示实现在这些情况下停止数据传输。但是，这样做可能会导致 NAT 绑定超时，并且数据将无法暂停。

其次，某些 RTP 有效载荷格式，例如文本对话的有效载荷格式 [[RFC4103](https://www.rfc-editor.org/rfc/rfc4103)]，发送数据包的频率可能太低，以至于间隔超过 NAT 绑定超时。

第三，如果使用静音抑制，长时间的静音可能会导致数据传输停止较长的时间，以致于 NAT 绑定超时。

由于这些原因，不能依赖数据包本身。ICE 使用 STUN 绑定指示定义了一个简单的周期性保活。这使得它的带宽需求高度可预测，因此可以接受 QoS 预留。

## B.7. 为什么倾向于对等方自反候选地址？

[第 5.1.2 节](https://www.rfc-editor.org/rfc/rfc8445.html#section-5.1.2) 描述了根据候选地址的类型和本地偏好计算候选地址优先级的过程。该节要求对等方自反候选地址的类型偏好总是高于服务器自反的。为什么是那样？原因与[第 19 节](https://www.rfc-editor.org/rfc/rfc8445.html#section-19) 中的安全注意事项有关。相对来说，攻击者让 ICE 代理使用一个错误的服务器自反候选地址比一个错误的对等方自反候选地址要容易得多。因此，ICE 通过优先选择对等方自反候选地址来阻止通过绑定请求针对地址收集进行的攻击。

## B.8. 为什么使用绑定指示来保活？

数据保活在 [第 11 节](https://www.rfc-editor.org/rfc/rfc8445.html#section-11) 中描述。当两个端点都支持 ICE 时，这些保活使用 STUN。 但是，保活不使用绑定请求事务（生成响应），而是使用绑定指示。这是为什么呢？

主要原因与网络 QoS 机制有关。一旦数据开始流动，网络元素将假定数据流具有相当规则的结构，以固定间隔使用周期性数据包，并可能出现抖动。如果 ICE 代理正在发送数据包，然后接收到绑定请求，则需要与其数据包一起生成响应包。这将增加携带数据包的 5 元组的实际带宽需求，并在这些数据包的传递中引入抖动。分析表明，在某些对数据使用相当严格的数据包调度程序的第 2 层接入网络中，这是一个问题。

此外，使用绑定指示允许禁用完整性，这可能会带来更好的性能。这对于大型端点非常有用，例如公共交换电话网络 (PSTN) 网关和会话边界控制器 (SBC)。

## B.9. 选择候选地址类型偏好

选择类型和本地偏好值的一个标准是数据中间设备的使用，例如 TURN 服务器、隧道服务（例如 VPN 服务器）或 NAT。通过数据中间设备，如果数据被发送到该候选地址，它将在被接收之前首先通过数据中间设备。涉及数据中间设备的一类候选地址是中继候选地址。另一种类型是主机候选地址，它是从 VPN 接口获得的。当数据通过数据中间设备传输时，它可能对传输和接收之间的延迟产生好积极的或消极的影响。它可能会或也可能不会增加数据包的丢失，由于可能采用的额外的路由器跳数。它可能会增加提供服务的成本，由于数据将被路由进出由提供商运行的数据中间设备。如果这些问题很重要，则需要仔细选择中继候选地址的类型偏好。

选择偏好的另一个标准是 IP 地址族。ICE 适用于 IPv4 和 IPv6。它提供了一种转换机制，允许双栈主机优先选择连接 IPv6，但在 v6 网络断开连接的情况下回退到 IPv4。实现应该遵循 [[RFC8421](https://www.rfc-editor.org/rfc/rfc8421)] 的指导，以避免在存在损坏路径的情况下，在连接检查阶段出现过度延迟。

选择偏好的另一个标准是拓扑结构。这对使用中间设备的候选地址是有益的。在这些情况下，如果 ICE 代理已经预先配置或动态发现了中间设备与其自身的拓扑邻近性的知识，它可以使用它，来为从更接近的中间设备获得的候选地址分配更高的本地偏好。

选择偏好的另一个标准可能是安全性或隐私性。如果用户是远程办公者，因此连接到公司网络和本地家庭网络，则用户可能更喜欢通过 VPN 或类似隧道路由它们的语音流量，以便在企业内部通信时将其保留在公司网络上，但在与企业外部的用户通信时可能会使用本地网络。在这种情况下，VPN 地址将比任何其它地址具有更高的本地优先级。

# 附录 C. 连接性检查带宽

下表显示了 IPv4 和 IPv6 执行连接检查所需的带宽，使用不同的 Ta 值（以 ms 为单位）和不同的 ufrag 大小（以字节为单位）。

结果由 Justin Uberti (Google) 于 2016 年 4 月 11 日提供。
```
                     IP version: IPv4
                     Packet len (bytes): 108 + ufrag
                          |
                       ms |     4     8    12    16
                     -----|------------------------
                      500 | 1.86k 1.98k 2.11k 2.24k
                      200 | 4.64k 4.96k 5.28k  5.6k
                      100 | 9.28k 9.92k 10.6k 11.2k
                       50 | 18.6k 19.8k 21.1k 22.4k
                       20 | 46.4k 49.6k 52.8k 56.0k
                       10 | 92.8k 99.2k  105k  112k
                        5 |  185k  198k  211k  224k
                        2 |  464k  496k  528k  560k
                        1 |  928k  992k 1.06M 1.12M
```

```
                     IP version: IPv6
                     Packet len (bytes): 128 + ufrag
                          |
                       ms |     4     8    12    16
                     -----|------------------------
                      500 | 2.18k  2.3k 2.43k 2.56k
                      200 | 5.44k 5.76k 6.08k  6.4k
                      100 | 10.9k 11.5k 12.2k 12.8k
                       50 | 21.8k 23.0k 24.3k 25.6k
                       20 | 54.4k 57.6k 60.8k 64.0k
                       10 |  108k  115k  121k  128k
                        5 |  217k  230k  243k  256k
                        2 |  544k  576k  608k  640k
                        1 | 1.09M 1.15M 1.22M 1.28M
```
图 12：连接性检查带宽

# 致谢

本文档中的大部分文本来自原始 ICE 规范 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245)。作者要感谢为该文档做出贡献的所有人。对于本规范修订版的其它贡献，我们要感谢 Emil Ivov、Paul Kyzivat、Pal-Erik Martinsen、Simon Perrault、Eric Rescorla、Thomas Stach、Peter Thatcher、Martin Thomson、Justin Uberti、Suhas Nandakumar、Taylor Brandstetter、 Peter Saint-Andre、Harald Alvestrand 和 Roman Shpount。Ben Campbell 进行了 AD 审查。Stephen Farrell 进行了 sec-dir 审查。Stewart Bryant 进行了 gen-art 审查。Qin We 进行了 ops-dir 审查。Magnus Westerlund 进行了 tsv-art 审查。

# 作者的地址
```
   Ari Keranen
   Ericsson
   Hirsalantie 11
   02420 Jorvas
   Finland

   Email: ari.keranen@ericsson.com


   Christer Holmberg
   Ericsson
   Hirsalantie 11
   02420 Jorvas
   Finland

   Email: christer.holmberg@ericsson.com


   Jonathan Rosenberg
   jdrosen.net
   Monmouth, NJ
   United States of America

   Email: jdrosen@jdrosen.net
   URI:   http://www.jdrosen.net
```

原文：[Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal](https://www.rfc-editor.org/rfc/rfc8445.html)
