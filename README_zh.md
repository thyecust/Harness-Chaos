# Harness Chaos

我们的目标是使用混沌工程对目前的Harness项目进行质量保证，比如

- [Claude Code](https://github.com/anthropics/claude-code)
- [OpenCode](https://github.com/anomalyco/opencode)
- [Codex](https://github.com/openai/codex)
- [Copilot CLI](https://github.com/github/copilot-cli)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)
- [Agents SDK](https://github.com/openai/openai-agents-python)
- [Pi](https://github.com/badlogic/pi-mono)

## 混沌工程原则

根据[混沌工程原则](./doc/principles_of_chaos_eng_zh.md)，我们的目标是揭示系统的弱点，比如

1. 服务不可用时没有正确的回滚/单点故障的级联失败，可能表现为Agent无法继续工作，例如 https://github.com/anomalyco/opencode/issues/4781
2. 不当的超时设置导致的重新风暴，可能表现为`SSE read timed out`，例如 https://github.com/anomalyco/opencode/issues/17307
3. 内存泄漏，例如 https://github.com/anomalyco/opencode/issues/20695
4. 多个实例的数据库之间存在的数据竞争，例如 https://github.com/anomalyco/opencode/issues/14194

为了揭示弱点，我们进行这样的实验

1. 用系统在正常行为下的输出来定义“稳定状态”
2. 确保在控制组和实验组都能维持这种稳定状态
3. 在实验组中引入一些变量来反映现实世界中的事件，如服务器崩溃、硬盘损坏、网络连接断开等
4. 通过控制组和实验组的输出差异来说明实验组不处于“稳定状态”

## 混沌工程实践

我们的目标是维护一个数据库，包括

- 弱点的表现
- 弱点的影响范围
- 相关依赖的版本
- 完整的实验步骤和结果
- GitHub上的Issue和修复MR
