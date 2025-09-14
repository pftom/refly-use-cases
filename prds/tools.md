## 背景

当前 Refly 的工具体系主要依赖 MCP 协议，包括两大类：

- **内置 MCP**：将 Refly 内部接口通过 @rekog/mcp-nest 暴露为 MCP Server，然后通过 localhost 连接来调用工具
- **用户配置 MCP**：用户填写 MCP Server 配置，目前由于架构限制仅支持 SSE、Streamable 类型的 MCP Server，数量非常少，主流的 Stdio 类型仅桌面应用、CLI 程序能较好支持

Tools 2.0 将包括以下类型的工具：

| **工具类型** | **描述** | **是否需要安装配置** |
| --- | --- | --- |
| 内部工具 | 将 Refly 系统级能力作为工具，例如画布、资源、文档、代码组件的增删改查、生图生视频、发送邮件等，通过依赖注入的方式提供给模型调用 | No |
| 常规工具 | LangChain 兼容的工具，存放在开源代码仓库内，支持用户手动或者通过 OAuth 流程来配置鉴权信息 | Yes，Cloud 版本会提供一些无需安装的全局工具 |
| MCP | 兼容已有的 MCP Server 配置体系，复用已有的 MCP 社区 | Yes |

详尽对比：

| **工具 1.0 体系** | **工具 2.0 体系** |
| --- | --- |
| ❌ 开箱即用的外部工具太少，配置繁琐❌ 内部工具比较少，需要逐一进行 MCP 适配，而且需要走 localhost 网络调用❌ 没有与 Agent 系统打通❌ 受限于 MCP 类型支持（仅 SSE 和 Streamable，很多工具无法接入）❌ MCP Server 依赖外部生态，对 Refly 开源很难形成反哺作用❌ 没有与积分体系打通，阻塞部分需要成本的工具接入 | ✅ 提供超过 50 款开箱即用的外部工具，通过 OAuth 的方式快速完成鉴权信息收集✅ 内部工具✅ 与 Agent 系统无缝集成✅ MCP 只是支持工具的一种，支持其他更灵活的工具，包括常规的 Function Calling✅ 通过开源运作的方式吸引外部贡献者添加更多工具，形成良好的开源生态✅ 打通积分体系，能够提供一些高级工具（例如投资圈常用的 whalewisdom） |

对应 PRD [【PRD】Workflow & Tools](https://powerformer.feishu.cn/wiki/KMUowF5Kpidc9PkOKfzciunMnIc)

Figma https://www.figma.com/design/9CGm70STlIlKnWMgpU9kbc/refly-tree?node-id=7336-86149&t=42ZaIL6Q4a3YloJM-0

## 竞品分析

[Untitled](https://www.notion.so/26e162107b43800bb5d4f7bc48d00710?pvs=21)

## 技术调研

Pipedream

> MCP 工具列表 https://mcp.pipedream.com
> 
> 
> 提供了一站式工具配置平台，需要在他们平台上配置后，通过 API Key 来授权和访问
> 
> https://github.com/PipedreamHQ/pipedream components 目录下包含具体 tools 实现
> 
> **结论：比较封闭的工具配置平台，而且是“假开源”，核心的平台代码并不开放、不能私有化部署，不考虑使用**
> 

MetaMCP

> MCP 聚合网关，允许配置任意多的 MCP Server，并通过 SSE 或者 Streamable 进行转发。
> 
> 
> https://github.com/metatool-ai/metamcp
> 
> **结论：比较灵活的 MCP 聚合工具，后续如果要补充内置 MCP 可以考虑（但是走常规工具似乎更好一点）**
> 

Supergateway

> Stdio  类型的 MCP 转发为 SSE、Streamable，仅支持单个 MCP Server 转发
> 
> 
> https://github.com/supercorp-ai/supergateway
> 
> **结论：不好用，一对一转发，不如 MetaMCP**
> 

FastAPI_MCP

> https://github.com/tadata-org/fastapi_mcp
> 
> 
> **结论：将 FastAPI 项目转为 SSE，对于非 FastAPI 项目用处不大。**
> 

LangChain

> Builtin Tools: https://js.langchain.com/docs/integrations/tools/
> 
> 
> Custom Tools:
> 
> ```
> import { DynamicStructuredTool } from "@langchain/core/tools";
> import { z } from "zod";
> 
> const multiplyTool = new DynamicStructuredTool({
>   name: "multiply",
>   description: "multiply two numbers together",
>   schema: z.object({
>     a: z.number().describe("the first number to multiply"),
>     b: z.number().describe("the second number to multiply"),
>   }),
>   func: async ({ a, b }: { a: number; b: number }) => {
>     return (a * b).toString();
>   },
> });
> 
> await multiplyTool.invoke({ a: 8, b: 9 });
> 
> ```
> 
> **结论：与现有的模型调用体系兼容，并且已经包含比较多的 Tool 案例，常规工具都将遵循此标准。**
> 

## 业务流程

整体流程：

1. **配置工具**：从 Tool Store 安装对应的工具，如需授权流程则需完成
2. **指定工具**：在 query 中用自然语言或者 mention 能力指定
    1. 通过 mention 来添加工具时，如果未安装此工具，需要弹出安装提示
3. **Ask/Workflow 单个节点**
    1. **检索工具**：用户指定的工具，同时自动召回最可能使用的工具（宁可多一点，不能缺失关键工具）
    2. **模型调用工具**：通过 bindTools 后让模型能够调用工具，如果配置了计费，则同步计费系统
4. **Agent**
    1. **检索工具**：Planner 在做任务拆解的时候，指定 Sub-agent 可能使用的工具（宁可多一点，不能缺失关键工具）
    2. **模型调用工具**：Sub-agent 通过 bindTools 让模型能够调用工具，如果配置了计费，则同步计费系统

### 工具授权流程

1. 手动填写：用户手动去对应平台获取 API Key，实现简单
2. OAuth 流程（Genspark 采用此流程，体验非常好）**第一期来不及先不做**

暂时无法在飞书文档外展示此内容

### Ask/Workflow 流程

> 帮我画一个 mermaid 时序图，包含几个实体：用户、API、ToolSelector (llm)、Actor Agent (llm)
> 
> 1. 用户选择工具：在 query 中用自然语言或者 mention 能力指定，调用 API
> 2. 检索工具：API 在用户指定的工具的基础上，调用 ToolSelector 召回最可能使用的工具
> 3. 模型调用工具：结合用户选择工具和 ToolSelector 召回工具，通过 bindTools 后让 Actor Agent 模型调用工具
> 4. 同步计费：如果对应的工具配置了计费，则同步 API 计费系统（发送 MQ 消息）

暂时无法在飞书文档外展示此内容

### Agent 流程

> 帮我画一个 mermaid 时序图，包含几个实体：用户、API、Planner Agent(llm)、Actor Agent (llm)
> 
> 1. 发起 Agent 流程：用户调用 API /pilot/session/new，发起新的 Agent 会话
> 2. 激活 Planner：Agent 会话激活 Planner Agent
> 3. 检索工具：Planner 根据上下文，进行任务拆解，并为每个任务选择相应的工具
> 4. Actor Agent 调用工具：Planner 将任务下发 Actor，Actor 通过 bindTools 调用 Planner 选择的工具
> 5. 同步计费：如果对应的工具配置了计费，则同步 API 计费系统（发送 MQ 消息）

暂时无法在飞书文档外展示此内容

### 工具召回流程

Ask/Workflow 和 Agent 的工具召回流程基本相似，包含以下上下文：

| 上下文 | Ask/Workflow | Agent |
| --- | --- | --- |
| 用户 query | ✅ | ✅ |
| 用户指定的工具 | ✅ | ✅ |
| 所有工具的 name 和描述 | ✅ | ✅ |
| 历史任务执行记录 |  | ✅ |

方案一：内置工具 + **用户安装常规工具** + 用户配置 MCP 传给 ToolSelector 判断：实现简单，但是用户可能会缺少安装某个关键工具导致执行效果不符合预期

```
--- 召回用户所有已安装工具
SELECT * FROM refly.tool_instances WHERE uid = 'xxx';

```

方案二：内置工具 + **所有常规工具** + 用户配置 MCP 传给 ToolSelector 判断：能够判断是否缺失关键工具，但是交互可能会比较复杂，需要引入额外的确认流程

## 数据存储

### 工具 & 工具集

单个 Tool 本身不走数据表，通过服务启动时把对应的 class 加载进内存。例如：

```
export class DiscordGetMessagesTool extends Tool {
  static lc_name() {
    return "DiscordGetMessagesTool";
  }

  name = "discord-get-messages";

  description = `A discord tool. useful for reading messages from a discord channel.
  Input should be the discord channel ID. The bot should have read
  permissions for the channel.`;

  protected botToken: string;

  protected messageLimit: number;

  protected client: Client;

  constructor(fields?: DiscordGetMessagesToolParams) {
    super();

    const {
      botToken = getEnvironmentVariable("DISCORD_BOT_TOKEN"),
      messageLimit = 10,
      client,
    } = fields ?? {};

    if (!botToken) {
      throw new Error(
        "Environment variable DISCORD_BOT_TOKEN missing, but is required for DiscordGetMessagesTool."
      );
    }

    this.client =
      client ??
      new Client({
        intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages],
      });

    this.botToken = botToken;
    this.messageLimit = messageLimit;
  }

  /** @ignore */
  async _call(input: string): Promise<string> {
    try {
      await this.client.login(this.botToken);

      const channel = (await this.client.channels.fetch(input)) as TextChannel;

      if (!channel) {
        return "Channel not found.";
      }

      const messages = await channel.messages.fetch({
        limit: this.messageLimit,
      });
      await this.client.destroy();
      const results =
        messages.map((message: Message) => ({
          author: message.author.tag,
          content: message.content,
          timestamp: message.createdAt,
        })) ?? [];

      return JSON.stringify(results);
    } catch (err) {
      await this.client.destroy();
      return "Error getting messages.";
    }
  }
}

```

工具集则是将一批相关的工具进行组合：

```
export class DiscordToolset {
    name = 'discord';
    description = 'Discord related tools';
    tools: Tool[];
}

```

### 工具实例

```
-- 工具实例表
CREATE TABLE tool_instance (
    instance_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- 工具实例 ID
    uid           UUID NOT NULL,                              -- 用户 ID
    tool_name     TEXT NOT NULL,                              -- 工具名称（需在代码库中定义）
    tool_type     TEXT NOT NULL CHECK (tool_type IN ('regular', 'mcp')), -- 工具类型
    auth_type     TEXT CHECK (auth_type IN ('manual', 'oauth')),        -- 鉴权类型
    auth_data     BYTEA,                                      -- 鉴权数据（对称加密存储）
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),         -- 创建时间
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),         -- 更新时间
    deleted_at    TIMESTAMPTZ                                 -- 删除时间（软删除）
);

```

### 工具计费配置

```
-- 工具积分配置表
CREATE TABLE tool_credit_config (
    tool_name   TEXT PRIMARY KEY,                                -- 工具名称（唯一）
    unit        TEXT NOT NULL CHECK (unit IN ('per_use')),       -- 计费单位，目前只支持 per_use
    unit_cost   INTEGER NOT NULL CHECK (unit_cost >= 0),         -- 单位积分消耗，非负数
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),              -- 创建时间
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),              -- 更新时间
    deleted_at  TIMESTAMPTZ                                      -- 删除时间（软删除）
);

```

### 工具使用记录

tool_call_result

```

```

## 工具实现规划（持续更新）

### 内置工具集

| **Toolset** | **包含工具** | **备注** | **进度** |
| --- | --- | --- | --- |
| resource | • getResourceDetail：读取资源详情
• createResource：创建资源
• updateResource：更新资源
• deleteResource：删除资源 |  | **已实现** |
| document | • getDocumentDetail：读取文档详情
• createDocument：创建文档
• updateDocument：更新文档
• deleteDocument：删除文档 |  | **已实现** |
| codeArtifact | • getCodeArtifactDetail：获取代码组件详情
• createDocument：创建代码组件
• updateDocument：更新代码组件
• deleteDocument：删除代码组件 |  |  |
| media | • generateImage：生成图片
• generateVideo：生成视频
• generateAudio：生成音频 |  |  |
| knowledgeBase | • searchKnowledgeBase：知识库搜索
• listKnowledgeBase：获取知识库列表
• createKnowledgeBase：创建知识库
• updateKnowledgeBase：创建知识库
• deleteKnowledgeBase：删除知识库 |  |  |
| email | • sendEmail：发送邮件到注册邮箱 |  |  |

### 常规工具集

> 根据Sim、Wordware、Pokee.ai、Zapier、Deforge、Dify、n8n支持的tools，梳理出来以下常见tools
> 

| **类别** | **Toolset** | **优先级** | **1.0 需要Ready** | **包含工具** | **是否全局配置** | **需要鉴权** | **备注** | **进度** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 信息源与内容获取 | api_call | P2 | 是 | • fetch：通过 API 调用任何 HTTP 接口 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | firecrawl | P0 | 是 | • firecrawl_scrape：爬取指定 URL
• firecrawl_batch_scrape：批量爬取 URL
• firecrawl_search：搜索指定内容
• firecrawl_crawl：爬取指定网站所有内容
• firecrawl_map：输入网站，获取所有 URL（sitemap）
• firecrawl_extract：从指定 URL 提取结构化数据 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | Jina | P0 | 是 | • Web Reader: Extracts content from web pages.
• Web Search: Searches the web for information.
• Fact-Check: Verifies factual statements. | 全局配置 | 不需要授权 |  |  |
| 开发与数据管理 | code_intepreter | P0 | 是 | • runPythonCode：运行 Python 代码
• runJsCode：运行 JavaScript 代码 | 全局配置 | 不需要授权 | **依赖外部服务**Python 沙盒：https://github.com/pyodide/pyodideJS 沙盒：待调研 |  |
| 开发与数据管理 | github | P1 | 是 | • repository：仓库管理
• issue：Issue 管理
• pullRequest：PR 管理
• searchRepo：关键词搜索 GitHub 仓库
• searchCode：搜索仓库代码片段 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | whalewisdom | P0 | 是 |  | 全局配置 | 不需要授权 | 没有MCP，需要接入API |  |
| 信息源与内容获取 | arxiv | P1 | 是 | • arxiv_search：按关键词、作者、标题或其他字段在 ArXiv 上搜索学术论文。
• arxiv_get_paper：通过 ID 获取特定 ArXiv 论文的详细信息。
• arxiv_get_author_papers：在 ArXiv 上搜索特定作者的论文。 | 全局配置 | 不需要授权 |  |  |
| 文件与知识库 | notion | P0 | 是 | • search: 在 Notion 工作区中进行内容搜索
• fetch:  根据页面或数据库的 URL 获取其内容
• create-pages: 创建一个或多个具有指定属性和内容的 Notion 页面。如未指定父级，则创建私有页面。
• update-page:  更新已有 Notion 页面，包括其属性或正文内容。
• move-pages: 将一个或多个页面或数据库移动到新的父级位置。
• duplicate-page: 异步复制现有页面。
• create-database:  创建新的数据库，并设置其属性（如字段等）。
• update-database:  更新数据库的属性、名称、描述或其他属性设置。
• create-comment: 在指定页面上添加评论。
• get-comments: 列出指定页面上的所有评论，包括线程评论。 | 否 | 需要授权 |  |  |
| 文件与知识库 | google_drive | P0 | 是 | • gdrive_search: 在 Google Drive 中执行文件搜索，支持关键词查询、分页和排序。
• gdrive_read_file: 读取指定文件的内容，支持多种文件格式，包括 Google Docs、Sheets 等。
• gsheets_read: 从 Google Sheets 中读取数据，支持指定范围、工作表等。
• gsheets_update_cell: 更新 Google Sheets 中指定单元格的内容。
• gdrive_copy: 复制 Google Drive 中的文件或文件夹。
• gdrive_create_file: 从文本创建新的文件，并上传到指定位置。
• gdrive_delete: 删除 Google Drive 中的文件或文件夹。
• gdrive_download: 下载 Google Drive 中的文件。
• gdrive_move: 移动文件或文件夹到新的位置。
• gdrive_share: 设置文件或文件夹的共享权限。
• gdrive_update: 更新文件或文件夹的元数据，如名称、描述等。
• gdrive_upload: 上传本地文件到 Google Drive。
• gdrive_create_folder: 创建新的文件夹。
• gdrive_delete_folder: 删除文件夹。
• gdrive_share_folder: 设置文件夹的共享权限。
• gdrive_create_shared_drive: 创建新的共享驱动器。
• gdrive_delete_shared_drive: 删除共享驱动器。 | 否 | 需要授权 |  |  |
| 文件与知识库 | google_suite | P1 | 是 | Gmail
• list_emails：列出邮箱内容，支持 Gmail 查询语法过滤
• search_emails：高级搜索邮件
• send_email：发送邮件，支持 CC/BCC
• modify_email：更改邮箱标签（如归档、删除、标记为已读/未读） Google Calendar
• list_events：按日期范围列出日历事件
• create_event：添加新事件，支持设定参与者
• update_event：更新现有事件
• delete_event：删除事件 Google Drive
• drive_search_files：搜索 Drive 文件
• drive_create_folder：创建新文件夹 Google Docs
• docs_create：创建文档
• docs_get_content：获取文档内容
• docs_update_content：更新文档内容 Google Sheets
• sheets_create：创建电子表格
• sheets_get_values：获取单元格值
• sheets_update_values：更新某范围内的单元格值 | 否 | 需要授权 |  |  |
| 文件与知识库 | onedrive | P1 | 是 | • Copy Item：将某个驱动项（文件或文件夹）异步复制到新的位置。
• Create Sharing Link：为指定的驱动项（文件或文件夹，需提供唯一 ID）生成共享链接。
• Delete Item：根据唯一 ID，删除当前用户 OneDrive 中的驱动项（文件或文件夹）。
• Download A File：通过文件的唯一 ID 下载 OneDrive 中的文件（仅支持文件，不支持文件夹）。
• Get Drive：根据唯一 ID 获取某个驱动的属性与关联关系。
• Get Item Metadata：根据唯一 ID 获取某个驱动项的元数据信息。
• Get Item Permissions：通过唯一 ID 或路径，获取某个驱动项的权限信息。
• Get Item Thumbnails：获取指定驱动项关联的缩略图。
• Get Item Versions：根据唯一 ID 获取某个驱动项的历史版本记录。
• Get Drive Quota：获取当前用户 OneDrive 的存储配额信息。
• Get Recent Items：列出当前用户最近使用的项目。
• Get Shared Items：列出已与当前用户共享的项目。
• Get Sharepoint List Items：获取指定 SharePoint 站点中的列表项。
• Get Site Details：根据站点 ID 获取指定 SharePoint 站点的元数据。
• Get Sharepoint Site Page Content：获取现代 SharePoint 站点页面的内容。
• Invite User To Drive Item：邀请用户或为 OneDrive 中的特定项目授予权限。 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | producthunt | P1 | 是 | • get_posts：获取帖子列表。
• get_post_details：获取指定帖子的详细信息。
• get_comments：获取帖子评论。
• get_comment_details：获取指定评论的详细信息。
• get_users：获取用户列表。
• get_user_details：获取指定用户的详细信息。
• get_votes：获取投票信息。
• search_posts：搜索帖子，支持按主题、日期、投票等筛选。
• get_collections：获取集合列表。
• get_collection_details：获取指定集合的详细信息。
• get_topics：获取主题列表。
• get_topic_details：获取指定主题的详细信息。 | 全局配置 | 不需要授权 |  |  |
| 开发与数据管理 | mysql | P2 | 是 | • query:  执行 SELECT 查询，支持预处理语句和参数绑定。
• execute:  执行 INSERT、UPDATE 或 DELETE 操作，支持参数化查询。
• list_tables:  列出当前数据库中的所有表。
• describe_table: 获取指定表的结构信息（字段名、类型、索引等）。
• query_to_csv: 执行 SELECT 查询并将结果保存为 CSV 文件。
• query_to_csv_string:  执行 SELECT 查询并以字符串形式返回 CSV 格式的结果。 | 否 | 需要授权 |  |  |
| 开发与数据管理 | postgres | P2 | 是 |  | 否 | 需要授权 |  |  |
| 信息源与内容获取 | unsplash | P2 | 是 | • search_photos: 根据关键词、颜色、方向等条件搜索 Unsplash 上的图片。
• get_photo: 获取指定 ID 的图片详情。
• random_photo: 获取一张随机图片，支持按关键词、颜色、方向等条件筛选。 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | x | P0 | 是 | • x_write：在 X (Twitter) 上发布新推文、回复推文或创建投票
• x_read：阅读推文详情，包括回复和对话内容
• x_search：使用关键词、主题标签或高级查询搜索推文
• x_user：获取用户个人资料信息 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | youtube | P1 | 是 | • youtube_search：使用 YouTube 数据 API 在 YouTube 上搜索视频。
• Get Video Info：提取 YouTube 视频的元数据，如标题、描述、作者、发布时间、观看次数等。
• Get Captions：提取视频字幕（支持自动生成及手动字幕），可指定语言。
• Convert to Markdown：将视频字幕转写并转为 Markdown 文稿，提供三种模板：简单字幕、带时间戳的完整内容、关键词高亮。
• List Templates：展示当前可用的 Markdown 模板名称。 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | bilibili | P2 | 是 | • Get user information by mid：通过用户 ID（mid）获取指定 B 站用户的详细资料。
• Search video information by bvid：通过视频唯一标识符（bvid）获取该视频的元数据和详细信息。
• Search videos by keywords：根据输入的关键词在 B 站检索相关视频，并返回匹配结果列表。 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | Reddit | P1 | 是 | • get_frontpage_posts：获取 Reddit 首页的热门帖子，可设置返回数量。
• get_subreddit_info：获取指定子版块（subreddit）的基本信息。
• get_subreddit_hot_posts：获取指定子版块的热门帖子，可设置返回数量。
• get_subreddit_new_posts：获取指定子版块的最新帖子，可设置返回数量。
• get_subreddit_top_posts：获取指定子版块的置顶/评分最高帖子，可设置数量与时间范围（如小时/天/周/月/年/全部）。
• get_subreddit_rising_posts：获取指定子版块的上升趋势帖子，可设置返回数量。
• get_post_content：获取指定帖子的详细内容，可附带评论树（支持设置评论数量与层级深度）。
• get_post_comments：获取指定帖子的评论内容，可设置返回数量。 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | Hugging Face | P2 | 是 | • search-models：根据查询词、作者、标签等条件搜索模型，并可设置返回数量。
• get-model-info：获取某个指定模型的详细信息。
• search-datasets：按条件搜索数据集。
• get-dataset-info：获取某个指定数据集的详细信息。
• search-spaces：按条件搜索 Hugging Face Spaces，包括支持的 SDK 类型。
• get-space-info：获取某个指定 Space 的详细信息。
• get-paper-info：获取某篇论文及其相关实现的详细信息。
• get-daily-papers：获取每日精选论文列表。
• search-collections：按条件搜索 Hugging Face Collections。
• get-collection-info：获取某个指定 Collection 的详细信息。 | 全局配置 | 不需要授权 |  |  |
| 社交与消息 | Telegram | P1 | 是 | • telegram_message：通过 Telegram Bot API 向 Telegram 频道或用户发送消息。支持直接通信和通知，并支持消息跟踪和聊天确认。**sendMessage**：向指定聊天发送文本消息。**sendPhoto**：向指定聊天发送图片。**deleteMessage**：删除指定消息。**getChat**：获取指定聊天的基本信息。**getChatHistory**：获取指定聊天的历史消息记录。**searchMessages**：按关键词搜索聊天消息。**markAsRead**：将指定聊天标记为已读。**getUserStatus**：获取某个用户的在线状态。**banChatMember**：在群组中封禁指定用户。**getChatMembers**：获取群组成员列表。 | 否 | 需要授权 |  |  |
| 社交与消息 | Slack | P1 | 是 | • slack_list_channels：列出工作区内的公共频道，支持分页。
• slack_post_message：向指定的 Slack 频道发送新消息。
• slack_reply_to_thread：在 Slack 中回复指定的消息线程。
• slack_add_reaction：为一条消息添加表情反应。
• slack_get_channel_history：获取某个频道的近期消息记录。
• slack_get_thread_replies：获取某条消息线程中的所有回复。
• slack_get_users：获取工作区内所有用户的基础资料信息。
• slack_get_user_profiles：批量获取多个用户的详细资料信息（适合批量操作）。
• slack_search_messages：在工作区中搜索消息 | 否 | 需要授权 |  |  |
| 社交与消息 | Instagram | P2 | 是 | • analyze_post_comments：分析指定 Instagram 帖子的评论，识别情绪倾向、主题和潜在线索。
• compare_accounts：对比多个 Instagram 账号的互动指标（如点赞、评论等），支持选择指标类型。
• extract_demographics：提取与指定账号或帖子互动用户的群体特征（如年龄、性别、地域），支持抽样。
• identify_leads：根据互动模式，识别潜在的高价值用户或潜在客户。
• generate_engagement_report：为指定 Instagram 账号生成完整的互动报告，支持时间范围设定。 | 否 | 需要授权 |  |  |
| 社交与消息 | LinkedIn | P1 | 是 | • get_person_profile：抓取并获取指定 LinkedIn 个人档案的详细信息，包括工作经历、教育背景、技能和人脉关系。
• get_company_profile：提取指定 LinkedIn 公司主页的完整公司信息。
• get_job_details：根据 LinkedIn 职位 ID 获取具体职位详情。
• search_jobs：通过关键词、地点等条件搜索职位信息。
• get_recommended_jobs：基于用户个人资料获取个性化职位推荐。
• close_session：关闭浏览器会话并清理资源。 | 否 | 需要授权 |  |  |
| 社交与消息 | Discord | P1 | 是 | • list_servers：列出可用的服务器。
• get_server_info：获取指定服务器的详细信息。
• get_channels：列出服务器中的频道。
• list_members：列出服务器成员及其角色。
• get_user_info：获取指定用户的详细信息。
• send_message：向频道发送一条消息。
• read_messages：读取频道的近期消息记录。
• add_reaction：为消息添加一个表情反应。
• add_multiple_reactions：为消息添加多个表情反应。
• remove_reaction：移除消息上的某个表情反应。
• moderate_message：删除消息并可对用户进行超时处理。
• create_text_channel：创建新的文本频道。
• delete_channel：删除已有的频道。
• add_role：为用户添加角色。
• remove_role：移除用户的某个角色。 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | Pinterest | P2 | 是 | • pinterest_search：根据关键词在 Pinterest 上搜索图片，支持设置返回数量与是否使用无头浏览模式。
• pinterest_get_image_info：获取指定 Pinterest 图片的详细信息（需提供图片 URL）。
• pinterest_search_and_download：根据关键词在 Pinterest 上搜索并下载图片，支持设置返回数量与是否使用无头浏览模式。 | 全局配置 | 不需要 |  |  |
| 社交与消息 | HotSpot | P2 | 是 | • hubspot_create_contact：创建新联系人，并自动避免重复。
• hubspot_create_company：创建新公司，并自动避免重复。
• hubspot_get_company_activity：获取指定公司的活动记录。
• hubspot_get_active_companies：获取最近活跃的公司列表。
• hubspot_get_active_contacts：获取最近活跃的联系人列表。
• hubspot_get_recent_conversations：获取近期的会话线程及消息内容。
• hubspot_search_data：在已获取的 HubSpot 数据中进行语义搜索。 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | Alpha Vantage | P2 | 是 | • get-stock-quote：获取指定公司的最新股票报价。
• get-company-info：获取指定公司的股票相关信息。
• get-crypto-exchange-rate：获取当前加密货币的实时汇率。
• get-time-series：获取某只股票的历史日度价格数据。
• get-historical-options：获取某只股票的历史期权链数据，支持排序功能。
• get-crypto-daily：获取加密货币的日度时间序列数据。
• get-crypto-weekly：获取加密货币的周度时间序列数据。
• get-crypto-monthly：获取加密货币的月度时间序列数据。 | 全局配置 | 不需要授权 |  |  |
| 信息源与内容获取 | Shopify | P2 | 是 | • get-products：获取全部产品，或按标题搜索产品，支持设置返回数量。
• get-products-by-collection：根据集合 ID 获取该集合下的产品列表。
• get-products-by-ids：根据一组产品 ID 获取对应产品详情。
• get-variants-by-ids：根据一组变体 ID 获取对应的产品变体信息。
• get-customers：分页获取客户信息，支持设置返回数量和翻页游标。
• tag-customer：为指定客户添加标签。
• get-orders：获取订单列表，支持高级筛选、分页和排序。
• get-order：根据订单 ID 获取单个订单详情。
• create-draft-order：创建草稿订单，支持指定商品、客户邮箱和收货信息。
• complete-draft-order：完成草稿订单，生成正式订单。
• create-discount：创建一个基础折扣码，支持设置折扣类型（百分比/固定金额）、数值、时间范围和用户限制。
• get-collections：获取全部集合，支持按名称筛选和设置返回数量。
• get-shop：获取店铺的基础信息。
• get-shop-details：获取店铺的详细扩展信息。
• manage-webhook：管理 Webhook，支持订阅、查询和取消订阅，需提供回调 URL 和事件主题。 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | Airtable | P2 | 是 | • list-bases：列出当前账号下所有可访问的 Base（数据库）。
• list-tables：获取指定 Base 中的所有表。
• list-records：获取指定表中的记录，支持分页和条件筛选。
• get-record：根据记录 ID 获取某条记录的详细信息。
• create-record：在指定表中创建新记录。
• update-record：更新指定表中某条记录的字段值。
• delete-record：删除指定表中的某条记录。
• search-records：在表中进行条件搜索，返回匹配的记录集合。 | 否 | 需要授权 |  |  |
| 信息源与内容获取 | Google Analytics | P1 | 是 | • get_ga4_data：获取 GA4 数据，支持自定义维度和指标。
• Smart data volume warnings：当查询结果超过 2,500 行时触发警告。
• Server-side aggregation：在需要时自动进行服务端聚合，提升查询效率。
• Intelligent sorting：返回最相关的数据，自动进行结果排序。
• User control parameters：支持参数控制。
• list_dimension_categories：浏览可用的维度分类。
• list_metric_categories：浏览可用的指标分类。
• get_dimensions_by_category：获取指定分类下的所有维度。
• get_metrics_by_category：获取指定分类下的所有指标。 | 否 | 需要授权 |  |  |
| 开发与数据管理 | GitLab | P2 | 是 | • get_mcp_server_version：返回当前 GitLab MCP 服务器的版本信息。
• get_issue：获取指定 GitLab 项目中的某个 Issue 详情。
• create_issue：在指定 GitLab 项目中创建新的 Issue，支持设置标题和描述。
• get_merge_request：获取指定 GitLab 项目中的某个合并请求（Merge Request）详情。
• get_merge_request_commits：获取指定合并请求中的提交列表。
• get_merge_request_changes：获取指定合并请求的文件变更（diff）。
• get_pipeline_jobs：获取指定 CI/CD 流水线中的任务（Jobs）。
• get_merge_request_pipelines_service：获取指定合并请求关联的所有流水线信息。 | 否 | 需要授权 |  |  |

## 接口定义

工具 DTO 定义：

```
export type ToolsetAuthType = 'manual' | 'oauth';

export type Toolset = {
  /**
   * Toolset ID
   */
  toolsetId: string;
  /**
   * Whether the toolset is global
   */
  isGlobal?: boolean;
  /**
   * Whether the toolset is builtin
   */
  isBuiltin?: boolean;
  /**
   * Toolset name
   */
  name?: string;
  /**
   * Toolset key
   */
  key?: string;
  /**
   * Toolset auth type
   */
  authType?: ToolsetAuthType;
  /**
   * Toolset auth data
   */
  authData?: {
    [key: string]: unknown;
  };
  /**
   * Toolset creation timestamp
   */
  createdAt?: string;
  /**
   * Toolset update timestamp
   */
  updatedAt?: string;
};

export type GenericToolsetType = 'regular' | 'mcp';

export type GenericToolset = {
  /**
   * Toolset type
   */
  type?: GenericToolsetType;
  /**
   * Toolset
   */
  toolset?: Toolset;
  /**
   * MCP server
   */
  mcpServer?: McpServerDTO;
};

```

接口列表（详细参数和返回值定义查看 OpenAPI Schema）：

[Untitled](https://www.notion.so/26e162107b4380dc9feae107ca1743ee?pvs=21)

## 预计排期

| 模块 | TODO | Owner | 开发时间 |  |
| --- | --- | --- | --- | --- |
| 工具调用流程（Ask/Workflow/Agent），包括 run workflow 校验 | • 常规工具 CRUD
• 工具调用 | @陈哲 | 2pd |  |
| 内置工具改造 |  | @陈哲 | 2pd |  |
| 常规工具实现（先做最高优的 10 个） |  | @陈哲 | 1pd |  |
| 前端输入框工具展示、选择 |  | @曹慧 | 1pd |  |
| 前端 Tool Store 安装与管理 |  | @曹慧 | 2pd |  |