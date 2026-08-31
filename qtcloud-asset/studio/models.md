# qtcloud-asset Studio 领域模型

## 一句话概括

Studio 几乎没有自己的领域模型——它是 Provider 模型的**展示镜像**：同一份 JSON，客户端定义同样的类。它独有的建模在展示层：分类、排序、筛选、契约页面。

## 展示镜像

`lib/main.dart` 里的 `Bucket`、`OssObject`、`FolderShare`、`ProviderUser` 与 Provider 的 schema 一一对应，字段名一致，`fromJson` 直接解析。

```dart
class Bucket {
  final String name;
  final String region;
  final String storageClass;
  final String createdAt;
}
```

唯一的本地派生逻辑在 `Bucket.category`：按桶名后缀推断用途分类——`-studio` → Studio、`-private` → Private、`-site` → Site、`-provider` → Provider、其余 → Other，并据此派生图标与主题色。这是 Provider `isPrivateBucket` 同一条分类逻辑在客户端的一份重复实现。

`OssObject` 也有一个展示派生：`isDir`（key 以 `/` 结尾）判断目录，目录层级下钻由此展开。

## 交互模型

排序与筛选是纯展示层模型：`BucketSortMode` 枚举（none/name/createdAt/createdAtThenName）支持桶列表独立排序开关，用途分类筛选条按 `category` 过滤，对象列表支持前缀搜索与日期/大小排序。这些模型不落盘、不涉及业务规则，只服务界面。

## 契约页面

`lib/screens/asset_contract_screen.dart` 是「资产注册表」页面：按记忆模型分层（宪法层/法律层/法理层）展示 Bylaw、Handbook、Tutorial、Specification 等资产卡片。当前内容是静态硬编码的格子，标注「与 .gitmodules 对齐」，尚未真正由契约数据驱动。

## 平台隔离

平台差异逻辑走接口 + 双实现：`download_url`（io/web）、`login_redirect`（io/web）、`provider_http_client`（io/web）各分三文件，业务层不感知平台。ZIP 打包（`client_zip.dart` 的 `StoredZipEntry`/`buildStoredZip`）是分享页下载全部时的纯前端工具，无状态模型。

## 与数字资产治理标准对照

Studio 是展示层，它的模型是 Provider 的镜像加上交互模型。对照数字资产治理标准（quanttide-asset `docs/specification`），主要缺口是分类逻辑重复实现、契约页面未真正由契约驱动。

### category 派生（Bucket.category）

`lib/main.dart` 里按桶名后缀推断分类（`-studio`/`-private`/`-site`/`-provider`）并派生图标颜色，对应标准的 `category`——「用于结构化分类和权限控制」。这是 Provider `isPrivateBucket` 同一逻辑的**重复实现**：分类在两端各硬编码一份。标准建模是资产属性驱动展示，应改为消费 Provider 返回的 category/tags 字段，删除客户端推断逻辑。

### AssetContractScreen

`lib/screens/asset_contract_screen.dart` 的资产注册表页面，对应标准契约的 `schemas`——「结构即界面」，契约驱动的目录镜像。当前是静态硬编码的资产卡片格子，未读取 `.quanttide/asset/contract.yaml`。演化方向：按契约 `schemas`/`discovery` 结果渲染，成为契约驱动界面的落地页。

### 其余模型

ProviderUser / FolderShare（身份与分享，标准无对应，保持独立建模）、交互模型（BucketSortMode、筛选、分页、isDir，纯展示层不冲突）、平台隔离双实现（已符合标准分层，无需改动）。

### 结论

对得上的：镜像模型、平台隔离双实现。

对不上的：category 分类逻辑在客户端重复硬编码（应改为消费资产属性）、契约页面静态未由契约驱动。

升级动作见 index.md 演化路线图（阶段四资产目录化后，Studio 由目录数据驱动展示）。
