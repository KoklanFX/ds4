# Customization Log — v41-m3ultra-Kevin

> 本分支在官方 `antirez/ds4` v41-m3ultra 基础上，针对本机 M3 Ultra + DeepSeek-V4.1-Flash 生产负载的三项定制优化。
> 对应提交：`40ac491`（2026-10-05）。本文件与代码同版本维护。

---

## 总览

| 优化 | 文件 / 函数 | 目的 | 状态 |
|:--|:--|:--|:--|
| 防死循环（repetition penalty） | `ds4.c` + `ds4_server.c` | 打破退化短序列震荡，消除无效 token 浪费 | ✅ 默认启用 |
| 语气词治理（tone mask） | `ds4.c` + `ds4_server.c` | 抑制 think 段句首「好」，输出更干净 | ✅ 默认启用 |
| 推理预算调整（reasoning budget） | `ds4_server.c` | 按 effort 精细控制思考长度，省时 / 更深度 | ✅ 默认启用 |

三者均为**采样层 / 生成循环层**的本地增强，不改模型权重、不改 GGUF，可编译回退。

---

## 一、防死循环（repetition penalty）—— 核心

### 现象
模型偶尔卡死在 4–5 token 短序列反复震荡（如 `（好，输出。）好。**好，输出。**（好，停止循环。）`），曾连续 9 个请求、每请求循环 240–260 次。引擎原版**完全没有** repetition / frequency / no-repeat-ngram 机制，模型陷入退化后无外力拉回。

### 实现位置
- **`ds4.h`**（L498–509）：对外接口声明
  - `ds4_repeat_set(hist, n, window, penalty)` — 注入历史与窗口
  - `ds4_repeat_active()` — 是否启用
  - `ds4_repeat_apply_freq(n_vocab)` — 重建频率计数
  - `ds4_repeat_count(id)` — 查询某 token 出现次数
- **`ds4.c`**（L1055–1106）：
  - `DS4_REPEAT_MAX_HIST = 4096`（ring buffer 容量）
  - `DS4_REPEAT_FREQ_CAP = 131072`（频率数组大小 = 词汇量级）
  - `ds4_repeat_set`：初始化全局历史窗口与惩罚强度（当前固定 `penalty=0.0`，即只做**硬排除**而非软惩罚）
  - `ds4_repeat_apply_freq`：把窗口内 token 出现次数写入频率表
  - `ds4_repeat_count`：返回窗口内重复次数（≥2 即判定为重复候选）
- **`ds4_server.c`**（生成主循环）：
  - L13905–13914：每请求初始化 ring buffer（`DS4_SERVER_REPEAT_HIST = 4096`）
  - L13946–13958：从 buffer 取**最近最多 256 个 token** 构成当前窗口 → `ds4_repeat_set` → `ds4_repeat_apply_freq`
  - L14062 / L14252：每生成一个 token，将其 id 写入 ring buffer
  - L14080 附近：采样前把窗口内重复 ≥2 次的 token 的目标 logit 置为 `-inf`（见下方「采样拦截」）

### 采样拦截（关键机制）
在 `sample_build_probabilities` / `sample_argmax` / `sample_full_vocab` 三条采样路径的统一入口（`ds4.c` 的采样器内核），对 `ds4_repeat_count(id) >= 2` 的 token 直接排除（logit → `-inf`），**归一化发生在排除之后**，故候选重建不会把被屏蔽 id 加回。投机解码（DSpark/MTP 的 verify 路径）复用同一采样器，同样被拦截。

### 效果
- 死循环无法自喂（重复 token 被立即剔除），模型被迫换词前进
- 实测 3 个本会触发循环的复杂任务全部正常完成、0 循环 0 重复
- 正常请求无回归；单测 `test_sampling.c` 覆盖边界

### 开关
无独立开关，固定启用（写死在 `ds4_server.c` 生成循环内）。恢复原版采样 = 用官方源码重编。

---

## 二、语气词治理（tone mask）

### 现象
推理（think）段高频出现句首「好」（如「好的，」「好，我们」），污染输出观感。

### 实现位置
- **`ds4.h`**（L498–500）：`ds4_tone_mask_set / active / covers` 接口
- **`ds4.c`**：采样入口应用，把 masked id 的 logit 置 `-inf`（同 repeat 机制的做法）
- **`ds4_server.c`**：
  - L12197–12198：`tone_mask_piece_begins_sentence()` 检测「下一个 token 是否为句首」
  - L13899–13900：`tone_mask_ids[1] = { 853 }`（**仅压 DeepSeek V4.1 tokenizer 的 token 853 = 「好」**）
  - L13939：`ds4_tone_mask_set(at_sentence_start && thinking.inside ? tone_mask_ids : NULL, ...)`
  - L13954 起：采样前对 logits 就地应用 mask

### 触发条件（双重限定）
1. **处于 think 段内**（`thinking.inside`）
2. **下一个 token 是句首**（前一 token 为标点 / 段首）

两个条件同时满足才压「好」。正文正常用法（如引用「好的，」）不受影响。

### 局限
- 只压了「好」这个**整词 token**（id 853）。「嗯」无整词 token（拆成 3 个字节片段散落在 18411 / 925 / 4042 等 id），单 token 压不掉，只能靠 SFT 根治。
- 换模型时需要重新查目标 tokenizer 的「好」token id。

---

## 三、推理预算调整（reasoning budget）

### 现象
原版引擎对思考长度无硬性预算，长任务无限思考或短任务思考过度。

### 实现位置
`ds4_server.c` 生成循环 L13960 起 `reasoning_budget_*` 逻辑：当处于 think 段且已思考 token 数 ≥ 请求的 `reasoning_budget` 时，强制结束思考段（`forced_end_think`），进入回答阶段。

### 效果
- 配合 Hermes 的 `reasoning_effort` 参数，低 effort 秒级出答案、高 effort 允许深度思考
- 避免单请求无界占用，提升引擎吞吐稳定性

### 机制说明
`reasoning_budget` 由请求方传入；未传则回退到引擎默认行为。不设置时与前版行为一致。

---

## 四、测试与回归

- `tests/test_sampling.c` 新增 56 行，覆盖 tone-mask 与 repetition 的边界：
  - 未激活时不清除任何 id
  - 激活时只屏蔽目标 / 重复 id，其它不受影响
  - 越界 / 非法 id 安全返回
- 编译方式（照官方）：`make clean && make && make test`
- 当前生产二进制 md5：`81701b253b40acd718cdbd8ab2579240`（含上述三项 + 后续 effort 微调）

---

## 五、回滚 / 备份

| 镜像 | 备份文件 |
|:--|:--|
| 好-tone 上线前 | `ds4-server.bak-pre-tone-deploy-1005-1808`、`ds4.c.bak-tone-1005-1740`、`ds4_server.c.bak-tone-1005-1740` |
| 死循环 repeat 上线 | `ds4-server.bak-repeat-1005-1926` |
| effort 相关 | `ds4-server.bak-effort-high-1005`、`ds4-server.bak-effortprefix-1004`、`ds4-server.bak-prerbudget-1004`、`ds4-server.bak-medium16000-1005` |
| 当前 | `ds4-server`（生产） |

回滚 = 停止引擎 → 用备份二进制覆盖 `ds4-server`（保留权限）→ 重启。

---

## 六、历史关联

- 日志：`~/wiki/日志/2026-10-05.md`（当天完整复盘）
- 仓库 `main`：官方 V4.1 主线；本分支在其上叠加定制
- 上游同步：`git fetch upstream && git merge upstream/main`（在本分支做，冲突只解一次）

*本文档由小平维护，随 `v41-m3ultra-Kevin` 分支更新。*