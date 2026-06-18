# jitter水印去除

去除 Lottie 动画 JSON 文件中 jitter.video 免费版的水印（包括水印文字和水印底色）。

## 使用方式

用户提供一个 Lottie 动画 JSON 文件路径，自动去除 jitter.video 水印并保存。

## 参数

- `$ARGUMENTS` - 输入文件路径和输出文件路径（空格分隔），如果只提供输入路径则在同目录下生成去水印版本

## 执行流程

使用 Python 脚本执行以下步骤：

1. **读取 JSON 文件** - 加载 Lottie 动画数据
2. **定位水印图层** - jitter.video 水印的特征：
   - 顶层 Layer 0 通常是一个 precomp（ty=0），引用水印资源
   - 水印 asset 包含：一个小尺寸矩形（约 764x212）+ 矢量文字路径（填充白色 [1,1,1,1]）
   - 水印 asset 通常是嵌套的：顶层 asset → 中间 asset → 最终水印 asset
3. **删除水印图层** - 从顶层 layers 中移除水印 precomp 图层及其配对的 null 图层（ty=3）
4. **删除水印资源** - 从 assets 中移除水印相关的 asset（通过 refId 追踪链）
5. **清理 meta 字段** - 移除 `"meta":{"g":"https://jitter.video"}` 引用
6. **验证** - 确认文件中不再包含 "jitter" 字符串
7. **保存文件** - 输出为紧凑 JSON 格式

## Python 脚本模板

```python
import json, os, sys

def remove_jitter_watermark(input_path, output_path):
    with open(input_path, 'r') as f:
        data = json.load(f)

    # Step 1: Find and trace the watermark asset chain
    # The watermark is typically the first precomp layer (ty=0) at the top level
    watermark_asset_ids = set()
    removed_layer = False

    if data['layers'] and data['layers'][0].get('ty') == 0:
        ref_id = data['layers'][0].get('refId', '')
        if ref_id:
            watermark_asset_ids.add(ref_id)
            # Trace the chain: find the referenced asset and check its sub-references
            for asset in data['assets']:
                if asset.get('id') == ref_id:
                    for layer in asset.get('layers', []):
                        sub_ref = layer.get('refId', '')
                        if sub_ref:
                            watermark_asset_ids.add(sub_ref)
                    break

            # Verify it's actually a watermark by checking the final asset
            is_watermark = False
            for asset in data['assets']:
                if asset.get('id') in watermark_asset_ids:
                    for layer in asset.get('layers', []):
                        shapes = layer.get('shapes', [])
                        for s in shapes:
                            if s.get('ty') == 'rc':
                                size = s.get('s', {}).get('k', [])
                                # Watermark rect is typically small (not full-screen)
                                if isinstance(size, list) and len(size) == 2:
                                    if size[0] < 1920 and size[1] < 1080 and size[0] > 200:
                                        is_watermark = True

            if is_watermark:
                # Remove the watermark layer
                data['layers'].pop(0)
                removed_layer = True
                # Remove paired null layer if present
                if data['layers'] and data['layers'][0].get('ty') == 3:
                    data['layers'].pop(0)

    # Step 2: Remove watermark assets
    if watermark_asset_ids:
        data['assets'] = [a for a in data['assets'] if a.get('id') not in watermark_asset_ids]

    # Step 3: Clean meta field
    if 'meta' in data:
        meta = data.get('meta', {})
        if isinstance(meta, dict) and 'jitter' in json.dumps(meta).lower():
            data['meta'] = {}

    # Step 4: Verify
    content = json.dumps(data)
    has_jitter = 'jitter' in content.lower()

    # Step 5: Save
    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, separators=(',', ':'))

    return {
        'removed_layer': removed_layer,
        'removed_assets': len(watermark_asset_ids),
        'still_has_jitter': has_jitter,
        'output_path': output_path
    }

# Parse arguments
args = "$ARGUMENTS".strip().split()
if len(args) >= 2:
    input_path = args[0]
    output_path = args[1]
elif len(args) == 1:
    input_path = args[0]
    base, ext = os.path.splitext(input_path)
    output_path = base + '_no_watermark' + ext
else:
    print("Usage: /jitter水印去除 <input_file> [output_file]")
    sys.exit(1)

result = remove_jitter_watermark(input_path, output_path)
print(f"Done! Saved to: {result['output_path']}")
print(f"  Removed watermark layer: {result['removed_layer']}")
print(f"  Removed assets: {result['removed_assets']}")
if result['still_has_jitter']:
    print("  WARNING: 'jitter' reference still found in file!")
```

## 注意事项

- 仅删除水印相关图层，不修改动画内容、背景、其他预合成
- 水印通常位于图层栈的最顶部（index 0）
- 如果文件结构不同（水印不在顶层），需要手动定位
- 输出文件使用紧凑 JSON 格式以减小体积
