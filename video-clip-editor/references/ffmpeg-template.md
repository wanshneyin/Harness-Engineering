---
AIGC:
    ContentProducer: Minimax Agent AI
    ContentPropagator: Minimax Agent AI
    Label: AIGC
    ProduceID: "00000000000000000000000000000000"
    PropagateID: "00000000000000000000000000000000"
    ReservedCode1: 304502203b66d326f83fd4e7f979514a54a6ebda0810bf6782ec9dfc6ad9c3fd2e5d0208022100c88316ae229bd936c64a43b3b132bf7c2de64ee64e117c1c214bc290c23645d9
    ReservedCode2: 3044022026b693fa8a1d59771a6c962f3d99be0ec5313ddb2f7861d95a04a21da360b82b02204f4fe230a3803afd298aa5ec08c17e25a16f0ec39d75dfdac1864ee2c1e2953f
---

# FFmpeg 命令模板参考

## 文件名规则

- 输入视频文件名：与当前会话 SRT 文件同名基础名（如 `xxxxx.mp4`）
- 输出视频文件名：基础名加 `-final.mp4`（如 `xxxxx-final.mp4`）

## 完整命令模板

```bash
# 模版说明：
# - 1080:1920 为示例分辨率，实际应使用源视频分辨率
# - 1.23 为示例整片倍速，实际应从方案池选择（1.21/1.23/1.25）
# - 3.1 为示例抽帧间隔，实际应从方案池选择（2.8/3.1/3.4）
# - 每段都补齐 setsar=1 再送入 concat

ffmpeg -i xxxxx.mp4 -filter_complex \
"[0:v]trim=start=0.000:end=6.000,crop=trunc(iw*0.97/2)*2:trunc(ih*0.97/2)*2:trunc((iw-trunc(iw*0.97/2)*2)/2/2)*2:trunc((ih-trunc(ih*0.97/2)*2)/2/2)*2,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v1]; [0:a]atrim=start=0.000:end=6.000,asetpts=PTS-STARTPTS[a1]; \
 [0:v]trim=start=8.200:end=15.800,hflip,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v2]; [0:a]atrim=start=8.200:end=15.800,asetpts=PTS-STARTPTS[a2]; \
 [0:v]trim=start=18.500:end=23.000,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v3]; [0:a]atrim=start=18.500:end=23.000,asetpts=PTS-STARTPTS[a3]; \
 [0:v]trim=start=30.500:end=37.000,crop=trunc(iw*0.96/2)*2:trunc(ih*0.96/2)*2:trunc((iw-trunc(iw*0.96/2)*2)*0.35/2)*2:trunc((ih-trunc(ih*0.96/2)*2)/2/2)*2,scale=1080:1920,setsar=1,setpts=PTS-STARTPTS[v4]; [0:a]atrim=start=30.500:end=37.000,asetpts=PTS-STARTPTS[a4]; \
 [v1][a1][v2][a2][v3][a3][v4][a4]concat=n=4:v=1:a=1[v_con][a_con]; \
 [v_con]setpts=PTS/1.23,select='if(gte(t\,3.1)*lt(mod(t\,3.1)\,0.04)\,0\,1)'[v_out]; \
 [a_con]atempo=1.23,loudnorm=I=-16:TP=-1.5:LRA=11[a_out]" \
-map "[v_out]" -map "[a_out]" -c:v libx264 -crf 18 -preset fast -c:a aac -b:a 192k -movflags +faststart -fps_mode vfr xxxxx-final.mp4
```

## 长片段去重策略池

| 策略 | 适用场景 | 代码写法 |
|------|---------|---------|
| **原向保留** | 单段 ≤5 秒，或相邻段已用其他策略 | 不添加额外滤镜 |
| **hflip** | 需要镜像反转效果 | 在 setpts 前添加 `,hflip` |
| **轻微裁切放大** | 推荐 103%-106% | `crop=trunc(iw*0.97/2)*2:trunc(ih*0.97/2)*2:...` |
| **轻微构图平移** | 水平或垂直偏移 | 添加 crop 偏移参数 |

## 整片倍速方案池

| 倍速值 | 适用场景 |
|--------|---------|
| **1.21** | 追求更自然听感 |
| **1.23** | 平衡完播率与听感（推荐） |
| **1.25** | 追求更高完播率 |

## 抽帧方案池

| 间隔 | select 表达式 |
|------|--------------|
| **每 2.8 秒抽 1 帧** | `select='if(gte(t\,2.8)*lt(mod(t\,2.8)\,0.04)\,0\,1)'` |
| **每 3.1 秒抽 1 帧** | `select='if(gte(t\,3.1)*lt(mod(t\,3.1)\,0.04)\,0\,1)'` |
| **每 3.4 秒抽 1 帧** | `select='if(gte(t\,3.4)*lt(mod(t\,3.4)\,0.04)\,0\,1)'` |

## 关键注意事项

1. **VFR 处理**：执行抽帧时必须添加 `-fps_mode vfr`，避免编码阶段自动补帧
2. **偶数尺寸**：涉及 crop/scale 时必须使用 `trunc(.../2)*2` 确保宽高为偶数
3. **统一分辨率**：所有片段送入 concat 前必须统一为相同分辨率和 SAR
4. **音频同步**：atempo 值必须与视频 setpts 值严格一致
5. **音量标准化**：音频链末尾必须添加 `loudnorm=I=-16:TP=-1.5:LRA=11`
