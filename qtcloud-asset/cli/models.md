# qtcloud-asset CLI 领域模型

## 一句话概括

CLI 的核心模型是**契约**——它不是运行时对象，而是放在项目里的 `.quanttide/asset/contract.yaml` 文件，描述「这个项目有哪些资产、该长什么样」，CLI 拿它来扫描和验证。

## 契约：项目自述文件

契约模型是 YAML 而非代码对象：

```rust
pub struct ContractSchema {
    pub assets: HashMap<String, AssetConfig>,   // 项目有哪些资产
    pub skills: HashMap<String, SkillConfig>,   // 有哪些可执行技能
    pub validation: Option<ValidationConfig>,   // 验证策略
}
```

例：`.quanttide/asset/contract.yaml` 里登记了 40 个 OSS 桶（`qtaccount-studio`、`qtcloud-private` 等）。CLI 的 `scan` 命令扫描目录产出资产清单，`validate` 命令把本地结构逐条和契约比对，不符合就报错——「实际结构必须和契约一致」靠它落地。
