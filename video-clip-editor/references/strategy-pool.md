---
AIGC:
    ContentProducer: Minimax Agent AI
    ContentPropagator: Minimax Agent AI
    Label: AIGC
    ProduceID: "00000000000000000000000000000000"
    PropagateID: "00000000000000000000000000000000"
    ReservedCode1: 3046022100a5cc00bfcf7eb8a6b9e492e95db2f86f676e22529f1aa71a9ebc40f479fed431022100b0ae49201d9dc708c7003f3ee2729232c37e7ac447023e6e1262116c73a4b332
    ReservedCode2: 3046022100f97a4f2d8a7a7ea6f919b675d887f67e51dc0f861c2e0f6b689bf38b1f6f1900022100ca55190bc9a3df935212611c82ada3cf3447a59a11a72a4ee855018470eab04a
---

# 策略池详解参考

## 长片段去重策略池

### 策略触发条件

**必须满足**：单段素材时长 > 5 秒

**时长计算**：片段总区间结束时间 - 总区间开始时间

### 策略池选项

#### 1. 原向保留（策略代码：keep）

**适用场景**：
- 单段时长 ≤ 5 秒（不触发策略池）
- 相邻长片段已使用其他策略，需要平衡
- 片段本身构图已经很独特

**代码实现**：
```bash
# 不添加额外滤镜，仅补齐分辨率
[v_raw]scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v_out]
```

**判断逻辑**：
```
if (单段时长 <= 5秒) → 使用"原向保留"
else → 从其他三种策略中选择
```

#### 2. 水平镜像反转（策略代码：hflip）

**适用场景**：
- 片段左侧留白较多
- 需要打破原有视线方向
- 作为备选策略而非首选

**代码实现**：
```bash
# 在 setpts 前添加 hflip
[v_raw]hflip,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v_out]
```

**注意事项**：
- 不宜全片滥用，以免形成新的模板痕迹
- 建议与裁切放大、平移策略交替使用

#### 3. 轻微裁切放大（策略代码：crop）

**适用场景**：
- 片段边缘有无效信息（如logo、时间戳）
- 需要局部放大突出细节
- 推荐放大比例 103%-106%

**推荐裁切参数**：

| 放大比例 | crop 参数示例 |
|---------|--------------|
| 103% | `crop=trunc(iw*0.97/2)*2:trunc(ih*0.97/2)*2:trunc((iw-trunc(iw*0.97/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.97/2)*2)/2/2)*2` |
| 104% | `crop=trunc(iw*0.96/2)*2:trunc(ih*0.96/2)*2:trunc((iw-trunc(iw*0.96/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.96/2)*2)/2/2)*2` |
| 105% | `crop=trunc(iw*0.95/2)*2:trunc(ih*0.95/2)*2:trunc((iw-trunc(iw*0.95/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.95/2)*2)/2/2)*2` |
| 106% | `crop=trunc(iw*0.94/2)*2:trunc(ih*0.94/2)*2:trunc((iw-trunc(iw*0.94/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.94/2)*2)/2/2)*2` |

**完整代码示例**：
```bash
[v_raw]crop=trunc(iw*0.97/2)*2:trunc(ih*0.97/2)*2:trunc((iw-trunc(iw*0.97/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.97/2)*2)/2/2)*2,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v_out]
```

#### 4. 轻微构图平移（策略代码：pan）

**适用场景**：
- 主体偏左或偏右，需要平衡画面
- 水平方向有调整空间
- 垂直方向需要微调

**代码实现**：
```bash
# 水平偏移（向右偏移画面 3%）
[v_raw]crop=trunc(iw*0.97/2)*2:trunc(ih*0.97/2)*2:trunc((iw-trunc(iw*0.97/2)*2)*0.35/2)*2:trunc((ih-trunc(ih*0.97/2)*2)/2/2)*2,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v_out]

# 垂直偏移（向下偏移画面 2%）
[v_raw]crop=trunc(iw*0.98/2)*2:trunc(ih*0.98/2)*2:trunc((iw-trunc(iw*0.98/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.98/2)*2)*0.6/2)*2,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v_out]
```

### 策略选择决策矩阵

| 单段时长 | 前一片策略 | 推荐策略 | 说明 |
|---------|-----------|---------|------|
| ≤5秒 | 任意 | 原向保留 | 不触发策略池 |
| >5秒 | 原向保留 | hflip/crop/pan | 三选一 |
| >5秒 | hflip | crop/pan/原向 | 避免连续同策略 |
| >5秒 | crop | hflip/pan/原向 | 避免连续同策略 |
| >5秒 | pan | hflip/crop/原向 | 避免连续同策略 |

---

## 整片倍速方案池

### 方案选项

| 倍速值 | 适用场景 | 听感影响 | 完播率影响 |
|--------|---------|---------|-----------|
| **1.21** | 追求更自然听感 | 轻微加速，几乎无失真 | 中等提升 |
| **1.23** | 平衡完播率与听感（推荐） | 自然加速，听感舒适 | 较好提升 |
| **1.25** | 追求更高完播率 | 明显加速，轻微变调 | 最佳提升 |

### 应用规则

1. **全片统一**：必须选择一个倍速值统一作用于整条视频与音频
2. **禁止混用**：不得在同一条视频内使用多档倍速
3. **音视频同步**：视频 `setpts=PTS/x` 与音频 `atempo=x` 必须使用相同的 x 值

### 代码实现

```bash
# 视频流应用倍速
[v_concat]setpts=PTS/1.23[v_speed]

# 音频流应用倍速（必须与视频一致）
[a_concat]atempo=1.23[a_speed]
```

### 选速决策参考

| 原始素材时长 | 推荐倍速 | 预期成片时长 |
|-------------|---------|-------------|
| 约 65 秒 | 1.21 | 约 54 秒 |
| 约 65 秒 | 1.23 | 约 53 秒 |
| 约 65 秒 | 1.25 | 约 52 秒 |
| 约 100 秒 | 1.21 | 约 83 秒 |
| 约 100 秒 | 1.23 | 约 81 秒 |
| 约 100 秒 | 1.25 | 约 80 秒 |

---

## 抽帧方案池

### 方案选项

| 间隔 | select 表达式 | 说明 |
|------|--------------|------|
| **每 2.8 秒抽 1 帧** | `select='if(gte(t\,2.8)*lt(mod(t\,2.8)\,0.04)\,0\,1)'` | 较高去同质化强度 |
| **每 3.1 秒抽 1 帧** | `select='if(gte(t\,3.1)*lt(mod(t\,3.1)\,0.04)\,0\,1)'` | 平衡去同质化 |
| **每 3.4 秒抽 1 帧** | `select='if(gte(t\,3.4)*lt(mod(t\,3.4)\,0.04)\,0\,1)'` | 较低去同质化强度 |

### 应用规则

1. **仅作用于视频流**：抽帧只处理最终视频总流，不得作用于音频流
2. **VFR 输出**：必须添加 `-fps_mode vfr` 参数
3. **时机**：在整片倍速完成后执行
4. **默认每次抽掉 1 帧**

### 代码实现

```bash
# 在倍速处理后的视频流上应用抽帧
[v_speeded]select='if(gte(t\,3.1)*lt(mod(t\,3.1)\,0.04)\,0\,1)'[v_final]
```

### 完整音频处理链

```bash
# 音频流：倍速 + 响度标准化
[a_speeded]atempo=1.23,loudnorm=I=-16:TP=-1.5:LRA=11[a_final]
```

### 参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| `I` | -16 | 综合响度目标（LUFS） |
| `TP` | -1.5 | 真实峰值上限（dBTP） |
| `LRA` | 11 | 响度范围（LU） |

---

## 策略组合示例

### 完整处理流程

```bash
ffmpeg -i input.mp4 -filter_complex "
# 第一步：各片段处理
[0:v]trim=start=0.000:end=6.000,crop=...,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v1]; [0:a]atrim=start=0.000:end=6.000,asetpts=PTS-STARTPTS[a1];
[0:v]trim=start=8.200:end=15.800,hflip,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v2]; [0:a]atrim=start=8.200:end=15.800,asetpts=PTS-STARTPTS[a2];
[0:v]trim=start=18.500:end=23.000,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v3]; [0:a]atrim=start=18.500:end=23.000,asetpts=PTS-STARTPTS[a3];

# 第二步：拼接
[v1][a1][v2][a2][v3][a3]concat=n=3:v=1:a=1[v_con][a_con];

# 第三步：整片倍速
[v_con]setpts=PTS/1.23[v_speed];
[a_con]atempo=1.23[a_speed];

# 第四步：抽帧去同质化
[v_speed]select='if(gte(t\,3.1)*lt(mod(t\,3.1)\,0.04)\,0\,1)'[v_out];

# 第五步：音频响度标准化
[a_speed]loudnorm=I=-16:TP=-1.5:LRA=11[a_out]
" -map "[v_out]" -map "[a_out]" -c:v libx264 -crf 18 -preset fast -c:a aac -b:a 192k -movflags +faststart -fps_mode vfr output-final.mp4
```
