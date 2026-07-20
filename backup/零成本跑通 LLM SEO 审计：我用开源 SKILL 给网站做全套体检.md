
> 把 SEO 流程交给大模型去跑，我前后折腾了好几轮，始终卡在两个地方：**评分结果不稳定**，以及**分析总是太浅**。后来发现，问题不在模型，在于流程本身不够完整。  
> 直到我把一个开源项目里的 11 套 SEO SKILL 挨个跑了一遍，才第一次拿到能真正指导行动、并在 Ahrefs 上看到正向反馈的审计报告。  
> 这篇文章，把踩过的坑、拼出来的工作流、以及一套**不翻墙、零成本**的使用方案，全部拆给你。

---

## 一、我是怎么一步步走到这套方案的

最早，我的想法很朴素：把自己做 SEO 优化的流程总结出来，让大语言模型（LLM）转成 SKILL，然后在评估过程中逐步完善流程。

但实际用起来发现两个问题：

1.  **自己的评估维度不够全**。我只能看到自己认知范围内的东西，换个网站类型，评估就漏项严重。
2.  **让 LLM 打分不靠谱**。同一个网站前后跑两次，分数能差不少，幻觉和不确定性太强。我很快意识到，**LLM 给出的分数只能当“参考信号”，绝不能当“决定元素”**。参考的意思是：“看到这个分，心里有个数，可能这里有大问题”；决定元素的意思是：“看到这个提醒，我必须改”。

后来看到有前辈分享 SEO 的 SOP，用 LLM 做了一批总结也转成了 SKILL，但我试用下来，分析还是不够全面。直到我在 GitHub 上找到了 [**coreyhaines31/marketingskills**](https://github.com/coreyhaines31/marketingskills) 这个项目，才第一次有了“这事真能落地”的感觉。

这个项目里内置了十几套专业营销技能 SKILL，几乎覆盖了 SEO 的所有核心方向。我逐项测完，效果可以，直接拿它给自己网站做了次全面审计和优化，**在 Ahrefs 上拿到了可验证的正向反馈**。这才有了今天这篇完整的复盘。

---

## 二、我的 LLM SEO 工作流

整个流程其实不复杂，我把步骤简化成下面这样：

1.  **加载 SKILL**：将 marketingskills 中的 SEO 相关 SKILL 加载到工具中（我用的是 openclaw）
2.  **输入目标网址**：告诉 LLM 我要分析哪个网站
3.  **自动审计/优化**：LLM 按照 SKILL 里写好的专业流程，执行技术审查、内容分析、竞品扫描等
4.  **输出报告 + 行动清单**：拿到一份按优先级排序的修复列表
5.  **人工复核 + 落地修改**：对高风险项做人工确认，然后实际改站
6.  **用 Ahrefs/GSC 验证效果**

> <img width="1651" height="700" alt="Image" src="https://github.com/user-attachments/assets/0feeefee-18ff-438d-8901-1a5ac801ca2b" />
> <img width="702" height="319" alt="Image" src="https://github.com/user-attachments/assets/c7889cc0-ca8f-4f90-af0c-84cd759f4cc5" />
> <img width="741" height="497" alt="Image" src="https://github.com/user-attachments/assets/985c9dd9-4066-4bcb-b64e-fb8f63d1867b" />
> <img width="734" height="464" alt="Image" src="https://github.com/user-attachments/assets/3c1c91d5-02ae-4fe8-9097-edbd12f98a59" />
> <img width="701" height="454" alt="Image" src="https://github.com/user-attachments/assets/47994377-cfd7-4dde-9486-879a1b870799" />
> <img width="734" height="679" alt="Image" src="https://github.com/user-attachments/assets/d7529a00-f4fd-4c3c-825c-098c63149fdc" />
> <img width="727" height="781" alt="Image" src="https://github.com/user-attachments/assets/7842d7e3-100f-414d-96b5-f175e9314bd4" />
> <img width="739" height="728" alt="Image" src="https://github.com/user-attachments/assets/9c766279-185a-463d-bf11-e080ca32af48" />

整个过程的核心，是把“专业 SEO 的经验流程”固化成 LLM 能严格执行的脚本。你自己不需要是每个细分领域的专家，但 SKILL 里已经封装了专家的检查逻辑。

---

## 三、核心武器：marketingskills 到底能做什么

这个项目里 SEO 相关的 SKILL，我分成了**直接优化**和**间接助力**两类，它们各自解决的问题和给我的实际帮助如下：

| 技能 | 解决什么问题 | 给我的网站带来的直接帮助 |
|------|--------------|--------------------------|
| **seo-audit** | 技术SEO、爬取/索引、页面Meta、速度、结构化数据等全面审计 | 输出完整审计报告 + 优先级行动清单，一下子把问题全暴露了 |
| **schema** | 添加 JSON-LD 结构化数据，让 Google 出富媒体结果 | 给博客文章补上了 Article Schema，搜索展示更丰富 |
| **ai-seo** | 针对 ChatGPT、Perplexity、Google AI Overviews 做优化 | 让我的工具站在 AI 推荐场景下被引用，特别是“推荐图片压缩工具”类问题 |
| **content-strategy** | 规划博客话题集群、选题矩阵、编辑日历 | 解决了我博客中英文内容重复、主题分散的老毛病，梳理出围绕核心关键词的内容地图 |
| **copywriting** | 优化首页、下载页、功能页文案，聚焦价值主张和 CTA | 把首页“三步搞定压缩”改得更戳海外用户痛点，转化意向明显提升 |
| **competitor-profiling** | 输入竞品 URL，输出完整竞品档案 | 快速摸清了 TinyPNG、Compressor.io 等产品的流量来源和内容策略 |
| **competitors** | 生成“我们 vs 竞品”的对比SEO页面 | 抓住 “TinyPNG alternative” 这类高意图搜索词，直接上量 |
| **programmatic-seo** | 用模板批量生成 SEO 页面 | 后期打算做“图片压缩工具对比”系列，一个模板能出几十个页面的框架 |
| **site-architecture** | 规划网站结构、导航、URL层级、内部链接 | 为后续加功能页和教程页提前打好信息架构地基 |
| **analytics** | 配置 GA4、事件追踪、转化漏斗 | 虽然已接 GA 和 Ahrefs Analytics，但优化了下载点击和注册的事件追踪 |

**对 SEO 有间接帮助的技能：**

| 技能 | 怎么帮到 SEO |
|------|--------------|
| **cro**（转化率优化） | 把 SEO 流量转化成实际下载，自然提升站内行为和排名信号 |
| **offers**（优惠/价值塑造） | 强化下载页“为什么要下载”的说服逻辑 |
| **copy-editing**（文案编辑） | 提升博客和页面文案可读性，降低跳出率 |
| **marketing-loops**（增长循环） | 设计“用户下载 → 推荐给别人”的闭环，产生自然外链 |
| **public-relations**（公关） | 在技术博客、工具合集站点争取外链机会 |

这些 SKILL 我都是一个一个跑过来的，测试结果比较稳定。尤其是 **seo-audit + ai-seo + competitors** 这三板斧下来，网站的自然搜索表现变化肉眼可见。

---

## 四、工具链怎么搭（国内网络免翻版）

很多人会问，这些 SKILL 能不能直接用 Claude Code 或 Codex 跑？当然可以，但我自己**懒得折腾代理和网络**，所以全部选择了国内能直连的方案。

### 我用的方案二
- **tbox.cn**：https://tbox.cn
- **交互框架**：openclaw（类似一个可以加载 SKILL 的 LLM 代理客户端）
- **模型**：DeepSeek V4 Pro

### 两种性价比方案
1.  **直接使用 DeepSeek V4 Pro**：能力足够跑通全部 SKILL，直接调用即可。
2.  **免费白嫖方案**：用 [**tbox.cn**](https://www.tbox.cn/) 提供的 openclaw 免费额度。它虽然不能直接操作浏览器，但提供免费的 DeepSeek V4 Pro 模型接口，而且可以直接对接到微信上使用，对于纯审计和分析类工作完全够用。

> <img width="1274" height="813" alt="Image" src="https://github.com/user-attachments/assets/7b6a5345-e9f9-4050-bba1-1ebeba604bfd" />

对多数个人站长和独立开发者来说，方案二等于**零成本拥有一套 24 小时待命的 SEO 审计助手**，非常划算。

---

## 五、两个不得不说的避坑经验

### 1. LLM 评分只能参考，不能做决策
这是我最想强调的一点。同一个网站，同一套 SKILL，不同时间跑出来的分数可能不一致。所以我现在把 LLM 的输出分为两档看待：
- **参考信号**：“这里分数异常低/高，我需要人工看一眼”
- **决定项**：“报告中明确指出了某条具体问题（比如 3 个页面缺 meta description），我一定要改”

不要把分数当成 KPI，把它当成一个提醒你去看细节的闹钟。

### 2. SKILL 再完整，也不能替代人工判断
marketingskills 的审计覆盖面已经很广了，但遇到特别小众的行业站，某些深层页面逻辑还是需要人工补充。工具负责 80% 的初筛和标准化检查，剩下的 20% 靠经验兜底，这个组合我用下来最舒服。

---

## 六、给你的极简启动清单

如果你也想照着来一遍，直接按这个清单操作：

1.  **复制项目链接**：https://github.com/coreyhaines31/marketingskills
2.  **领取免费额度**：访问 tbox.cn，获取 openclaw 免费使用资格（内置 DeepSeek V4 Pro）
3.  **绑定微信并安装 seo-audit SKILL**：如下图
4.  **跑出审计报告**：重点关注 “High Priority” 条目
5.  **按优先级修复**：先改技术 SEO（索引、速度、结构化数据），再动内容和竞品页
6.  **验证效果**：在 Ahrefs 或 Google Search Console 观察索引量、点击和排名的变化

整个过程跑通，顶多半天时间。但后续持续优化带来的自然流量回报，会长尾地作用很久。

<img width="1683" height="835" alt="Image" src="https://github.com/user-attachments/assets/be5bbd3c-c38f-4b75-90a2-bc61e8ad5ad1" />

<img width="1620" height="154" alt="Image" src="https://github.com/user-attachments/assets/deab4c71-b7a7-486c-a17b-b490f8f482cf" />

---

**最后想说**，这套方案的精髓不在于某个模型有多强，而在于把“专业营销经验”通过 SKILL 的形式复用了。我们普通开发者不用成为每个细分领域的 SEO 专家，也能做出接近专家水平的审计和优化——而且，不用折腾网络，还不用花钱。这一点，在当下的国内环境里，很难得。

有需要的朋友可以直接抄作业。后续我也会在博客更新更多 SKILL 的详细测试结果，欢迎保持关注。

---

> 如果这篇文章对你有帮助，欢迎分享给同样在折腾独立站 SEO 的朋友。有任何踩坑经验，也欢迎留言交流。