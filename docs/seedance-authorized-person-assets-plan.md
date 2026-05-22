# Seedance 2.0 已授权真人素材接入调查

## 背景

Seedance 2.0 系列模型对含真人人脸的参考图/视频有额外限制。项目后续可能需要支持“真人授权 ID”或“真人授权素材”作为视频生成参考源，因此本次先根据 `docs/seedance-api/` 中的本地文档整理接入方案。

## 文档依据

主要参考：

- `docs/seedance-api/创建视频生成任务API.md`
- `docs/seedance-api/ark_seedance2.0.md`

关键结论来自：

- `docs/seedance-api/创建视频生成任务API.md:66-75`：Seedance 2.0 不支持直接上传含真人人脸的参考图/视频，但支持已授权真人素材作为输入。
- `docs/seedance-api/创建视频生成任务API.md:144-149`：图片输入的 `url` 可传素材 ID，格式为 `asset://<ASSET_ID>`。
- `docs/seedance-api/创建视频生成任务API.md:226-230`：视频输入的 `url` 可传素材 ID，格式为 `asset://<ASSET_ID>`。
- `docs/seedance-api/ark_seedance2.0.md:2180-2210`：已授权真人图片、视频、音频素材均通过 `asset://<asset ID>` 放入 `content.<模态>_url.url` 使用。
- `docs/seedance-api/ark_seedance2.0.md:2216-2218`：提示词中不支持直接用 Asset ID 指代素材，必须用“图片 n / 视频 n / 音频 n”的素材序号引用。

## 核心结论

所谓“真人授权 ID”在视频生成调用中不是独立字段，例如不是：

```json
{
  "authorization_id": "..."
}
```

而是平台真人认证和本人授权完成后，素材入库得到的 Ark asset ID。调用 Seedance 2.0 时，应把它作为素材 URL 传入：

```text
asset://<asset ID>
```

示例：

```json
{
  "type": "image_url",
  "image_url": {
    "url": "asset://asset-xxxx"
  },
  "role": "reference_image"
}
```

视频和音频同理：

```json
{
  "type": "video_url",
  "video_url": {
    "url": "asset://asset-xxxx"
  },
  "role": "reference_video"
}
```

```json
{
  "type": "audio_url",
  "audio_url": {
    "url": "asset://asset-xxxx"
  },
  "role": "reference_audio"
}
```

## 当前项目相关位置

当前根项目 legacy 视频生成入口主要在：

- `main.py`
  - `CanvasVideoRequest`
  - `POST /api/canvas-video`

现有请求字段包括：

```python
images: List[AIReference] = []
videos: List[str] = []
reference_audio_urls: List[str] = []
```

其中 `AIReference` 当前结构为：

```python
class AIReference(BaseModel):
    url: str = ""
    name: str = ""
    role: str = ""
```

这意味着 MVP 可以优先复用现有字段：

- 已授权真人图片素材：放入 `images[].url`
- 已授权真人视频素材：放入 `videos[]`
- 已授权真人音频素材：放入 `reference_audio_urls[]`

统一使用 `asset://<asset ID>` 形式。

## 推荐 MVP 方案

### 1. 后端支持 `asset://` 透传

新增判断函数：

```python
def is_ark_asset_url(url: str) -> bool:
    return str(url or "").strip().startswith("asset://")
```

处理规则：

- 图片输入如果是 `asset://...`，不要上传、不要转 base64，直接透传。
- 视频输入如果是 `asset://...`，直接作为视频参考素材传给上游。
- 音频输入如果是 `asset://...`，直接作为音频参考素材传给上游。

当前需要重点检查 `/api/canvas-video` 中类似逻辑：

```python
up_url = await upload_image_for_apimart(client, provider, ref.url)
```

如果 `ref.url` 是 `asset://...`，应改为直接使用：

```python
up_url = ref.url
```

### 2. 前端提供手动输入入口

MVP 可以在视频生成区域增加可选输入：

- 已授权真人图片 asset ID
- 已授权真人视频 asset ID
- 已授权真人音频 asset ID

前端提交前做标准化：

- 用户输入 `asset-xxx` → 转成 `asset://asset-xxx`
- 用户输入 `asset://asset-xxx` → 原样保留

### 3. 提示词辅助

当用户使用授权素材时，UI 应提示：

```text
请在提示词中用“图片1 / 视频1 / 音频1”引用素材，不要直接写 asset ID。
```

示例提示词：

```text
参考图片1中的人物形象，保持五官、气质和发型一致，生成一段近景人物视频。
```

## 后续增强方案

### 角色级绑定

后续可以在角色数据上绑定真人授权素材：

```json
{
  "authorized_person_assets": {
    "image_asset_id": "asset-xxx",
    "video_asset_id": "asset-yyy",
    "audio_asset_id": "asset-zzz"
  }
}
```

这样视频生成时可以自动把角色绑定的授权素材加入参考输入，不需要用户每次手动填写。

### 素材类型区分

后续可把 `AIReference` 扩展为更明确的结构，例如：

```python
class AIReference(BaseModel):
    url: str = ""
    name: str = ""
    role: str = ""
    source_type: str = "url"  # url | local | ark_asset
```

MVP 阶段可以先只通过 `asset://` 前缀判断。

## 限制与注意事项

1. Seedance 2.0 不支持直接上传含真人人脸的参考图/视频。
2. 已授权真人素材必须先完成真人认证、本人授权和素材入库。
3. 调用时使用的是素材 asset ID，不是单独 authorization 字段。
4. 提示词里不能直接写 asset ID，应使用“图片 n / 视频 n / 音频 n”引用。
5. 多模态参考里音频不能单独输入，至少要同时有图片或视频。
6. 图片参考、视频参考、音频参考数量需遵守 Seedance 2.0 限制：图片最多 9，视频最多 3，音频最多 3。

## 推荐实施顺序

1. 后端 `/api/canvas-video` 增加 `asset://` 透传支持。
2. 前端视频生成 UI 增加手动 asset ID 输入。
3. 增加提示词引用说明和基础校验。
4. 验证图片、视频、音频三类授权素材分别可用。
5. 再考虑角色级绑定和自动带入。
