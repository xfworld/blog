## skill 介绍
- `daily-finance`：每日联网采集并校验高质量财经新闻、市场数据和关键事件，生成可直接发布的公众号财经日报，并作为后续深度分析的事实底稿。  
  
- `finance-core-analysis`：基于 `daily-finance` 的事实底稿，再联网复核关键数据，用流动性、利率、风险偏好、资金流、政策和资产负债表模型生成可发布的深度财经分析。  
  
- `finance-explosive-article`：基于前两份财经文档和最新外部数据校验，用“第一性原理 + 反直觉 + 系统模型”的德哥风格生成公众号爆款财经文章。  
  
- `digoal`：基于 digoal/德哥博客沉淀，面向 PostgreSQL、AI+数据库、开源生态、技术文章和架构判断，输出证据驱动、场景优先、可验证的德哥式分析与方案。  
  
- `github-weekly-trending`: 输入从 `https://github.com/trending?since=weekly&spoken_language_code=` 拷贝的内容, 编写本周热门开源项目文章, 输出到当前项目 markdown 目录中.   
  
- `open-source-project-article`: 输入开源项目地址, 深度分析该开源项目, 输出到当前项目 markdown 目录中.  
  
- `marketing-wechat-operator`: `微信公众号运营`, 编写爆款文章.  

- `postgres-commit-history-article`: 先进入 postgres 项目目录, 输入 `commitid1 commitid2` , 分析并解读这两个 commitid 中间的所有提交 (也包括这两个 commit), 输出到当前项目 markdown 目录中.
  
- `paper-interpretation`: 输入论文 PDF 或论文 URL , 通俗易懂解读论文. 例如用于解读 AI 论文 https://huggingface.co/papers/trending https://arxiv.org/abs/2604.14141 https://arxiv.org/abs/2508.02739 
  
- `pgfaq`: clone https://github.com/postgres/postgres 源码, 将其作为项目目录. 输入 PostgreSQL 相关的问题, 将回答结果保存到项目目录的 markdown 子目录中. 回答时会参考代码、文档和deepwiki, 并对回答内容正确性进行校验.  

## only for claude web
`skills_for_claude_web` 目录中的 skill 仅用于 Claude web 版.
  
`.skill` 可能是 claude skill 的二进制格式.  
  
- `paper-interpreter`: 输入论文 PDF 或论文 URL , 通俗易懂解读论文.   
