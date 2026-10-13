# 贡献指南

感谢你对 feedback-collect-classify 技能的关注！本文档将指导你如何为这个项目做出贡献。

## 如何贡献

### 报告问题

如果你发现了 bug 或有功能建议，请通过以下方式报告：

1. **Bug 反馈**: 点击 NIO Chat 回答下方的"踩"按钮
2. **功能建议**: 访问 [NIO Chat 许愿池](https://nio.feishu.cn/wiki/OLFpwEhdeiiCpdkLJdfc4hLpnJb)
3. **技术讨论**: 联系产品负责人 NI.SONG 宋旎 或研发负责人 Yujie ZHOU 周毓杰

### 提交代码

如果你想直接贡献代码，请遵循以下流程：

1. **Fork 项目**: 创建你自己的分支
2. **创建分支**: 为你的功能或修复创建新分支
3. **编写代码**: 遵循代码规范和最佳实践
4. **测试**: 确保你的更改通过所有测试
5. **提交**: 创建清晰的提交信息
6. **发起 PR**: 提供详细的描述和测试结果

## 开发指南

### 环境设置

1. **克隆仓库**
```bash
git clone <repository-url>
cd feedback-collect-classify
```

2. **安装依赖**
```bash
npm install
```

3. **运行测试**
```bash
npm test
```

### 代码结构

```
feedback-collect-classify/
├── SKILL.md                 # 技能主要说明文档
├── README.md                # 用户指南
├── CHANGELOG.md            # 版本历史
├── CONTRIBUTING.md         # 贡献指南（本文件）
├── .nio-skill.json         # 技能元数据
├── examples/               # 使用示例
│   └── basic-usage.md
└── references/             # 技术参考文档
    ├── classification-rules.md
    ├── field-mapping-guide.md
    ├── image-processing-guide.md
    └── technical-architecture.md
```

### 编码规范

#### 文件命名
- 使用小写字母和连字符：`file-name.md`
- 文档文件使用 `.md` 扩展名

#### 代码风格
- 使用清晰的变量和函数命名
- 添加适当的注释
- 遵循现有的代码风格

#### 文档规范
- 使用 Markdown 格式
- 提供清晰的标题和结构
- 包含代码示例
- 更新相关文档

### 测试

#### 单元测试
为核心功能编写单元测试：

```typescript
describe('FeedbackClassifier', () => {
  it('should classify technical issues correctly', () => {
    const result = classify('点击保存按钮没有反应');
    expect(result.category).toBe('技术问题');
    expect(result.confidence).toBeGreaterThan(70);
  });
});
```

#### 集成测试
测试完整的处理流程：

```typescript
describe('Feedback Processing Pipeline', () => {
  it('should process feedback from start to end', async () => {
    const config = createTestConfig();
    const result = await processFeedback(config);
    expect(result.collected_items.length).toBeGreaterThan(0);
  });
});
```

### 文档更新

当你添加新功能或修改现有功能时，请更新相关文档：

1. **SKILL.md**: 更新技能说明
2. **README.md**: 更新用户指南
3. **examples/**: 添加新的使用示例
4. **references/**: 更新技术文档
5. **CHANGELOG.md**: 记录版本变更

## 提交规范

### 提交信息格式

使用清晰的提交信息：

```
<type>: <subject>

<body>

<footer>
```

### 类型 (type)

- `feat`: 新功能
- `fix`: 修复 bug
- `docs`: 文档更新
- `style`: 代码格式调整
- `refactor`: 代码重构
- `test`: 测试相关
- `chore`: 构建/工具相关

### 示例

```
feat: 添加自定义分类规则支持

- 允许用户定义自己的分类规则
- 支持规则优先级配置
- 添加规则验证机制

Closes #123
```

## 代码审查

### 审查要点

1. **功能正确性**: 代码是否实现了预期功能
2. **代码质量**: 代码是否清晰、可维护
3. **测试覆盖**: 是否有足够的测试
4. **文档完整性**: 文档是否更新
5. **性能影响**: 是否有性能问题

### 审查流程

1. 自动检查（CI/CD）
2. 同行审查
3. 测试验证
4. 文档审核
5. 最终批准

## 发布流程

### 版本号规则

遵循语义化版本控制 (Semantic Versioning)：

- **主版本号**: 不兼容的 API 变更
- **次版本号**: 向下兼容的功能新增
- **修订号**: 向下兼容的问题修复

### 发布步骤

1. **更新版本号**: 修改 `.nio-skill.json` 中的版本号
2. **更新 CHANGELOG**: 记录所有变更
3. **创建标签**: `git tag v1.0.0`
4. **推送标签**: `git push origin v1.0.0`
5. **发布**: 创建 GitHub Release

## 社区准则

### 行为准则

1. **尊重他人**: 尊重所有贡献者
2. **建设性反馈**: 提供建设性的反馈
3. **协作精神**: 鼓励协作和知识分享
4. **开放心态**: 对新想法保持开放

### 沟通渠道

- **技术问题**: 联��研发负责人
- **产品建议**: 联系产品负责人
- **Bug 反馈**: 使用 NIO Chat 反馈渠道

## 常见问题

### Q: 如何开始贡献？

A: 从小处开始！你可以从修复文档错别字、改进示例代码、或修复小 bug 开始。

### Q: 代码审查需要多长时间？

A: 我们通常在 1-3 个工作日内完成代码审查。

### Q: 可以贡献新功能吗？

A: 当然可以！但在开始之前，建议先创建一个 issue 讨论你的想法。

### Q: 如何获取帮助？

A: 你可以通过 issue、邮件或即时通讯工具联系我们。

## 许可证

本技能为 NIO Chat 内部技能，仅供蔚来员工内部使用。所有贡献都遵循相同的许可证。

## 致谢

感谢所有为 feedback-collect-classify 技能做出贡献的人员！

---

**联系方式**:
- 产品负责人: NI.SONG 宋旎
- 研发负责人: Yujie ZHOU 周毓杰
- 许愿池: https://nio.feishu.cn/wiki/OLFpwEhdeiiCpdkLJdfc4hLpnJb

---

**最后更新**: 2026-08-19