---
title: "stitch ui设计初体验"
source: "https://linux.do/t/topic/2286943"
author:
  - "[[withnoidea]]"
published:
created: 2026-06-02
description: "3323acc7e72e7563fc638150897d94eb1920×1043 374 KB 分享一下我用提示词给stitch生成的ui，下面是用claude code + gpt5.5生成了ios app，测试下来感觉还不错。 微信图片_20260529122043_134"
tags:
  - "clippings"
---
[![3323acc7e72e7563fc638150897d94eb](https://cdn3.ldstatic.com/optimized/4X/2/e/0/2e02ebc2068dc407a51bef68198319ffd5cc73cb_2_690x374.jpeg)

3323acc7e72e7563fc638150897d94eb1920×1043 374 KB

](https://cdn3.ldstatic.com/original/4X/2/e/0/2e02ebc2068dc407a51bef68198319ffd5cc73cb.jpeg "3323acc7e72e7563fc638150897d94eb")

**分享一下我用提示词给stitch生成的ui，下面是用claude code + gpt5.5生成了ios app，测试下来感觉还不错。**

[![微信图片_20260529122043_1349_14](https://cdn3.ldstatic.com/optimized/4X/f/8/f/f8f455ebe94d3cb9b255a494b47d95fd180082dd_2_231x500.jpeg)

微信图片\_20260529122043\_1349\_141170×2532 213 KB

](https://cdn3.ldstatic.com/original/4X/f/8/f/f8f455ebe94d3cb9b255a494b47d95fd180082dd.jpeg "微信图片_20260529122043_1349_14")

[![微信图片_20260529122040_1347_14](https://cdn3.ldstatic.com/optimized/4X/4/b/3/4b3a388209998515093f908ff0e7e37dd076734e_2_231x500.jpeg)

微信图片\_20260529122040\_1347\_141170×2532 427 KB

](https://cdn3.ldstatic.com/original/4X/4/b/3/4b3a388209998515093f908ff0e7e37dd076734e.jpeg "微信图片_20260529122040_1347_14")

[![微信图片_20260529122039_1346_14](https://cdn3.ldstatic.com/optimized/4X/e/1/1/e11256c74ec230908da9ac03e3cbf7d272a72953_2_231x500.png)

微信图片\_20260529122039\_1346\_141170×2532 248 KB

](https://cdn3.ldstatic.com/original/4X/e/1/1/e11256c74ec230908da9ac03e3cbf7d272a72953.png "微信图片_20260529122039_1346_14")

[![微信图片_20260529122042_1348_14](https://cdn3.ldstatic.com/optimized/4X/a/8/3/a8361b5c05c4c850878eefbc757b833b4b4543cb_2_231x500.jpeg)

微信图片\_20260529122042\_1348\_141170×2532 427 KB

](https://cdn3.ldstatic.com/original/4X/a/8/3/a8361b5c05c4c850878eefbc757b833b4b4543cb.jpeg "微信图片_20260529122042_1348_14")

[![微信图片_20260529122038_1345_14](https://cdn3.ldstatic.com/optimized/4X/3/b/b/3bbf34692ae0cb39c78dd102cd12d7c62d5b63d8_2_231x500.jpeg)

微信图片\_20260529122038\_1345\_141170×2532 208 KB

](https://cdn3.ldstatic.com/original/4X/3/b/b/3bbf34692ae0cb39c78dd102cd12d7c62d5b63d8.jpeg "微信图片_20260529122038_1345_14")

**上述是我在一篇贴子中的评论，佬友希望我做个教程，这里发表一下个人使用流程，不保证效果，仅供参考。**

我本身不太擅长凭空想象 UI，通常需要先看到一些现有设计，才能联想到自己想要的效果。所以设计UI 时，我更看重 AI 一次性生成的能力，**“初稿及终稿”**，希望通过需求分析、设计规范和提示词优化，用比较低的成本得到一个可用方案。

大概思路是先按照软件开发流程来拆：

**需求分析、概要设计、详细设计**。和 UI 关系比较大的主要是需求分析和详细设计。前者用来明确 App 的目标用户、使用场景、核心功能和用户痛点；后者则落到具体视觉风格、页面结构、交互方式和 UI 提示词。

我一开始尝试过直接让 AI 生成 UI，但效果比较随机。后来发现先让 AI 帮我把产品需求、核心页面、视觉风格和设计规范梳理清楚，再让它生成 UI 提示词，最终效果会稳定很多。

工具方面我使用的是Cherry Studio进行需求分析和提示词生成，试了一下发现能配置上anyrouter，里面有一些预设助手，尤其是"智囊团"，对我这种缺少灵感的人挺有帮助，它可以从不同人不同角度给出建议。

[![image](https://cdn3.ldstatic.com/optimized/4X/d/a/f/daf1179a02094515744a653bbc923019a285e2d7_2_690x310.png)

image1337×601 30.8 KB

](https://cdn3.ldstatic.com/original/4X/d/a/f/daf1179a02094515744a653bbc923019a285e2d7.png "image")

实际操作时我的流程很简单，我只是和"智囊团"进行了几轮对话，让它先帮我做骑行 App 的需求分析，再给出技术框架和页面结构建议，最后让"智囊团"帮我生成ui提示词，图标我也是让智囊团帮我设计的image2提示词。

至此就基本结束了，提示词直接丢给stitch就可以生成初稿了。

之前提到的**初稿及终稿可能并不现实**，这一次的设计总体来说运气比较好，大部分的结果符合我的预期，但实际上初稿有按钮显示不全，页面不完整,图标白色背景没去除等问题，还是需要不断调整。以下是"智囊团"生成的对话，仅供参考。

[Assistant.pdf](https://linux.do/uploads/short-url/wWU2mBZlTuCddAxZpU9SDeqAxER.pdf)<iframe src="blob:https://linux.do/497cfa45-3537-472c-8cdf-f58e3a8fff80" height="500" loading="lazy" class="pdf-preview"></iframe>

这里分享一下之前看到的[【木子狸的Vibe Coding随笔】 总1w5字 VibeCoding 真解 - 文档共建 - LINUX DO](https://linux.do/t/topic/1752642)帖子，在后续vibecoding的过程中可以供大家进行参考。

---

## Comments

> **Muzilee** · [2026-06-01](https://linux.do/t/topic/2286943/2?u=rubyceng)
> 
> 可以试着设计一个 软件工程师的 智囊
> 
> 或者设定为 熟练使用 Software Engineering Tenth Edition 这本书的 人
> 
> 然后写 故事和场景