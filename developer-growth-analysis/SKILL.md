---
name: developer-growth-analysis
description: Analyzes your recent Claude Code chat history to identify coding patterns, development gaps, and areas for improvement, curates relevant learning resources from HackerNews and Stack Overflow, and automatically sends a personalized growth report to your email.
---

# Developer Growth Analysis

This skill provides personalized feedback on your recent coding work by analyzing your Claude Code chat interactions and identifying patterns that reveal strengths and areas for growth.

## When to Use This Skill

Use this skill when you want to:

- Understand your development patterns and habits from recent work
- Identify specific technical gaps or recurring challenges
- Discover which topics would benefit from deeper study
- Get curated learning resources tailored to your actual work patterns
- Track improvement areas across your recent projects
- Find high-quality articles that directly address the skills you're developing

This skill is ideal for developers who want structured feedback on their growth without waiting for code reviews, and who prefer data-driven insights from their own work history.

## What This Skill Does

This skill performs a six-step analysis of your development work:

1. **Reads Your Chat History**: Accesses your local Claude Code chat history from the past 24-48 hours to understand what you've been working on.

2. **Identifies Development Patterns**: Analyzes the types of problems you're solving, technologies you're using, challenges you encounter, and how you approach different kinds of tasks.

3. **Detects Improvement Areas**: Recognizes patterns that suggest skill gaps, repeated struggles, inefficient approaches, or areas where you might benefit from deeper knowledge.

4. **Generates a Personalized Report**: Creates a comprehensive report showing your work summary, identified improvement areas, and specific recommendations for growth.

5. **Finds Learning Resources**: Uses HackerNews and Stack Overflow to curate high-quality articles and discussions directly relevant to your improvement areas, providing you with a reading list tailored to your actual development work.

6. **Sends to Your Email**: Automatically delivers the complete report to your email address so you can reference it anytime, anywhere.

## Prerequisites

### Required Environment Variables

Before using this skill, you need to set up the following environment variables for email sending:

| Variable          | Description                              | Example                 |
| ----------------- | ---------------------------------------- | ----------------------- |
| `EMAIL_HOST`      | SMTP server address                      | `smtp.gmail.com`        |
| `EMAIL_PORT`      | SMTP port (optional, defaults to 587)    | `587`                   |
| `EMAIL_USER`      | Your email address                       | `your-email@gmail.com`  |
| `EMAIL_PASSWORD`  | App password (NOT your regular password) | `xxxx xxxx xxxx xxxx`   |
| `EMAIL_FROM_NAME` | Sender name displayed in email           | `Claude Code`           |
| `EMAIL_TO`        | Recipient email address                  | `recipient@example.com` |

**Email subject prefix** will be auto-generated based on report content (e.g., "Dev Growth Report - TypeScript Patterns").

### How to Get an App Password

**For Gmail:**

1. Enable 2-Factor Authentication on your Google account
2. Go to Google Account → Security → App passwords
3. Select "Mail" and "Other (Custom name)"
4. Enter "Claude Code" and click Generate
5. Copy the 16-character password (spaces optional)

**For Outlook/Office 365:**

1. Go to Microsoft Account → Security → App passwords
2. Create a new app password
3. Use this password in your configuration

**For other email providers:**

- Check your email provider's documentation for SMTP settings and app password creation

## How to Use

Ask Claude to analyze your recent coding work:

```
Analyze my developer growth from my recent chats
```

Or be more specific about which time period:

```
Analyze my work from today and suggest areas for improvement
```

The skill will generate a formatted report with:

- Overview of your recent work
- Key improvement areas identified
- Specific recommendations for each area
- Curated learning resources from HackerNews and Stack Overflow
- Action items you can focus on

## Instructions

When a user requests analysis of their developer growth or coding patterns from recent work:

1. **Access Chat History**
   
   Read the chat history from `~/.claude/history.jsonl`. This file is a JSONL format where each line contains:
   
   - `display`: The user's message/request
   - `project`: The project being worked on
   - `timestamp`: Unix timestamp (in milliseconds)
   - `pastedContents`: Any code or content pasted
   
   Filter for entries from the past 24-48 hours based on the current timestamp.

2. **Analyze Work Patterns**
   
   Extract and analyze the following from the filtered chats:
   
   - **Projects and Domains**: What types of projects was the user working on? (e.g., backend, frontend, DevOps, data, etc.)
   - **Technologies Used**: What languages, frameworks, and tools appear in the conversations?
   - **Problem Types**: What categories of problems are being solved? (e.g., performance optimization, debugging, feature implementation, refactoring, setup/configuration)
   - **Challenges Encountered**: What problems did the user struggle with? Look for:
     - Repeated questions about similar topics
     - Problems that took multiple attempts to solve
     - Questions indicating knowledge gaps
     - Complex architectural decisions
   - **Approach Patterns**: How does the user solve problems? (e.g., methodical, exploratory, experimental)

3. **Identify Improvement Areas**
   
   Based on the analysis, identify 3-5 specific areas where the user could improve. These should be:
   
   - **Specific** (not vague like "improve coding skills")
   - **Evidence-based** (grounded in actual chat history)
   - **Actionable** (practical improvements that can be made)
   - **Prioritized** (most impactful first)
   
   Examples of good improvement areas:
   
   - "Advanced TypeScript patterns (generics, utility types, type guards) - you struggled with type safety in [specific project]"
   - "Error handling and validation - I noticed you patched several bugs related to missing null checks"
   - "Async/await patterns - your recent work shows some race conditions and timing issues"
   - "Database query optimization - you rewrote the same query multiple times"

4. **Generate Report**
   
   Create a comprehensive report with this structure:
   
   ```markdown
   # Your Developer Growth Report
   
   **Report Period**: [Yesterday / Today / [Custom Date Range]]
   **Last Updated**: [Current Date and Time]
   
   ## Work Summary
   
   [2-3 paragraphs summarizing what the user worked on, projects touched, technologies used, and overall focus areas]
   
   Example:
   "Over the past 24 hours, you focused primarily on backend development with three distinct projects. Your work involved TypeScript, React, and deployment infrastructure. You tackled a mix of feature implementation, debugging, and architectural decisions, with a particular focus on API design and database optimization."
   
   ## Improvement Areas (Prioritized)
   
   ### 1. [Area Name]
   
   **Why This Matters**: [Explanation of why this skill is important for the user's work]
   
   **What I Observed**: [Specific evidence from chat history showing this gap]
   
   **Recommendation**: [Concrete step(s) to improve in this area]
   
   **Time to Skill Up**: [Brief estimate of effort required]
   
   ---
   
   [Repeat for 2-4 additional areas]
   
   ## Strengths Observed
   
   [2-3 bullet points highlighting things you're doing well - things to continue doing]
   
   ## Action Items
   
   Priority order:
   1. [Action item derived from highest priority improvement area]
   2. [Action item from next area]
   3. [Action item from next area]
   
   ## Learning Resources
   
   [Will be populated in next step]
   ```

5. **Search for Learning Resources**
   
   Use Rube MCP to search HackerNews and Stack Overflow for articles related to each improvement area:
   
   - For each improvement area, construct a search query targeting high-quality resources
   - Search HackerNews using RUBE_SEARCH_TOOLS with queries like:
     - "Learn [Technology/Pattern] best practices"
     - "[Technology] advanced patterns and techniques"
     - "Debugging [specific problem type] in [language]"
   - Search Stack Overflow using queries like:
     - "site:stackoverflow.com [Technology/Pattern] best practices"
     - "site:stackoverflow.com [specific problem] solution"
     - "site:stackoverflow.com [language] advanced tips"
   - Prioritize posts with high engagement (comments, upvotes)
   - For each area, include 2-3 most relevant articles with:
     - Article title
     - Publication date
     - Brief description of why it's relevant
     - Link to the article
   
   Add this section to the report:
   
   ```markdown
   ## Curated Learning Resources
   
   ### For: [Improvement Area]
   
   1. **[Article Title]** - [Date]
      [Description of what it covers and why it's relevant to your improvement area]
      [Link]
   
   2. **[Article Title]** - [Date]
      [Description]
      [Link]
   
   [Repeat for other improvement areas]
   ```

6. **Present the Complete Report**
   
   Deliver the report in a clean, readable format that the user can:
   
   - Quickly scan for key takeaways
   - Use for focused learning planning
   - Reference over the next week as they work on improvements
   - Share with mentors if they want external feedback

7. **Send Report to Email**
   
   Use Rube MCP to send the complete report to the user's email:
   
   - Read email configuration from environment variables: `EMAIL_HOST`, `EMAIL_PORT` (optional, defaults to 587), `EMAIL_USER`, `EMAIL_PASSWORD`, `EMAIL_FROM_NAME`, `EMAIL_TO`
   - If any required environment variable is missing, inform the user and ask them to configure it
   - Generate email subject based on report content (max 10 characters, e.g., "DevGrowth Report" or "Growth Insights")
   - Use RUBE_MULTI_EXECUTE_TOOL or appropriate email tool to send the report:
     - Subject: "[Dev Growth] {auto-generated-prefix} - [Date]"
     - Body: The complete formatted report as HTML or plain text
     - From: The configured email address
     - To: The recipient email address
   - Confirm delivery in the CLI output
   
   This ensures the user has the report in their email inbox and can reference it throughout the week.

## Example Usage

### Input

```
分析我最近的开发者成长情况
```

### Output

```markdown
# 你的开发者成长报告

**报告周期**: 2024年11月9-10日
**最后更新**: 2024年11月10日 21:15 UTC

## 工作总结

过去两天，你主要专注于后端基础设施和API开发。主要项目是一个开源展示应用，在连接管理、UI改进和部署配置方面取得了重大进展。你使用了TypeScript、React和Node.js，应对了从数据安全到响应式设计的各种挑战。你的工作体现了功能实现和技术债务处理的平衡。

## 提升领域（按优先级排序）

### 1. 高级TypeScript模式与类型安全

**为什么重要**: TypeScript是你工作的核心，利用其高级特性（泛型、工具类型、条件类型、类型守卫）可以显著提高代码可靠性，减少运行时错误。更好的类型安全在编译时就能捕获bug，而不是在生产环境。

**我观察到的问题**: 在最近的对话中，你处理连接数据结构时，在正确类型化身份验证配置方面遇到了一些困难。你还需要多次迭代联合类型以处理不同的连接状态。有机会更有效地使用可辨识联合和类型守卫。

**建议**: 学习TypeScript的高级类型系统，特别是工具类型（Omit、Pick、Record）、条件类型和可辨识联合。将这些模式应用到你的连接配置处理和认证状态管理中。

**预计学习时间**: 5-8小时专注学习与实践

### 2. 前端敏感数据处理与信息隐藏

**为什么重要**: 你发现并修复了一个安全问题，敏感连接数据被显示在控制台中。防止信息泄露对于处理用户凭据和API密钥的应用程序至关重要。良好的实践可以防止安全事件和用户信任违规。

**我观察到的问题**: 你发现"你的应用"页面显示了完整的连接数据，包括身份验证配置。这显示了良好的安全意识，下一步是将此纳入你处理敏感信息时的默认思维。

**建议**: 复习处理前端应用程序敏感数据的安全最佳实践。创建可重用的模式，用于在显示前过滤/屏蔽敏感信息。考虑实现一个明确白名单可显示内容的安全数据层。

**预计学习时间**: 3-4小时

### 3. 组件架构与响应式UI模式

**为什么重要**: 你正在设计的UI需要在不同屏幕尺寸和用户交互下工作。强大的组件架构使构建复杂UI而不出现bug变得更加容易，并提高了可维护性。

**我观察到的问题**: 你在处理"市场"UI（以前的浏览工具），根据设计图重新创建它。你还发现并修复了内容溢出容器的滚动问题。有机会加强你对布局约束和响应式设计模式的理解。

**建议**: 学习React组件组合模式和CSS布局最佳实践（特别是flexbox和grid）。专注于防止溢出问题的容器查询和响应式模式。了解组件组合库和设计系统方法。

**预计学习时间**: 6-10小时（取决于深度）

## 观察到优势

- **安全意识**: 你主动在问题变成问题之前识别了数据泄露问题
- **迭代改进**: 你有条不紊地处理UI需求，提出澄清问题并改进设计
- **全栈能力**: 你能够轻松处理后端API、前端UI和部署相关工作
- **解决问题的方法**: 你将复杂任务分解为可管理的步骤

## 行动项

优先级顺序：
1. 花1-2小时学习TypeScript工具类型和可辨识联合；应用到你的连接数据结构
2. 为你的项目记录安全模式（哪些数据可以安全显示、过滤/屏蔽函数）
3. 学习一篇关于高级React模式的文章，并将一个模式应用到当前的UI工作中
4. 建立一个专注于类型安全和数据安全的代码审查清单，供未来PR使用

## 精选学习资源

### 主题：高级TypeScript模式

1. **TypeScript高级类型：泛型、工具类型和条件类型** - HackerNews，2024年10月
   深入探索TypeScript类型系统，包含实用示例和实际应用。涵盖可辨识联合、类型守卫和确保复杂应用程序编译时安全的模式。
   [讨论链接]

2. **在TypeScript中构建类型安全API** - HackerNews，2024年9月
   使用TypeScript设计API的实用指南，能及早捕获错误。与你的连接配置工作特别相关。
   [讨论链接]

### 主题：前端敏感数据处理

1. **防止Web应用程序信息泄露** - HackerNews，2024年8月
   前端应用程序数据安全综合指南，包括过滤敏感信息、安全日志和审计跟踪。
   [讨论链接]

2. **OAuth和API密钥管理最佳实践** - HackerNews，2024年7月
   如何在应用程序中安全处理身份验证令牌和API密钥，包含不同框架的示例。
   [讨论链接]

### 主题：组件架构与响应式设计

1. **高级React模式：组合优于配置** - HackerNews
   探索可扩展的组件组合策略，包含现代React模式的示例。
   [讨论链接]

2. **CSS布局精通：Flexbox、Grid和容器查询** - HackerNews，2024年10月
   学习防止溢出问题并在所有屏幕尺寸上工作的响应式设计模式。
   [讨论链接]
```

## Tips and Best Practices

- Run this analysis once a week to track your improvement trajectory over time
- Pick one improvement area at a time and focus on it for a few days before moving to the next
- Use the learning resources as a study guide; work through the recommended materials and practice applying the patterns
- Revisit this report after focusing on an area for a week to see how your work patterns change
- The learning resources are intentionally curated for your actual work, not generic topics, so they'll be highly relevant to what you're building

## How Accuracy and Quality Are Maintained

This skill:

- Analyzes your actual work patterns from timestamped chat history
- Generates evidence-based recommendations grounded in real projects
- Curates learning resources that directly address your identified gaps
- Focuses on actionable improvements, not vague feedback
- Provides specific time estimates based on complexity
- Prioritizes areas that will have the most impact on your development velocity
