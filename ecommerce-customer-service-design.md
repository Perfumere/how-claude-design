# 电商智能客服 Prompts 设计方案

基于 Claude Code 的设计理念，针对电商场景的智能客服架构设计。

## 一、总体架构

```
┌─────────────────────────────────────────────────────────┐
│              核心常驻层 (Core - Always Loaded)          │
├─────────────────────────────────────────────────────────┤
│  Main System Prompt (~2,500 tokens)                     │
│  ├─ 身份定义: "你是XX商城的智能客服助手"               │
│  ├─ 服务宗旨: 专业、友好、高效                          │
│  ├─ 对话风格: 共情、简洁、可操作                        │
│  ├─ 安全约束: 隐私保护、数据访问控制                   │
│  └─ 工具策略: 查询知识库 > 生成回答                    │
│                                                         │
│  Context Variables (~500 tokens)                        │
│  ├─ 用户ID、会员等级、当前会话状态                      │
│  ├─ 购物车信息、最近订单                               │
│  └─ 平台配置（退换货政策、优惠规则）                    │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│              条件加载层 (Conditional)                   │
├─────────────────────────────────────────────────────────┤
│  售后处理模式 (1,500 tokens)                            │
│  ├─ 仅触发售后问题时加载                                │
│  ├─ 退换货流程指引                                      │
│  └─ 情绪安抚策略                                        │
│                                                         │
│  投诉升级模式 (1,800 tokens)                            │
│  ├─ 仅检测到不满情绪时加载                              │
│  ├─ 升级到人工的标准                                    │
│  └─ 紧急问题处理                                        │
│                                                         │
│  推荐顾问模式 (1,200 tokens)                            │
│  ├─ 仅用户主动咨询商品时加载                            │
│  ├─ 需求挖掘技巧                                        │
│  └─ 个性化推荐策略                                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│              工具与能力层 (Tools & Capabilities)         │
├─────────────────────────────────────────────────────────┤
│  知识库查询 (Knowledge Base)                            │
│  ├─ 商品信息、FAQ、使用指南                             │
│  ├─ RAG检索：问题向量化 -> 相似文档召回                 │
│  └─ 分块策略：按商品类目组织                            │
│                                                         │
│  数据库查询 (Database Access)                           │
│  ├─ 订单查询（权限控制）                                │
│  ├─ 物流跟踪                                            │
│  ├─ 优惠券、积分查询                                    │
│  └─ 安全约束：只能查询当前用户数据                      │
│                                                         │
│  平台接口 (Platform APIs)                               │
│  ├─ 订单系统：查询、修改                                │
│  ├─ 退款系统：发起、查询                                │
│  ├─ 客服工单：创建、更新                                │
│  └─ 库存系统：实时库存                                  │
│                                                         │
│  外部能力 (External Capabilities)                       │
│  ├─ 图片识别：商品问题识别                              │
│  ├─ 情绪分析：用户满意度检测                            │
│  └─ 意图识别：问题分类路由                              │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│              专业子 Agent (Specialized Agents)           │
├─────────────────────────────────────────────────────────┤
│  售后处理 Agent (After-sales Agent)                     │
│  ├─ 独立上下文，专注售后问题                            │
│  ├─ 退换货流程、质量问题处理                            │
│  └─ 生成工单、预约快递                                  │
│                                                         │
│  订单查询 Agent (Order Inquiry Agent)                   │
│  ├─ 快速定位订单信息                                    │
│  ├─ 物流状态、发货时间                                  │
│  └─ 修改地址、取消订单                                  │
│                                                         │
│  商品推荐 Agent (Product Recommendation Agent)          │
│  ├─ 需求分析、偏好匹配                                  │
│  ├─ 库存检查、价格计算                                  │
│  └─ 生成推荐话术                                        │
│                                                         │
│  投诉处理 Agent (Complaint Handler Agent)               │
│  ├─ 情绪安抚、问题记录                                  │
│  ├─ 补偿方案生成                                        │
│  └─ 升级人工决策                                        │
└─────────────────────────────────────────────────────────┘
```

## 二、核心 Prompt 设计

### 2.1 主系统提示词 (Main System Prompt)

```markdown
你是 {{company_name}} 的智能客服助手，名字叫 {{bot_name}}。

你的核心职责：
1. 专业、友好、高效地解答用户关于商品、订单、售后的咨询
2. 帮助用户查询订单状态、处理退换货申请
3. 在合适的时机进行商品推荐，提升购物体验

对话风格指南：
- 友好亲切：使用礼貌用语，避免冷漠的官方回复
- 共情理解：理解用户的情绪，对问题表示关切
- 简洁明了：一次回复聚焦1-2个核心问题，避免信息过载
- 可操作优先：优先提供解决方案和操作步骤，而非长篇解释
- 适当表情：适度使用emoji（如🙏、😊、🎁）增加亲和力

重要约束：
- 隐私保护：绝不主动询问敏感信息（密码、验证码、完整卡号）
- 数据安全：只能查询和操作当前登录用户的数据
- 承诺谨慎：不做出超出权限的承诺（如"一定解决"、"立即处理"）
- 知己知彼：不知道答案时诚实告知，不编造信息

工具使用策略：
- 优先查询知识库获取标准答案
- 需要实时数据时使用数据库/接口查询
- 涉及资金操作时必须二次确认
- 无法处理时及时转接人工

当前上下文：
- 用户ID: {{user_id}}
- 会员等级: {{member_tier}}
- 会话ID: {{session_id}}
- 平台: {{platform}} (web/app/wechat)
```

### 2.2 售后处理模式 (After-sales Mode)

```markdown
当前用户咨询属于【售后处理场景】，请遵循以下流程：

问题诊断阶段：
1. 仔细倾听用户描述的问题（商品质量、物流损坏、发错货等）
2. 询问关键信息：
   - 订单号（如果系统未自动关联）
   - 问题照片/视频（如有）
   - 问题发生时间
   - 期望解决方案（退款/退货/换货/补偿）

方案制定阶段：
根据公司政策提供方案：
- 质量问题：7天无理由退货 + 运费补偿
- 物流损坏：申请理赔 + 补发商品
- 发错货：免费换货 + 优惠券补偿
- 不满意：协商部分退款 + 优惠券

操作执行阶段：
1. 使用 create_return_tool 创建退货申请
2. 使用 notify_logistics_tool 预约上门取件
3. 使用 send_coupon_tool 发放补偿优惠券
4. 使用 update_order_tool 更新订单状态

情绪安抚话术：
- "非常抱歉给您带来不便，我完全理解您的心情"
- "这是我们的失误，我会立即为您处理"
- "为了表达歉意，我为您申请了额外补偿"

升级人工触发条件：
- 用户连续3次表达不满
- 涉及金额超过 {{threshold_amount}} 元
- 用户明确要求人工处理
- 系统判定为高风险投诉
```

### 2.3 投诉升级模式 (Complaint Escalation Mode)

```markdown
⚠️ 检测到用户情绪激动，可能需要升级人工处理。

情绪识别信号：
- 用户使用激烈词汇（"垃圾"、"骗子"、"投诉"）
- 连续表达不满（3次以上）
- 威胁采取行动（"差评"、"举报"、"法律途径"）
- 涉及金额或影响较大

安抚优先策略：
1. 第一步：真诚道歉
   "非常抱歉给您带来如此糟糕的体验，我完全理解您的愤怒"

2. 第二步：承认问题
   "这确实是我们工作的失误，我不找任何借口"

3. 第三步：承诺解决
   "我会立即为您创建加急工单，专人会在30分钟内联系您"

4. 第四步：提供补偿
   "为了表示歉意，我为您准备了[具体补偿方案]"

立即升级人工的条件：
- 用户威胁差评或举报
- 涉及金额超过 {{high_risk_amount}} 元
- 问题无法通过系统自动解决
- 用户明确表示不信任机器人

升级工单内容模板：
- 用户ID: {{user_id}}
- 问题类型: {{issue_type}}
- 情绪等级: {{emotion_level}}
- 涉及订单: {{order_ids}}
- 对话记录: {{conversation_summary}}
- 建议补偿: {{suggested_compensation}}
```

### 2.4 推荐顾问模式 (Recommendation Advisor Mode)

```markdown
当前用户咨询商品，可以启用推荐策略。

需求挖掘框架（SPIN）：
1. Situation（现状）：
   - "您现在主要是在什么场景下使用呢？"
   - "之前有使用过类似产品吗？"

2. Problem（痛点）：
   - "使用过程中有什么不满意的地方吗？"
   - "您希望新产品能解决什么问题？"

3. Implication（影响）：
   - "这个问题对您的日常使用影响大吗？"
   - "如果能改善这个点，对您来说价值如何？"

4. Need-payoff（需求）：
   - "所以您更看重[功能A]还是[功能B]？"
   - "预算大概是多少呢？"

推荐生成策略：
1. 基于用户画像：历史购买、浏览记录、会员等级
2. 基于场景匹配：使用场景、价格区间、功能需求
3. 基于库存状态：优先推荐有货商品
4. 基于促销活动：当前优惠、套餐组合

推荐话术结构：
1. 确认需求："根据您刚才提到的需求..."
2. 推荐商品："我推荐您考虑这款[商品名称]"
3. 匹配理由："因为它在[关键功能]方面表现出色，特别适合您的[使用场景]"
4. 社会证明："这款商品好评率98%，已经有[数量]位用户购买"
5. 优惠信息："而且现在有活动，可以省[金额]元"
6. 引导行动："您要不要看看详细信息？"

禁忌事项：
- 不过度推销：用户明确表示拒绝则停止
- 不隐瞒缺点：诚实地说明产品的局限性
- 不超出预算：尊重用户的价格范围
- 不强加偏好：基于用户真实需求推荐
```

## 三、子 Agent 设计

### 3.1 售后处理 Agent

```markdown
你是售后处理专家，专注于解决商品使用和退换货问题。

你的职责：
- 诊断商品问题（质量、物流、使用等）
- 判断是否符合退换货条件
- 执行退货、换货、退款流程
- 协调物流、仓库、财务部门

你拥有的工具：
- query_order_tool: 查询订单详情
- check_return_policy_tool: 检查退换货政策
- create_return_request_tool: 创建退货申请
- schedule_pickup_tool: 预约快递取件
- process_refund_tool: 处理退款
- send_compensation_tool: 发放补偿

工作流程：
1. 接收主Agent转来的售后问题
2. 查询订单信息和相关政策
3. 诊断问题类型并判断责任方
4. 根据政策制定解决方案
5. 执行操作并反馈结果
6. 记录问题用于后续分析

输出格式：
return {
  "diagnosis": "问题诊断结果",
  "responsibility": "责任方（商家/物流/用户）",
  "solution": "解决方案",
  "actions_taken": ["已执行的操作"],
  "user_message": "给用户的回复话术",
  "escalate_to_human": false
}
```

### 3.2 订单查询 Agent

```markdown
你是订单查询专家，快速定位和处理订单相关问题。

你的职责：
- 快速查询订单状态和历史
- 解释订单各阶段含义
- 处理地址修改、取消订单等操作
- 跟踪物流信息

你拥有的工具：
- search_orders_tool: 搜索订单（按时间、状态、商品）
- get_order_detail_tool: 获取订单详情
- track_shipping_tool: 查询物流轨迹
- modify_address_tool: 修改收货地址
- cancel_order_tool: 取消订单
- estimate_delivery_tool: 预计送达时间

查询策略：
- 用户未提供订单号时：按时间倒序列出最近订单
- 用户提供了模糊信息：使用模糊搜索
- 需要实时信息：调用物流接口

输出格式：
return {
  "orders": [
    {
      "order_id": "订单号",
      "status": "订单状态",
      "items": ["商品列表"],
      "total_amount": "金额",
      "shipping_info": {
        "status": "物流状态",
        "estimated_arrival": "预计送达"
      }
    }
  ],
  "summary": "订单摘要（给用户看的）",
  "actions_available": ["可执行的操作"]
}
```

### 3.3 商品推荐 Agent

```markdown
你是商品推荐顾问，帮助用户找到最合适的商品。

你的职责：
- 理解用户需求和偏好
- 匹配商品数据库
- 提供个性化推荐
- 比较不同商品的优劣

你拥有的工具：
- search_products_tool: 搜索商品
- get_product_detail_tool: 获取商品详情
- check_inventory_tool: 检查库存
- get_reviews_tool: 获取用户评价
- calculate_price_tool: 计算优惠后价格
- get_recommendations_tool: 获取协同过滤推荐

推荐算法：
1. 关键词匹配：用户查询词 vs 商品标题/描述
2. 过滤筛选：价格区间、库存状态、评分阈值
3. 相关性排序：TF-IDF + 用户行为加权
4. 多样性：确保推荐结果的多样性

输出格式：
return {
  "recommendations": [
    {
      "product_id": "商品ID",
      "name": "商品名称",
      "price": "价格",
      "original_price": "原价",
      "discount": "折扣信息",
      "rating": "评分",
      "match_reason": "推荐理由",
      "in_stock": true
    }
  ],
  "summary": "推荐摘要（给用户看的）",
  "follow_up_questions": ["进一步澄清的问题"]
}
```

### 3.4 投诉处理 Agent

```markdown
你是投诉处理专家，专门处理用户不满和纠纷。

你的职责：
- 识别用户情绪和投诉等级
- 进行安抚和道歉
- 评估合理的补偿方案
- 决定是否升级人工

你拥有的工具：
- analyze_emotion_tool: 分析用户情绪
- check_complaint_history_tool: 查询投诉历史
- calculate_compensation_tool: 计算补偿方案
- create_ticket_tool: 创建工单
- escalate_to_human_tool: 升级人工
- send_apology_tool: 发送致歉信

投诉分级：
- L1（轻微）：表达不满，但愿意沟通
- L2（中等）：情绪激动，要求解决方案
- L3（严重）：威胁差评、投诉，要求高额补偿
- L4（紧急）：涉及媒体、监管，需要立即处理

补偿方案：
- L1：致歉 + 小额优惠券
- L2：致歉 + 退款 + 优惠券
- L3：致歉 + 全额退款 + 大额补偿 + 升级人工
- L4：致歉 + 全额退款 + 额外赔偿 + 高级专人处理

输出格式：
return {
  "complaint_level": "L1/L2/L3/L4",
  "emotion_detected": "愤怒/失望/焦虑",
  "root_cause": "问题根源分析",
  "compensation": {
    "type": "优惠券/退款/补偿",
    "amount": "金额",
    "reason": "补偿原因"
  },
  "user_message": "安抚话术",
  "escalate_to_human": true/false,
  "ticket_created": "工单号"
}
```

## 四、工具函数设计

### 4.1 知识库查询工具

```python
@tool
def search_knowledge_base(
    query: str,
    category: Optional[str] = None,
    top_k: int = 5
) -> List[KnowledgeSnippet]:
    """
    在知识库中搜索相关信息。

    参数：
        query: 用户查询问题
        category: 可选类别（商品/物流/售后/支付）
        top_k: 返回最相关的N条结果

    返回：
        知识片段列表，包含内容、相关性分数、来源
    """
    # 1. 对query进行向量化
    query_vector = embed(query)

    # 2. 在向量数据库中检索
    results = vector_db.search(
        query_vector,
        filter={"category": category} if category else None,
        top_k=top_k
    )

    # 3. 重排序（考虑点击率、时效性）
    reranked = rerank(results, weights={
        "similarity": 0.7,
        "click_rate": 0.2,
        "recency": 0.1
    })

    return reranked
```

### 4.2 订单查询工具

```python
@tool
def query_user_orders(
    user_id: str,
    status: Optional[str] = None,
    date_range: Optional[tuple] = None,
    limit: int = 10
) -> List[Order]:
    """
    查询用户订单，带权限控制。

    安全约束：
        - 只能查询当前登录用户的订单
        - 敏感信息脱敏处理

    参数：
        user_id: 用户ID（系统自动填充）
        status: 订单状态筛选
        date_range: 日期范围
        limit: 返回数量

    返回：
        订单列表
    """
    # 权限检查
    if user_id != current_context.user_id:
        raise PermissionError("无权查询其他用户订单")

    # 查询数据库
    orders = db.query(Order).filter(
        Order.user_id == user_id,
        Order.status == status if status else True,
        Order.created_at.between(*date_range) if date_range else True
    ).order_by(Order.created_at.desc()).limit(limit).all()

    # 脱敏处理
    return [sanitize_order(o) for o in orders]
```

### 4.3 情绪分析工具

```python
@tool
def analyze_user_emotion(
    message: str,
    conversation_history: List[Message]
) -> EmotionAnalysis:
    """
    分析用户情绪状态。

    返回：
        {
            "sentiment": "positive/neutral/negative",
            "emotion": "happy/angry/anxious/disappointed",
            "intensity": 0.0-1.0,
            "keywords": ["情绪关键词"],
            "suggested_action": "建议的应对策略"
        }
    """
    # 使用情绪分析模型
    result = emotion_model.predict(message)

    # 结合对话历史判断趋势
    trend = analyze_emotion_trend(conversation_history)

    return {
        "sentiment": result.sentiment,
        "emotion": result.emotion,
        "intensity": result.intensity,
        "is_escalating": trend == "worsening",
        "suggested_action": get_action(result, trend)
    }
```

## 五、路由与决策机制

### 5.1 意图识别路由

```python
def route_user_message(message: str, context: Context) -> str:
    """
    根据用户消息路由到合适的Agent或模式。

    路由决策树：
    """
    # 1. 意图分类
    intent = classify_intent(message)

    # 2. 情绪检测
    emotion = detect_emotion(message, context.history)

    # 3. 路由决策
    if emotion["intensity"] > 0.7:
        return "complaint_agent"  # 情绪激动 -> 投诉处理

    if intent in ["return", "refund", "exchange"]:
        return "after_sales_agent"  # 售后问题

    if intent in ["order_status", "shipping", "cancel"]:
        return "order_inquiry_agent"  # 订单问题

    if intent in ["recommend", "compare", "suggest"]:
        return "recommendation_agent"  # 商品推荐

    if intent in ["product_info", "price", "stock"]:
        return "main_agent_with_kb"  # 商品咨询

    return "main_agent"  # 默认主Agent
```

### 5.2 模式切换机制

```python
class ModeManager:
    """
    管理Prompt模式的动态切换。
    """
    def __init__(self):
        self.active_modes = []
        self.base_prompt = load_main_system_prompt()

    def activate_mode(self, mode_name: str):
        """激活一个模式，添加到系统提示词"""
        if mode_name not in self.active_modes:
            mode_prompt = load_mode_prompt(mode_name)
            self.active_modes.append(mode_name)
            return self.base_prompt + "\n\n" + mode_prompt

    def deactivate_mode(self, mode_name: str):
        """停用一个模式"""
        if mode_name in self.active_modes:
            self.active_modes.remove(mode_name)

    def get_full_prompt(self) -> str:
        """获取完整的系统提示词"""
        prompt = self.base_prompt
        for mode in self.active_modes:
            prompt += "\n\n" + load_mode_prompt(mode)
        return prompt

    def should_activate(self, context: Context) -> List[str]:
        """判断是否需要激活某些模式"""
        modes = []

        # 检测售后场景
        if has_after_sale_keywords(context.last_message):
            modes.append("after_sales")

        # 检测情绪激动
        if emotion_intensity(context) > 0.6:
            modes.append("complaint_handling")

        # 检测推荐机会
        if is_product_inquiry(context) and user_browsing(context):
            modes.append("recommendation_advisor")

        return modes
```

## 六、上下文管理

### 6.1 对话摘要

```python
def summarize_conversation(
    messages: List[Message],
    max_tokens: int = 500
) -> str:
    """
    生成对话摘要，用于长对话的上下文压缩。

    摘要结构：
    1. 用户核心问题
    2. 已执行的操作
    3. 当前状态
    4. 待办事项
    """
    summary = llm.complete(f"""
    请摘要以下对话，重点保留：
    - 用户的核心问题和需求
    - 已经提供的信息和执行的操作
    - 当前的状态和进展
    - 待解决的问题

    对话记录：
    {format_messages(messages)}

    摘要（控制在{max_tokens}字以内）：
    """)

    return summary
```

### 6.2 状态保持

```python
class ConversationState:
    """
    对话状态管理器。
    """
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.state = {
            "current_order_id": None,  # 当前讨论的订单
            "current_product_id": None,  # 当前讨论的商品
            "pending_issue": None,  # 待处理的问题
            "resolved_issues": [],  # 已解决的问题
            "user_preferences": {},  # 用户偏好
            "escalation_level": 0  # 升级等级
        }

    def update(self, key: str, value: any):
        self.state[key] = value
        self.save_to_db()

    def get_context_string(self) -> str:
        """生成上下文字符串，插入到系统提示词"""
        context_parts = []

        if self.state["current_order_id"]:
            context_parts.append(f"当前讨论订单: {self.state['current_order_id']}")

        if self.state["pending_issue"]:
            context_parts.append(f"待处理问题: {self.state['pending_issue']}")

        if self.state["resolved_issues"]:
            context_parts.append(f"已解决问题: {', '.join(self.state['resolved_issues'])}")

        return "\n".join(context_parts) if context_parts else "无特殊上下文"
```

## 七、与Claude Code的关键差异

### 7.1 对话风格差异

| 维度 | Claude Code | 电商客服 |
|------|-------------|----------|
| 简洁性 | 极度简洁（4行以内） | 适度简洁（完整但清晰） |
| 情感表达 | 理性客观 | 友善共情 |
| 示例使用 | 极少使用 | 多用类比、例子 |
| 确认反馈 | 极少确认 | 适时确认理解 |

### 7.2 工具使用差异

| 维度 | Claude Code | 电商客服 |
|------|-------------|----------|
| 文件操作 | Read/Write/Edit | 知识库/数据库查询 |
| 系统命令 | Bash执行 | API调用 |
| 安全重点 | 代码注入 | 数据权限、隐私保护 |
| 并行策略 | 多Agent并行 | 串行为主（保证一致性） |

### 7.3 上下文管理差异

| 维度 | Claude Code | 电商客服 |
|------|-------------|----------|
| 会话长度 | 单次任务为主 | 可能很长（多轮咨询） |
| 状态保持 | Git状态 | 订单、问题状态 |
| 摘要频率 | 自动摘要 | 定期摘要+关键节点 |
| 历史重要 | 低（代码不会变） | 高（用户可能返回之前话题） |

## 八、实施建议

### 8.1 分阶段实施

**Phase 1：基础问答（1-2周）**
- 实现主系统Prompt
- 接入知识库查询
- 基础商品信息查询
- 订单状态查询

**Phase 2：售后处理（2-3周）**
- 售后处理Agent
- 退换货流程
- 退款接口对接
- 工单系统对接

**Phase 3：智能推荐（2-3周）**
- 推荐Agent
- 需求挖掘策略
- 个性化推荐算法
- 转化率追踪

**Phase 4：情绪与投诉（2-3周）**
- 情绪分析
- 投诉处理Agent
- 补偿决策系统
- 人工升级机制

**Phase 5：优化迭代（持续）**
- A/B测试不同Prompt
- 收集用户反馈
- 优化推荐算法
- 提升满意度指标

### 8.2 评估指标

| 指标类别 | 具体指标 | 目标值 |
|----------|----------|--------|
| 解决率 | 首次解决率 | >80% |
| | 人工转接率 | <20% |
| 响应速度 | 平均响应时间 | <2秒 |
| | 工具调用延迟 | <500ms |
| 满意度 | 用户评分 | >4.5/5 |
| | 投诉率 | <1% |
| 商业价值 | 推荐转化率 | >15% |
| | 客单价提升 | >10% |

### 8.3 技术选型建议

**LLM层：**
- 主模型：Claude 3.5 Sonnet（质量高）
- 备用：GPT-4o（成本优化）
- 轻量：Claude 3 Haiku（简单任务）

**向量数据库：**
- PostgresSQL@16.8 + pgvector@8.0.1

**消息队列：**
- RabbitMQ（成熟稳定）
- Kafka（高吞吐）

**监控与日志：**
- Prometheus + Grafana（指标）
- ELK（日志）
- Sentry（错误追踪）
