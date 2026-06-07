# ReAct: Synergizing Reasoning and Acting in Language Models

## Abstract

While large language models have demonstrated impressive capabilities across tasks in language understanding and interactive decision making, their abilities for reasoning (e.g. chain-of-thought prompting) and acting (e.g. action plan generation) have primarily been studied as separate topics. In this paper, we explore the use of LLMs to generate both reasoning traces and task-specific actions in an interleaved manner, allowing for greater synergy between the two.

## Introduction

A unique feature of human intelligence is the ability to seamlessly combine task-oriented actions with verbal reasoning (or inner speech), which has been theorized to play an important role in human cognition for self-regulation or strategizing. Consider the example of cooking a meal in a new kitchen. Between actions (opening cabinets, checking for ingredients), you might reason ("Now that I've found the soy sauce, I need to find the sesame oil which is likely in the same cabinet") to track progress and guide your next action.

## Method

ReAct prompts LLMs to generate both verbal reasoning traces and actions pertaining to a task in an interleaved manner. This allows the model to perform dynamic reasoning to create, maintain, and adjust high-level plans for acting, while also interact with the external environments to incorporate additional information into reasoning.

### The Role of Intermediate Signals

The interleaving of reasoning and acting creates a rich signal at every step. Unlike purely internal reasoning chains, ReAct receives external observations after each action. These observations provide intermediate signals that ground the reasoning process in actual outcomes rather than hypothetical ones. The model can detect when its reasoning has gone astray by comparing expected observations with actual ones.

This intermediate feedback mechanism serves as a form of error detection and correction. When the model takes an action and receives an unexpected observation, it can revise its reasoning and choose a different path. The combination of internal deliberation and external feedback creates a more robust problem-solving process than either alone.

### Comparison to Pure Reasoning

The key distinction between ReAct and chain-of-thought approaches lies in the treatment of intermediate reasoning. Chain-of-thought generates a complete linear reasoning chain internally before producing a final answer. ReAct interleaves reasoning steps with actions and observations from the environment. This interleaving means that:

1. Reasoning can be grounded in actual observations rather than assumptions
2. The model can course-correct when observations don't match expectations  
3. External tool outputs provide information that would be impossible to generate through reasoning alone
4. The reasoning trace becomes a mix of planning, reflection, and integration of new information

## Results

ReAct outperforms chain-of-thought prompting on diverse benchmarks including question answering (HotpotQA, FEVER) and interactive decision making tasks (ALFWorld, WebShop). The interleaving of reasoning and acting provides complementary benefits — reasoning helps the model track progress and determine what to do next, while acting provides grounding from the environment that pure reasoning cannot access.

## Error Analysis

The most common failure mode in ReAct involves the model generating plausible but incorrect reasoning that leads to wrong actions. However, compared to pure reasoning approaches, ReAct failures tend to be more recoverable because subsequent observations can reveal the error.
