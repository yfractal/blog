# LightRAG 论文笔记

## 前言

首先 AI 我不熟，所以理解很可能是有误的。但我读论文/学知识的的原则是，要有明确的理解，这个理解可以是不完全，有偏差，甚至是错误的，这样方便我去修正和验证，如果没有，就没办法改进和收敛。类似假设，长期来看，假设的可被证否性比正确性更重要。

## 问题

LLM 训练是根据已有的数据进行训练，训练完成后，数据就是固定的了，所谓的 cutoff。这个时候，用户可以通过增加 context，让 LLM 基于 context 回答。

但很多时候，用户没有办法提供具体的 context，比如查找公司内部的流程。所以需要我们把这些 context 存起来，当用户输入问题时，找到相应的 context，再由 LLM 回答。即所谓的 RAG。

那这里面的问题有 3 个

1. 存储这些信息
2. 查找到相关信息
3. 给 LLM 提供 context

LightRAG 的亮点是，在提供给 LLM 的 context 里加入了关系（relationship），即 High-Level，帮助 LLM 更好的回答我们的问题。

下面通过这 3 个问题，介绍 LightRAG 的大体实现。

## LightRAG 实现

关键思想是在 context 中加入关系信息，通过结构化的知识图谱提供更准确的上下文，从而提高 LLM 回答质量。

### 存储（graph-based text indexing）
大致上是，得到文本后，让 LLM 分析，得到结构化的数据（Entities 和 Relationships），之后把这些数据存到数据库里，方便用户提问的时候提取这些信息。

文本输入会先被分成块（相关方法 `chunking_by_token_size`，分出来的块有重叠，似乎是 LLM 处理中的常见做法），交给 LLM 进行信息提取（相关方法 `_process_entity_relation_graph` -> `extract_entities`）。

信息提取使用 `PROMPTS["entity_extraction"]` prompt，主要提取实体名称、类型、描述等，然后识别所有实体之间的关系，给出关系关键词、关系描述等。

存储方面 LightRAG 采用双存储架构：
- **知识图谱数据库**：存储实体和关系的结构化信息，支持图遍历查询
- **向量数据库**：分别存储实体和关系的向量表示，支持语义搜索

整体流程图（Cursor 根据代码生成，[流程图链接](https://www.mermaidchart.com/play?utm_source=mermaid_live_editor&utm_medium=share#pako:eNq9V0tvJDUQ_ivWrMSFzG66O5OQEVop8-qNNoFkZ4WQCBp53NUzzfRrbXeSASFxCBckQEqQ9gICceGEuHLix-wfgJ-A3Xa7H9MTKYtELtMuV3311cNl54sOSTzo9DvdbvciJknsB4v-RYxQiNdJxvvID67Bu4jzbT9MrsgSU45ejoQOy-YLitMlGmGO55gB--Si88_Pr2_-_vN7I0NTnlC8EL9rxiFiF51PJbz6Gw2s3OTuG_Q8Tq5C8ISiqzAH78_p0zdf_Yo-gGTvM_RE_PKrhK4-Fp9nCeMLCtPzk0JpymlGeEZxiCiEmAdJzJZByop9BcopvgTKhNKrDGgATTZ2zuaHb9E45gFfo4-ACPYVLpOj4-lUEDj3KI65-DgNwsvMeNFmEM3B84J4YTamEAn9gCBQGgwwJcuGd0d5f41eVCJ4MIea8T1Mqmmq84HYq1b35ORUFfa3X-Q3Oo45UExy21oEYlOV87s_ikyMr7lWLfwfx6lsq5dwzRFZZvGq2Pgw4_lObilKg96thWICOKNJlAo9lckZGA9NLrqYN8gNAcciB9KUAGMNKmcULoMkYwg2yBacjkQOpVw0Dmh6W-iIE8SDOIN7eKky3_1U5GiaRRGmwee4JU2nWciDNATkASM0SPNsNNmdAl2AV1XZQo5VPTVp7RW0ag30P5CrNmIrRdWP4u9IUbxBo4RkkQgKDWUHyS5E3e5TNFAVv0VH5FUW0HKUoJOErEzAg1x5WOnp_ma_Gu2h0ladffv79v6TcWl70yUIx97mQPIFBOLmBJSuLOXLdG5Ora19h7rNKj2L0pyY3DWFCeRJrR_Uoa18OMXgPcNUTGl5sF8AE4ZsI471k9qs8MRkL-GcHG7Ulsxpo5boHTSu6W3vNIM_UvDbss-a7Zm3G4qKBOhx22zQIOYJIskSqGwiBbIufaoyjFQZbv_acjetCuksn5OzIGb8cZYyoHwWi0tV6sjLDwzbwNupUtlBfJ2CcTvOvY43Qt1-PloCrlfqQWGPVdjj_xC2FFfCZpTkMfMF34z9CoLFkpdJV305KS4QMZVTLI5w4yau9t5EWVjF4JokNMJiJoizIeKTPHTeYxyBuE4q7ksMFfREHznxUIBYnhnhuHZ1mot05mcxKc01a6fIWdvDoRgGs0tvrjNV7Q3jR1CMgOPa-RorB24zLa1PhKqhq-zuSY6qjnCqyiM-VrAWLyyPbcmVq3LlvmWuXB2KydX2Z05tYr5N2iZqLD1TTO-kK5CP0fYLwdXaavUsXxwL0zc_fi0ssbdGvqB2Lp6M61q_PpZTYmCVlaoJJAclsEs3SuCU814K5MOpnM5aYpfDSEuc8pxqyZ6-wY_6_X6qbgeRES0cCGEow1TLoSXXYVQs7frSaYMY1W1G0sbTD3stG9dVxi0qkzboidUqleamtIXQ2cR026zdVky3DdNtwXxWSxgJMWMj8JEITvwXFIb9R74Pc2e-wzhNViCWvQPfOtDL7lXg8WXfSa93SBImtP9od3e3gVT403B7BPu9XQNnw4Hn2FvhfN9vwJmYNN4hsQ_mJd4-tuaH-AF4ZeY0oG0d7vuOAbR6-z2yWwe07wOUyTSp6x3YtoHy3ttzrN793Dpf_guGpRKH)）：


<img width="936" alt="Image" src="https://github.com/user-attachments/assets/410da0b3-77dd-42e5-bbc5-a80cd853c0c6" />

### 查找相关信息
用户输入查询后，会通过 LLM 提取出 keywords，包括：
- **high-level**: 总体概念或主题（overarching concepts or themes）
- **low-level**: 具体的实体或细节

对应方法是 `get_keywords_from_query`。

之后根据查询模式：
- **Global 模式**：high-level keywords 用来调用 _get_edge_data，在关系向量数据库中搜索相关关系
- **Local 模式**：low-level keywords 用来调用 _get_node_data，在实体向量数据库中搜索相关实体

系统还会进行重要性排序（基于节点度数）和 token 预算管理。

整体流程图（Cursor 根据代码生成，[流程图链接](https://www.mermaidchart.com/play#pako:eNqlV0tv20YQ_isLB8mldCJLVtIQRQpLpJw2dpzGeRySQFiSQ5EwRbK7K8tqEiCHFijapkEeQC4tEhQoeip67ak_Jn-g-QmdffApyU1RATZA7uzMNzPfPPhww88C2LA3JozmEbnl3E8J_s6eJZ-l-Uzop51779-8ekpuc2Dkixmwhf2Jxy5cuRtRQbwZj1PgnDBIqIizlEdxzgmcxFx8-oBsbl4hA7z-21uyt7dvk2uwmGcsIO6JYNSX8g-0jdLusgS5EVEOWmCgNW6hyhe_kxssm-bCJkf6Dh9DeUkhVD7YGviXErh6ezAT6vXnhwfXyTwWUXnfYBlsaStdtPLyO3KDMg4SPrkJPEcPQakxAEkUT6JxAseQjAs9jfMkm7eO2y7vYwo2D3Pw4zD2iUMF9dBhFeoYuMHUVZiGD9-_-fEPnQV173FL117m00SdIG4R6fdDefdRdfSIOFs6p24qYrEgd8AXGSOHQJkfGXiOjoIjo_DiLwJSEuGMjwPvfBVME-JlJ9XpPogoC-xCP4-ncUKZtMiVqUY-RJaPjwqZ0l6BRvvv9HROBlT4UYF-L8uOZrnSdZRm8wSCCYwVocdxysX5CYhxim7zsSevKcFdENrEguQsy4HVbfW0rW1p61tjy4EJAyBDmvgzzfT1BqWxcaAu1G0Wd6GwHE_zjAma-kAYTY9K-9vafl_m6DUZxWlQuHqzXmUf4rE8afvtZ2mKCYGgWbRtWu4mmbeOS7WzR8Q1ZKqDW0kpV1PKNZRqWF_Nq3XF9T-ItcpnV7PLbbCr4c6HcGxVrOvWlpnmaqa5Daa5qOY_0U09n0a3Bog1pHM16VxJuue_atINS6K4phqV4hG23aZOSIM8i1PB27UVgKBxskQtVCywV5MdzmHqJQvD-76CMJLB-J7cRHDkHDlEsLLxzhKjXb3wFrZCb5E5IEWEOrnFZqkv_ZWnIjuClCSYdFG6aPTrp5F-kNx9-TMZZWyK46wAJpuw7uLGcZvEgUVSOgWLiEWO_7G6fBbnZWpKutiEMx-lJsIqJ8uy-C1p5naK6GxZkAIDZpEwTmCcY621A4bpwHIr5g_ZhRSYsmZ80YW1W5u0RgRaV43iXS3fmqOMTsasPuNMFRZR-ai2AzQq7DoVM4ZWEppOZhTpS1M-B1YY0-iu3nv30zcIBEXTNp7S0XL6HS64gCnHQpAM0ud85uk9xRkUx_fub5D3b15__fefz5qTc0EOchMjfn_DmJG_O85g5fBzBsqld09-waY1pXjmm2ZCcAcp-VxmkZfi2iXMoxFCSkMQxOmEY8SmiD8o2CSlMYQBiVPbjOpz5OrCY3FAprJfN3F29QBY1VdPR9soTsOuZbjNEv4w0Kbxr0W96wx0D71WdCqya1JWKtNtTgfrQgNEorpsFVnTBf2qC3ISovey9BFqKadN4L51DIwjvKxMvYGGDaq9KOE-VzFkNc1Q5iAvOKYL63RuoYDi1vMVa2wJdgfrcfEVcDKTxRQ3k2PkeZFMAWxaxWOIBT3JWCxvezjrzM4oSaeGZBNKV7e2FV2j4s4iFRFwpZCBwH3zGBPt63qvwmt6CceTCBjiNQVeSuxT9AP_pESqkqA29xD3wrUZqJbYNeHXRzr6jaW3GfT9A0dtIM_e1rbfEtko82fIdV3rmz5iZ2Wp1OldlK9ZcbEITDWzpZVLcRi4kFS0yd0owymE30LyPwZHfmnwOvcKjDIdPzytr1VtkPVCPx3q0uqEeKu1DurTuon2ajaXOBdWs_rXYZYL0asnRbGvwjzMph5-AQaE5rje0CWkgww_sI6LpiWby6RZrCtAos48gRMyxaEfb4bUB-lVG-Myo8y-grO_WDyqj4fzOIFk668WPvOmWy398g02sGoLb7xw2xJuW6L6fJSvZTOoBqB5021BPhSLBDuZfvYTyrkDIUmSqdwFEvtMGILX8ywuGO4z-Ni_FG5dMo-b8zgQkd3LTyw_SzBwZzqdTktTUMxErW7bp2G_U6rrwqWg112rLgzDljpMsY9f-4jYKOxuXb4Y9kqFW_2Lfb_TVNg9TaGcH6Wrlz_uVNggvOh3Ov-OraYQQ29hrDF69ZdO13J6lrNtOX3L7Vpuz3K3LdwFi9DUZXesQdcaWbhPVZ7Wz4cK8MbjfwBkCQIx)）

<img width="475" alt="Image" src="https://github.com/user-attachments/assets/a475e93f-78c6-4be6-98b3-48ea5bf14a29" />


### 给 LLM 提供 context
得到相关数据之后，LightRAG 会根据 context，对话记录等生成 prompt 发给 llm 得到答案，具体的 prompt 在 PROMPTS["rag_response"]。

## 其他
LightRAG 是用 LLM 来评价回答的质量的，不清楚这个是不是业界常见做法。我关注 evaluation 的一个原因是，AI 需要进行调优，评估标准的方法也可以用来调优。有明确答案的数据，可以直接验证，这种需要"主观"判断的，用 LLM 来做蛮有趣的。另外一个原因是，可以通过 evaluation 推导出论文的适用范围甚至是局限性。LightRAG 在其实验的对象表现的很优秀，但当我们把它使用到更多、更复杂的场景下，可能需要一些改进。

论文地址：https://arxiv.org/abs/2410.05779

## 总结
RAG 也可以理解为"智能搜索 + 上下文理解"。LightRAG 通过 LLM 构建结构化数据，查询时再让 LLM 分析问题提取关键词，然后在多个数据库中检索相关信息，最后将这些信息作为 context 提供给 LLM 生成答案。整个流程中 LLM 的质量会同时影响知识提取和问题理解两个方面，所以模型选择也很重要。
