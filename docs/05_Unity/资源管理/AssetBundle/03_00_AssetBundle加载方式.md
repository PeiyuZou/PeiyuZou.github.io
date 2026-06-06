### 总体对比

AssetBundle有**4大类、8种**加载方式：

| 分类     | 同步方法       | 异步方法                   | 适用场景          |
|----------|----------------|----------------------------|-------------------|
| 文件加载 | LoadFromFile   | LoadFromFileAsync          | AB在本地磁盘      |
| 内存加载 | LoadFromMemory | LoadFromMemoryAsync        | 很少，整包AES加密 |
| 流加载   | LoadFromStream | LoadFromStreamAsync        | 自定义流解密      |
| 网络加载 | -              | UnityWebRequestAssetBundle | 从网络下载AB      |
| 已废弃   | -              | WWW                        | 已废弃            |

### 性能对比

| 加载方式            | 内存占用  | 加载速度  | 阻塞线程 | 推荐度 |
|---------------------|-----------|-----------|----------|--------|
| LoadFromFile        | S（最低） | S（最快） | 是       | S      |
| LoadFromFileAsync   | S（最低） | S-（快）  | 否       | S      |
| LoadFromMemory      | C（双倍） | C（慢）   | 是       | C      |
| LoadFromMemoryAsync | C（双倍） | C（慢）   | 否       | C      |
| LoadFromStream      | A（低）   | B（中）   | 是       | B      |
| LoadFromStreamAsync | A（低）   | B（中）   | 否       | B      |
| UnityWebRequest     | A（低）   | B（网络） | 否       | S      |

### 选择建议速查表

| 场景 | 首选方案 | 备选方案 | 避免使用 |
|------|---------|---------|---------|
| 本地小文件 (< 5MB) | LoadFromFile | LoadFromFileAsync | LoadFromMemory |
| 本地大文件 (> 5MB) | LoadFromFileAsync | LoadFromFile | LoadFromMemory |
| 网络下载 | UnityWebRequest | - | WWW |
| 需要解密 | LoadFromStream | LoadFromMemory | - |
| 需要进度显示 | LoadFromFileAsync<br>UnityWebRequest | - | LoadFromFile |
| 移动平台 | LoadFromFileAsync<br>UnityWebRequest | LoadFromFile | LoadFromMemory |
| PC平台 | LoadFromFile<br>LoadFromFileAsync | - | LoadFromMemory |
| 热更新 | UnityWebRequest | - | 其他方式 |