# 18. IAB 注意事项

IAB 研究了 “单边自地址修复”（UNSAF）问题，这是 ICE 代理尝试通过协作协议反射机制确定在 NAT 另一侧的另一个域其地址的一般过程 [ [RFC3424](https://www.rfc-editor.org/rfc/rfc3424) ]。ICE 是执行此类功能的协议的一个示例。有趣的是，ICE 的过程不是单边的，而是双边的，并且这种差异对 IAB 提出的问题有重大影响。实际上，ICE 可以被视为一个 双边自地址修复 (B-SAF) 协议，而不是 UNSAF 协议。无论如何，IAB 已强制要求为此目的开发的任何协议都记录一组特定的注意事项。本节满足这些要求。

## 18.1. 问题定义

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 精确定义将通过 UNSAF 提案解决的特定、范围有限的问题。不应将短期解决方案推广到解决其他问题。这种泛化导致对提议的短期修复的长期依赖和使用 —— 这意味着将其称为 “短期” 不再准确。
 
ICE正在解决的具体问题是：

> 为两个对等方提供一种确定可用于通信的传输地址集的方法。

> 为代理提供一种方法来确定它希望与之通信的另一个对等方可到达的地址。

## 18.2. 退出策略

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 退出策略/过渡计划的描述。更好的短期修复是那些随着适当技术的部署自然地越来越少使用的修复。

ICE 本身并不容易被淘汰。但是，即使在全球连接的互联网中，它也很有用，例如，它可以用作检测路由器故障是否暂时中断连接的手段。ICE 还有助于防止某些与 NAT 无关的安全攻击。然而，ICE 所做的是帮助逐步淘汰其他 UNSAF 机制。ICE 有效地在这些机制中进行挑选，提高更好的机制的优先级，而降低更差的机制的优先级。随着 IPv6 的引入，NAT 开始消散，服务器自反和中继候选地址（两种形式的 UNSAF 地址）根本不会被使用，因为本地主机候选地址存在更高优先级的连接。因此，服务器的使用越来越少，最终可以在它们的使用量变为零时被删除。

事实上，ICE 可以帮助从 IPv4 过渡到 IPv6。当两台双栈主机与 SIP 通信（使用 IPv6）时，它可用于确定是使用 IPv6 还是使用 IPv4。它还可以允许具有 6to4 和本机 v6 连接的网络确定在与对等方通信时使用哪个地址。

## 18.3. ICE 引入的脆弱性

根据 [RFC 3424](https://www.rfc-editor.org/rfc/rfc3424)，任何 UNSAF 提案都需要提供：

> 讨论可能使系统更加 “脆弱” 的具体问题。例如，涉及在多个网络层使用数据的方法会产生更多的依赖，增加调试挑战，并使其更难转换。

ICE 实际上消除了现有 UNSAF 机制的脆弱性。特别是，经典 STUN（如 [RFC 3489](https://www.rfc-editor.org/rfc/rfc3489) [[RFC3489](https://www.rfc-editor.org/rfc/rfc3489)] 中所述）有几个脆弱点。其中一个是发现过程，该过程需要 ICE 代理尝试对其身处其后的 NAT 的类型进行分类。这个过程很容易出错。使用 ICE，根本不使用该发现过程。不是单方面评估地址的有效性，而是通过测量与对等方的连接性来动态确定其有效性。确定连接性的过程非常健壮。

经典 STUN 和任何其他单边机制的另一个脆弱点是它对附加服务器的绝对依赖。ICE 使用服务器来分配单边地址，但如果可能，它允许代理直接连接。因此，在某些情况下，即使 STUN 服务器发生故障，使用 ICE 时仍然允许进行通话。

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

攻击者可能会尝试破坏 STUN 连接检查。最终，所有这些攻击都会欺骗 ICE 代理，使其对连接检查的结果有不正确的认知。根据攻击的类型，攻击者需要具备不同的能力。在某些情况下，攻击者需要在连接检查的路径上。在其他情况下，攻击者不需要在路径上，只要它能够生成 STUN 连接检查就可以。尽管对连接检查的攻击通常由网络实体执行，但如果攻击者能够控制端点，它可能能够触发连接检查攻击。攻击者可以尝试并导致的可能错误结论是：

错误的无效：攻击者可以欺骗一个代理对使其认为一个候选地址对是无效的，而实际上并非如此。这可被用于使代理优先一个不同的候选地址（例如由攻击者注入的候选地址）或通过强制所有候选地址失败来中断呼叫。

错误的有效：攻击者可以欺骗一个代理对使其认为一个候选地址对是有效的，而实际上并非如此。这可能会导致代理继续进行会话，但随后无法接收任何数据。

错误的对等方自反候选地址：攻击者可以使代理发现一个新的对等方自反候选地址，而这是不期望的。这可用于将数据流重定向到 DoS 目标或攻击者，用于窃听或其他目的。

错误的候选地址之上的错误有效：攻击者已经让代理相信有一个候选地址具有一个实际上不路由到该代理的地址（例如，通过注入虚假的对等方自反候选地址或虚假的服务器自反候选地址）。然后攻击者发起攻击，迫使代理相信这个候选地址是有效的。

如果攻击者可以导致错误的对等方自反候选地址或错误的候选地址之上的错误的有效，它可以发起 [[RFC5389](https://www.rfc-editor.org/rfc/rfc5389)] 中描述的任何攻击。

为了强制错误的无效结果，攻击者必须等待来自其中一个代理的连接检查被发送。如果是这样，攻击者需要注入一个带有不可恢复错误响应（例如 400）的虚假响应，或者丢弃响应以使其永远不会到达代理。但是，由于候选地址实际上是有效的，原始请求可能会到达对等方代理并导致成功响应。攻击者需要通过 DoS 攻击、第 2 层网络中断或其他技术强制丢弃此数据包或其响应。如果它不这样做，成功响应也将到达发起者，提醒它可能的攻击。攻击者生成虚假响应的能力通过 STUN 短期凭证机制得到缓解。为了使这个响应被处理，攻击者需要密码。如果候选地址交换信令是安全的，则攻击者将无法拥有密码，且它的响应将被丢弃。

欺骗性 ICMP 硬错误（类型 3，代码 2 - 4）也可用于创建错误的无效结果。如果 ICE 代理对这些 ICMP 错误做出响应，则攻击者能够生成 ICMP 消息，该消息将传递给发送连接检查的代理。代理对 ICMP 错误消息的验证是其唯一的防御措施。对于类型 3 代码=4，外部 IP 标头不提供验证，除非连接检查发送时设置 DF=0。对于由主机发起的代码 2 或 3，其地址应为远程代理的主机、自反或中继候选 IP 地址中的任何一个。ICMP 消息包括触发错误的消息的 IP 头和 UDP 头。这些字段也需要验证。IP 目标和 UDP 目标端口需要匹配目标候选地址和端口或候选地址的基地址。源 IP 地址和端口可以是发送连接检查的代理的相同基地址的任何候选地址。因此，任何有权访问候选地址交换的攻击者都将拥有必要的信息。因此，验证是一个薄弱的防御，并且没有源地址验证的话，来自网络中的节点的非路径攻击者发送欺骗性的 ICMP 攻击也是可能的。

强制伪造的有效结果以类似的方式工作。攻击者需要等待来自每个代理的绑定请求并注入一个虚假的成功响应。同样，由于 STUN 短期凭证机制，为了让攻击者注入有效的成功响应，攻击者需要密码。或者，攻击者可以将通常会被网络丢弃或拒绝的有效成功响应路由（例如，使用隧道机制）到代理。

可以通过虚假请求或响应或重放来强制错误的对等方自反候选地址结果。我们首先考虑虚假请求和响应的情况。它要求攻击者向一个代理发送绑定请求，其中包含虚假候选地址的源 IP 地址和端口。此外，攻击者需要等待来自其他代理的绑定请求，并生成一个带有包含虚假候选地址的 XOR-MAPPED-ADDRESS 属性的虚假响应。与此处描述的其他攻击一样，STUN 消息完整性机制和安全的候选地址交换可以缓解这种攻击。

使用数据包重放强制错误的对等方自反候选地址结果是不同的。攻击者一直等到其中一个代理发送检查。它拦截此请求并以伪造的源 IP 地址将其重放给其他代理。它还需要防止原始请求到达远程代理，方法是发起 DoS 攻击以导致数据包被丢弃或使用第 2 层机制强制丢弃它。由于完整性检查通过（完整性检查不能也不会覆盖源 IP 地址和端口），因此重放的数据包在另一个代理处被接收并被接受。然后对其进行响应。该响应将包含一个带有错误候选地址的 XOR-MAPPED-ADDRESS，并将发送给该错误候选地址。然后，攻击者需要接收它并将其重放给发起者。

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

STUN 放大攻击类似于 “语音锤” 攻击，其中攻击者使其他代理将语音数据包定向到攻击目标。但是，不是将语音数据包定向到目标，而是将 STUN 连接检查定向到目标。攻击者发送大量候选地址，例如，50 个。响应代理接收候选地址信息并启动它的被定向到目标的检查，因此，永远不会产生响应。在 WebRTC 的情况下，用户甚至可能不知道这种攻击正在进行，因为它可能是由用户获取的恶意 JavaScript 代码在后台触发的。应答者将每隔 Ta ms 开始一次新的连接检查（例如，Ta=50ms）。然而，由于大量的候选地址，重传定时器被设置为很大的数字。因此，数据包将以每 Ta 毫秒一个的间隔发送，然后间隔增加。因此，STUN 不会以比数据发送速度更快的速率发送数据包，并且 STUN 数据包只会短暂持续，直到 ICE 会话失败。尽管如此，这是一种放大机制。

消除放大是不可能的，但可以通过各种启发式方法来减小数量。ICE 代理应该 (SHOULD) 将它们执行的连接检查的总数限制为 100。此外，代理可以 (MAY) 限制他们接受的候选地址的数量。

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

本文档中的大部分文本来自原始 ICE 规范 [RFC 5245](https://www.rfc-editor.org/rfc/rfc5245)。作者要感谢为该文档做出贡献的所有人。对于本规范修订版的其他贡献，我们要感谢 Emil Ivov、Paul Kyzivat、Pal-Erik Martinsen、Simon Perrault、Eric Rescorla、Thomas Stach、Peter Thatcher、Martin Thomson、Justin Uberti、Suhas Nandakumar、Taylor Brandstetter、 Peter Saint-Andre、Harald Alvestrand 和 Roman Shpount。Ben Campbell 进行了 AD 审查。Stephen Farrell 进行了 sec-dir 审查。Stewart Bryant 进行了 gen-art 审查。Qin We 进行了 ops-dir 审查。Magnus Westerlund 进行了 tsv-art 审查。

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
