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
