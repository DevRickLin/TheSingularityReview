# A2A 与 MCP：引领智能体互联的协议之争？

## 引言：互操作性——智能体生态的关键挑战

人工智能 Agent 的浪潮正汹涌而来，有望自动化处理日益复杂的任务。然而，一个巨大的障碍阻碍着它们的潜力释放：不同开发者、不同框架、甚至不同供应商构建的 Agent 往往彼此隔离，形成一个个"孤岛"，难以有效协作。就像早期互联网需要统一的 HTTP 协议来连接信息孤岛一样，当前的 AI 生态迫切需要标准化的"语言"来实现 Agent 之间的互操作性。

近期，两大科技巨头分别推出了旨在解决这一问题的协议：Google 开源了其 Agent2Agent (A2A) 协议 [1]，而 Anthropic 早些时候也发布了 Model Context Protocol (MCP) [5]。这两大协议的出现，连同其他社区或研究驱动的努力，标志着业界正集中力量攻克 Agent 互操作性难题，为构建未来以智能体为中心的互联网奠定基础。

## Google A2A 协议详解：为异构 Agent 搭建沟通桥梁

Google 的 A2A 协议 [1] 旨在成为连接不同 Agent 生态系统的实用桥梁。它并非试图重新发明轮子，而是建立在成熟、开放的 Web 标准（如 HTTP 和 JSON）之上，降低了采用门槛。其核心目标是让独立的 Agent 应用能够：

1. **相互发现与理解：** Agent 如何知道另一个 Agent 的存在以及它能做什么？
2. **协商交互方式：** 如何确定是通过文本、表单还是更复杂的流媒体进行沟通？
3. **执行和管理任务：** 如何启动、跟踪和完成一个跨 Agent 的任务？

为了实现这些目标，A2A 定义了一系列核心概念和交互模式：

* **Agent Card (智能体名片):** 这是 A2A 的服务发现机制。它是一个公开的元数据文件（通常位于服务器的 `/.well-known/agent.json` 路径下），描述了 Agent 的基本信息、能力（Skills）、API 端点 URL 以及所需的认证方式。客户端或其他 Agent 可以通过获取 Agent Card 来了解目标 Agent 并决定如何与其交互。
* **A2A Server & Client:** 任何实现了 A2A 协议 HTTP 端点的 Agent 应用都可以作为 A2A Server。而调用这些端点的应用或 Agent 则扮演 A2A Client 的角色。
* **Task (任务):** 这是 Agent 间协作的核心单元。客户端通过向服务器发送 `tasks/send` (同步或短时任务) 或 `tasks/sendSubscribe` (长时或流式任务) 请求来启动一个 Task。每个 Task 拥有唯一的 ID，并经历明确定义的生命周期状态，如 `submitted` (已提交), `working` (处理中), `input-required` (需要输入), `completed` (已完成), `failed` (失败), `canceled` (已取消)。这使得任务的跟踪和管理变得标准化。
* **Message & Part (消息与部件):** 通信通过 Message 在 Client (`role: "user"`) 和 Server (`role: "agent"`) 之间进行。每个 Message 由一个或多个 Part 组成。Part 是承载内容的基本单元，可以是：
  * `TextPart`: 纯文本内容。
  * `FilePart`: 文件内容，可以通过内联字节数据或 URI 引用。
  * `DataPart`: 结构化的 JSON 数据，常用于表单提交或返回结构化结果。
* **Artifact (产物):** Agent 在执行任务过程中可能生成文件、最终的结构化数据等输出，这些被称为 Artifact。Artifact 也由 Part 组成。
* **Streaming (流式传输):** 对于需要持续更新或长时间运行的任务 (例如，生成报告、处理视频)，支持 `streaming` 能力的 A2A Server 可以使用 `tasks/sendSubscribe` 端点。客户端会收到 Server-Sent Events (SSE)，实时获取 `TaskStatusUpdateEvent` (任务状态更新) 或 `TaskArtifactUpdateEvent` (任务产物更新) 事件。
* **Push Notifications (推送通知):** 对于无法保持长连接的场景，支持 `pushNotifications` 的服务器允许客户端通过 `tasks/pushNotification/set` 提供一个 webhook URL，以便服务器在任务状态更新时主动推送通知。

**典型交互流程:**

1. **发现 (Discovery):** 客户端从服务器的周知 URL (`/.well-known/agent.json`) 获取 Agent Card。
2. **启动 (Initiation):** 客户端根据 Agent Card 的信息，发送一个包含初始用户消息和唯一 Task ID 的 `tasks/send` 或 `tasks/sendSubscribe` 请求。
3. **处理 (Processing):**
   * **流式:** 服务器通过 SSE 发送状态和产物更新事件。
   * **非流式:** 服务器同步处理任务，并在最终响应中返回 Task 对象。
4. **交互 (Interaction, 可选):** 如果任务进入 `input-required` 状态，客户端使用相同的 Task ID 发送后续消息。
5. **完成 (Completion):** 任务最终达到 `completed`, `failed`, 或 `canceled` 状态。

Google 不仅提出了协议规范（以 JSON Schema 定义），还提供了 Python 和 JavaScript 的客户端/服务器参考实现，以及将 A2A 集成到 Agent Developer Kit (ADK)、CrewAI、LangGraph、Genkit 等流行 Agent 框架的示例 [1]，极大地推动了协议的落地和验证。

## Anthropic MCP 协议详解：规范模型与工具的交互

与 A2A 不同，Anthropic 推出的 Model Context Protocol (MCP) [5] 主要聚焦于**标准化大型语言模型 (LLM) 与其使用的外部工具 (Tools/APIs) 之间的交互过程**。在 Agent 应用中，LLM 常常扮演"大脑"的角色，需要调用各种工具来获取信息或执行操作。MCP 的目标就是让这个调用过程更安全、更可靠、更标准化。

MCP 的核心设计围绕以下几点：

* **结构化通信:** 定义了一套基于 JSON 的消息格式，用于模型向工具发起请求 (包含工具名称、输入参数) 和工具向模型返回结果 (成功响应、错误信息)。
* **上下文管理:** MCP 允许在模型和工具的交互中传递和管理上下文信息，这对于需要多轮交互或保持状态的工具调用至关重要。
* **安全与权限:** 协议的设计考虑了安全因素，允许定义工具的使用权限和范围，防止模型滥用工具或访问未授权的功能。虽然具体的安全实现可能需要上层应用配合，但协议本身提供了基础框架。
* **模型无关性:** 尽管由 Anthropic 提出，MCP 的设计旨在独立于特定的 LLM，理论上可以被任何需要与外部工具交互的模型或 Agent 框架采用。

简单来说，MCP 像是为 LLM 提供了一个标准的"插座"，让各种"电器"（工具/API）能够以统一、安全的方式接入。它解决了 LLM 直接调用多样化、非标准化 API 可能带来的混乱和安全风险。虽然其直接目标是模型-工具交互，但通过规范 Agent 核心"大脑"与外部能力的连接方式，MCP 同样是以 Agent 为中心的互联网生态中不可或缺的一环，确保了 Agent 能够有效、安全地利用外部资源。

## 协议比较：A2A 与 MCP 的定位差异

尽管目标都是促进 AI 生态的互联互通，A2A 和 MCP 的定位和侧重点有着明显的不同：

* **交互主体:**

  * **A2A:** 主要面向**Agent 与 Agent 之间的通信**。它假设通信双方都是相对独立的、具备一定自主能力的 Agent 应用。
  * **MCP:** 主要面向**模型 (LLM) 与工具 (API) 之间的通信**。它更关注模型作为"调用者"与外部功能作为"被调用者"之间的接口规范。
* **核心解决的问题:**

  * **A2A:** 解决异构 Agent 间的**发现、能力协商、任务流程管理**问题。
  * **MCP:** 解决模型调用外部工具时的**接口标准化、上下文管理、安全调用**问题。
* **范围与层次:**

  * **A2A:** 更侧重于 Agent 间协作的**业务流程和任务生命周期管理**。
  * **MCP:** 更侧重于模型进行**单次或多次工具调用的具体实现细节**。
* **技术基础:**

  * **A2A:** 明确基于 HTTP/JSON 等通用 Web 标准。
  * **MCP:** 协议本身是基于 JSON 的消息规范，具体的传输机制相对灵活。

**互补性:** A2A 和 MCP 并非完全对立，反而具有很强的互补性。在一个复杂的 Agent 系统中，一个 Agent (遵循 A2A 与其他 Agent 通信) 的内部可能包含一个 LLM，这个 LLM 在执行任务时需要调用外部工具 (遵循 MCP)。因此，A2A 可以看作是 Agent 间的"外交协议"，而 MCP 则是 Agent (或其内部 LLM) 使用自身"工具箱"的"操作手册"。两者结合，才能构建出既能相互协作，又能有效利用外部能力的强大 Agent 系统。

## 其他值得关注的协议

除了 Google 的 A2A 和 Anthropic 的 MCP，还有一些其他的协议和标准也在探索 Agent 通信的可能性，尽管目前的推动力度或社区关注度可能有所不同：

* **Agent Network Protocol (ANP):** 这是一个由社区驱动的开源项目 [6]，其愿景更为宏大，旨在定义 Agent 如何相互连接，构建一个开放、安全、高效、容纳数十亿智能体的协作网络。ANP 特别强调网络层面的身份认证、端到端加密通信以及 Agent 发现机制，试图构建 Agent 互联网的底层协议栈。
* **Agora:** 来自研究领域 [7]，Agora 提出了一种用于大型语言模型网络的可扩展通信协议，旨在通过精心设计的通信策略和网络拓扑，让 Agent 网络能够高效地协同解决复杂问题。

这些协议代表了从不同角度（如底层网络安全、大规模网络效率）对 Agent 通信标准化的探索，它们的发展同样值得关注。

## 协议之上：通往以智能体为中心的互联网

标准化通信协议，特别是像 A2A 和 MCP 这样获得主要厂商支持的协议的出现，正是构建"以 Agent 为中心的互联网" (Agent-Centric Internet) 的关键基石。这个常被称为 Web 4.0 或"代理式网络 (Agentic Web)" [2] 的概念，代表了互联网范式的根本转变：

* **从"人机交互"到"机机交互"为主:** 未来的互联网将不仅仅是人类通过浏览器访问信息的界面。自主、智能的软件 Agent 将代表用户或组织，在网络上自动执行任务、获取信息、进行交互 [3]。网站和应用可能需要提供专门面向 Agent 的接口 (APIs) 或视图，甚至演变为仅提供 API 的"幽灵厨房" [4]。
* **智能与自主性:** Agent 不再是简单的脚本，而是由 AI 驱动，能够理解复杂指令，感知环境，自主决策并适应变化 [2]。
* **多 Agent 协作:** 复杂任务将由多个专门的 Agent 协同完成，形成"心智社会 (Society of Mind)" [2]。A2A 提供了 Agent 间协作的"共同语言"，而 MCP 确保了 Agent 内的"大脑"能有效调用工具。
* **用户代理:** Agent 的核心是代表用户行事，根据用户的偏好和目标过滤信息、完成任务、管理数字生活，极大地提升效率 [2]。

没有像 A2A 这样的 Agent 间通信标准，以及 MCP 这样的模型-工具交互标准，Agent 的协作和能力扩展将困难重重，成本高昂，大规模的 Agent 网络也无从谈起。这些协议解决了 Agent 间如何沟通 (How) 以及 Agent 如何利用外部能力 (How) 的问题，为实现以 Agent 为中心的互联网的各种特性 (What) 铺平了道路。

## 总结与展望

Google A2A 和 Anthropic MCP 协议的发布是推动 Agent 互操作性标准化的重要里程碑。它们分别从 Agent 间协作和模型-工具交互两个关键维度切入，为解决当前 AI Agent 生态的"孤岛效应"提供了切实可行的方案。A2A 以其开放性和实用性，有望加速异构 Agent 生态的融合；而 MCP 则为 Agent 的核心能力——利用 LLM 和外部工具——提供了规范。虽然未来哪个协议会成为主导，或者它们将如何融合演进尚不可知，但它们共同指向了一个更加互联、智能和自动化的互联网未来。协议的标准化只是开始，围绕协议的生态建设、安全标准的完善、以及更复杂协作模式的探索，将是接下来的关键。

# 关于《奇点评论》

《奇点评论》是一个**正在尝试以去中心化、开源方式共建**的评论专栏，致力于追踪和探讨人工智能、前沿科技及其对社会未来可能产生的深远影响。我们关注技术突破、行业趋势、伦理挑战以及那些预示着"奇点"临近的信号。**我们相信，通过社区的集体智慧和开放讨论**，才能更全面地理解和引导这场变革。

**作为一个开放共建的平台**，我们诚挚欢迎来自研究者、开发者、创业者、思考者以及所有对科技未来充满好奇心的朋友们**参与贡献**。如果您有独特的见解、深入的分析或相关的实践经验愿意分享，请通过 [https://github.com/DevRickLin/TheSingularityReview](https://github.com/DevRickLin/TheSingularityReview) 与我们联系。让我们共同记录、思考并塑造这个加速变化的时代。

## 参考文献

1. google/A2A GitHub Repository. [https://github.com/google/A2A](https://github.com/google/A2A)
2. A Brief Exploration of Web 4.0: The Agent-Centric Internet | by Chain | Medium. [https://medium.com/@chaincom/a-brief-exploration-of-web-4-0-the-agent-centric-internet-b7dbcf3822df](https://medium.com/@chaincom/a-brief-exploration-of-web-4-0-the-agent-centric-internet-b7dbcf3822df)
3. The end of the human-centric web. Why AI agents will take over online… | by Christian Freischlag | Medium. [https://medium.com/@christian.freischlag/the-end-of-the-human-centric-web-87703e4e6421](https://medium.com/@christian.freischlag/the-end-of-the-human-centric-web-87703e4e6421)
4. The future of the internet - Lina's Portfolio. [https://linacolucci.com/2023/09/the-future-of-the-internet/](https://linacolucci.com/2023/09/the-future-of-the-internet/)
5. Introducing the Model Context Protocol - Anthropic. [https://www.anthropic.com/news/model-context-protocol](https://www.anthropic.com/news/model-context-protocol) (See also: [https://docs.anthropic.com/en/docs/agents-and-tools/mcp](https://docs.anthropic.com/en/docs/agents-and-tools/mcp))
6. agent-network-protocol/AgentNetworkProtocol GitHub Repository. [https://github.com/agent-network-protocol/AgentNetworkProtocol](https://github.com/agent-network-protocol/AgentNetworkProtocol)
7. A Scalable Communication Protocol for Networks of Large Language Models - arXiv. [https://arxiv.org/html/2410.11905v1](https://arxiv.org/html/2410.11905v1)

---

*本文由人类与 AI 协作编辑完成，内容力求准确，但难免疏漏，欢迎读者勘误指正。*
