# 交接文档 — RND 简体中文补丁（字体/文本渲染方向）

> 最后更新：2026-09-09
> 本文件面向"接手继续开发"的人（包括未来的自己），目标是**不必重新推导已查明的结论**。

---

## 0. 一分钟速览

| 项 | 内容 |
|---|---|
| 项目 | ROBOTICS;NOTES DaSH 简体中文补丁 |
| 基础 | Committee of Zero (CoZ) 的英文优化补丁（LanguageBarrier 运行时） |
| 仓库 | `D:\DATA\tran\agent tran\GitHub\RND_Chinese` |
| 远程 | `https://github.com/Syun1524/RND_Chinese.git`（origin，无 upstream） |
| 分支 | `main` |
| 游戏目录 | `D:\Ruanjian\Steam\steamapps\common\ROBOTICS;NOTES DaSH` |
| 已修 | 4 项（3 代码级 + 1 数据级），均已在游戏内验证 |
| **待修** | **数字 `9` 不显示（见 §5，根因已查明，方案已定）** |
| 待评估 | 来电人名无滚动动效（见 §6，属 CoZ 功能缺失，非 bug） |

---

## 1. 目录与关键路径

```
仓库 D:\DATA\tran\agent tran\GitHub\RND_Chinese\
├── LanguageBarrier_rndchs\LanguageBarrier\      ← C++ 源码（改了 4 个文件）
│   ├── GameText.cpp / GameText.h                ← ruby 撞码修复
│   ├── TextRendering.cpp / TextRendering.h      ← 缓存指纹校验
│   └── dinput8-Release\dinput8.dll              ← 编译产物（发布必需）
├── sc3tools_rndchs\                             ← CN 字符集提取/回写工具
│   ├── resources\rnd\charset.utf8               ← CN 字符表（必须与运行时一致）
│   └── target\release\sc3tools.exe              ← 发布必需
├── sc3tools_jp\                                 ← JP 字符集工具（合作者 Kurashift 添加）
├── docs\                                        ← 本目录（修复记录 + 本文档）
└── .gitignore                                   ← 排除构建产物

游戏目录 D:\Ruanjian\Steam\steamapps\common\ROBOTICS;NOTES DaSH\
├── NOTES DaSH\dinput8.dll                       ← 实际生效的补丁 DLL（注意在子目录！）
├── languagebarrier\
│   ├── patchdef.json                            ← 运行时配置（字符表/字体/重定向）
│   ├── enscript\*.msb                           ← 译文（323 个）
│   ├── fonts\                                   ← 字体 + 烘焙缓存
│   ├── c0data\                                  ← 烤字图集（UI）
│   ├── subs\                                    ← 影片字幕 ASS
│   └── log.txt                                  ← 运行日志
└── languagebarrier\enscript_backup_swap_*\      ← 字符集搬运前的 msb 全量备份
```

工作区（非仓库）：
```
D:\DATA\tran\agent tran\9.6文本外工作\
├── 临时\                                        ← 排查脚本、备份（可清理）
│   └── swap_charset_slot.py                     ← 字符位交换工具（见 §7）
├── RND补丁汉化隔壁成品\                          ← 隔壁方案（CHDTevior/RNDash_CN）
└── RND补丁汉化隔壁的源代码\                      ← 隔壁的脚本与术语表（有参考价值）
```

---

## 2. 构建环境与命令

| 项 | 值 |
|---|---|
| 编译器 | MSBuild（VS2022 安装目录 `C:\VS2022BT`） |
| 工具集 | **v142（VS2019）**，需 14.29 版本 |
| **配置名** | **`dinput8-Release`**（注意：不是 `Release`！） |
| 平台 | `Win32`（x86） |

**编译 DLL**：
```bash
cd "D:/DATA/tran/agent tran/GitHub/RND_Chinese/LanguageBarrier_rndchs/LanguageBarrier"
MSYS_NO_PATHCONV=1 "C:/VS2022BT/MSBuild/Current/Bin/MSBuild.exe" \
  LanguageBarrier.vcxproj -p:Configuration=dinput8-Release -p:Platform=Win32 \
  -v:minimal -nologo
# 产物: dinput8-Release/dinput8.dll
# 部署: 复制到 游戏目录/NOTES DaSH/dinput8.dll
```
> Git Bash 下必须加 `MSYS_NO_PATHCONV=1`，否则 `/p:` 会被路径转换。

**编译 sc3tools（改了字符集后必须重编！）**：
```bash
cd "D:/DATA/tran/agent tran/GitHub/RND_Chinese/sc3tools_rndchs"
cargo build --release --offline
```
> ⚠️ 该工具用 `rust-embed` **编译期嵌入**字符集。只改 `resources/rnd/charset.utf8`
> 而不重编 = 工具仍是旧字符表 → 与运行时不一致 → 乱码。

---

## 3. 已完成的工作（已验证）

详细内容见 `docs/CHANGES-字体与文本渲染修复.md` 与 `docs/CHANGES-运行时数据与配置.md`。
此处只列摘要：

### 3.1 代码级

| # | 问题 | 根因 | 状态 |
|---|---|---|---|
| 1 | `A`/`8`/`9` 静默消失（如 `PHASE NAE`→`PHSE NE`） | `0x80 0x09/0x0A/0x0B` 既是 ruby 控制码又是字符集索引 9/10/11（`8`/`9`/`A`），被无条件当控制码吞掉 | 部分修复（见 §5） |
| 2 | 改字符集后旧字体缓存不失效 | `loadCache()` 只校验语言不校验字符集 | ✅ 已修（加 `charsetHash` 指纹） |
| 3 | 缓存撕裂（修复 2 的回归） | DDS 缺失时残留索引；saveCache 未删图集 | ✅ 已修 |

修复 1 的具体做法：新增 `safeRubyMarkers`（默认开），三处判断（`GameText.cpp:1422 / 2594 / 2773`）加条件
`(!SAFE_RUBY_MARKERS || sc3string[1] == 10 || insideRubyText)`。
效果：`0x0B`('A') 与 `0x09`('8') 在非 ruby 上下文正常渲染；**`0x0A`('9') 仍被吞**（见 §5）。

### 3.2 数据级

| 项 | 内容 |
|---|---|
| 字体更换 | `noto.ttc`(JP) → **`NotoSansCJKsc-Regular.otf`**（简中黑体，SIL OFL 可分发） |
| 高位字形搬运 | 索引 ≥4096 的 336 个字搬入低位空槽（见 §4），重映射 7389 处 msb 引用 |
| 缓存清理 | 多次（换字体/改字符集后必须清，见 §8） |
| sc3tools 重编 | 字符集变更后重编，并用官方工具验证提取正确 |

### 3.3 仓库整理

- 新增 `.gitignore`（排除构建产物，保留发布必需的 dll/exe）
- 移除误提交的构建产物共 **932 个**（CN 466 + JP 466）
- 提交：`e34114b`（修复）、`018739f`、`3719b06`（清理），均已推送

**安全点（可回档）**：
```
分支: backup-before-cleanup / backup-before-jp-cleanup
tag : cleanup-safe-20260909_025921 / cleanup-jp-20260909_030957
```
回档示例：`git reset --hard 727cfd7`（推送前）或 `git reset --hard backup-before-cleanup`

---

## 4. 关键机制：4096 分界线（重要！）

**游戏原生字体系统只支持 4096 个字形槽位。**

- 原版 CoZ RND 字符集 = **4096** 项
- 本补丁扩展后 = **4499** 项（多 403 个）
- 索引 ≥4096 的字形：
  - **对话**（LB 接管绘制）→ 正常显示
  - **口袋电脑/存档/TIPS 等**（游戏原生绘制）→ **静默消失**（零宽度空精灵）

这就是「汐」在对话正常、在口袋电脑消失的原因。

**已做的修复**：把"索引 ≥4096 且被 msb 使用"的 336 个字搬入**低位空槽**。
- 工具：`临时/swap_charset_slot.py --pairs a:b,c:d --apply`
- 必须同步更新三处：`patchdef.json` 的 charset、`sc3tools_rndchs/.../charset.utf8`、所有 msb 引用
- 搬运后必须清字体缓存

**⚠️ 遗留缺陷**：该工具用"**交换**"而非"单向移动"，会把低位原有的字符换到高位。
已确认目前**无害**（被换上去的字符如 `*` `+` `$` `"` 在 msb 中实际使用 0 次），
但若将来新增文本用到这些符号，它们会消失。**建议改进为单向移动**。

**当前状态**：msb 中使用的 3986 个字符，**全部在 4096 以下**（已校验）。

---

## 5. 【待修】数字 `9` 不显示

### 现象
TIPS 等界面里所有数字 `9` 消失（如 `2010年9月11日` → `2010年 月11日`）。

### 根因
`9` 位于字符集**索引 10**，编码 `80 0A`，与 **ruby（注音）开始标记 `0x0A` 完全同码**。
字节层面无法区分，二者必选其一。

### 实测数据
```
0x0A 总出现次数        : 437
  其中真 ruby(9→10→11) :  30   （仅 9 个字符串）
  其余(应为数字 9)     : 407
引用 '9'(idx 10) 的次数: 445，涉及 7 个文件
索引 20..4095 可用空槽 : 265 个
```

### 推荐方案（按优先级）

**方案 A：把 `9` 搬出索引 10 + 上下文感知重写**
1. 选一个**未被 msb 使用**的低位空槽 X（有 265 个可选，如索引 3044）
2. 交换：索引 10 (`9`) ↔ 索引 X
3. 重写 msb 时**只改"孤立的 `80 0A`"**（即数字 9），
   **保留真 ruby 的 `80 0A`**（其前必然有 `80 09`，配对出现）
4. 重编 sc3tools + 清字体缓存

> 关键点：现有 `swap_charset_slot.py` 会无差别重写全部 `80 0A`，
> **会破坏 30 处真 ruby**。需先增强该工具（按上下文区分），或手工处理这 9 个字符串。

**方案 B：彻底禁用 ruby**
若确认中文补丁不需要振假名（真 ruby 仅 9 个字符串，且多为字符集测试串如 `zz_00.msb`）
→ 把 `SAFE_RUBY_MARKERS` 逻辑改为"9/10/11 全部当普通字形"。
代价：那 9 处 ruby 注音会显示为字面字符。

**方案 C：不动**
接受 `9` 不显示。**不推荐**（数字 9 在年份/日期/数量中很常见）。

### 建议
先人工确认那 9 个含真 ruby 的字符串在中文版里的实际显示需求，
若确实无用 → 直接**方案 B**（最简单）；否则 → **方案 A**。

---

## 6. 【待评估】来电人名无滚动动效

### 现象
口袋电脑主屏左上角来电人名，原版是滚动文字，汉化后"像图片粘在那"。

### 已查明
- **LB 源码中没有任何滚动/逐字动画实现**（已全量搜索）
- `sghdDrawInteractiveMailHook`（`GameText.cpp:2834`）**一次性画完所有字符**：
  ```cpp
  for (int i = 0; i < str.length; i++) gameExeDrawGlyph(...);
  ```
- 该 hook 替换了游戏原生绘制函数（`GameText.cpp:986`），但**未复刻动画**
- hook 签名含 `lineSkipCount` / `lineDisplayCount`，疑为原版滚动参数，但 LB 把它们当作
  "行数范围"使用而非动画偏移

### 结论
**这是 CoZ 补丁的功能缺失**（优先保证文字显示，动画未移植），非 bug，也不是本次改动造成的。

### 若要恢复
需要逆向定位"当前滚动位置"变量（可能在 ScrWork 或全局），再在 hook 内按时间计算偏移。
工作量大、有风险，建议**优先级低于 §5**。

---

## 7. 工具用法

### sc3tools（官方提取/回写）
```bash
# 提取（CN 文本用 rndchs 工具）
sc3tools_rndchs/target/release/sc3tools.exe extract-text <file.msb> rnd
#   → 输出到 txt/<file>.msb.txt

# 回写
sc3tools_rndchs/target/release/sc3tools.exe replace-text <file.txt> rnd
```

> ⚠️ **CN 与 JP 工具不可混用**：
> - `sc3tools_rndchs` = CN 字符集（4499 项）→ 处理**译文**
> - `sc3tools_jp` = JP 字符集（3020 项）→ 处理**日文原文提取**
> 混用会导致字符表不匹配 → 乱码。

### 字符位交换工具（自研，慎用）
```bash
python 临时/swap_charset_slot.py --pairs 4202:92            # 空跑预览
python 临时/swap_charset_slot.py --pairs 4202:92 --apply    # 落盘
```
自动备份 patchdef / charset.utf8 / 全部 msb 到 `enscript_backup_swap_<时间戳>/`。

**已知限制**：无差别重写，无法区分 `0x0A` 是"数字9"还是"ruby标记"（见 §5 方案 A）。

### msb 格式（自研脚本解析用）
```
0x00: b'MES\0'
0x04: u32, 0x08: u32          （未用）
0x0C: u32  idx_end            （索引区结束位置）
0x10: 索引区起点，每项 8 字节 (u32 id, u32 offset)
idx_end + offset              （字符串数据地址）
```
字形流：
- 字形 = 2 字节 `(0x80|(id>>8)), (id&0xFF)`，**低字节可以 < 0x80**（易踩坑）
- `0x03` = 1 字节控制码（cnscript 显示为 `[%p]`）
- `0xFF` = 字符串结束
- 其余 `<0x80` 为控制码（长度依命令而定，`0x04` 为 2 字节）

> **踩坑记录**：若把低字节 `<0x80` 误判为控制码并只跳 1 字节，会导致整串错位，
> 产生大量**虚假的"越界字形 id"**。排查期间因此多次误判，务必用 2 字节对齐解析。

---

## 8. 字体缓存规则（必须遵守）

1. **换字体 / 改字符集后必须清缓存**：
   ```
   删除 languagebarrier/fonts/ 下：
     font_*.dds  outline_*.dds  fontData.bin
   ```
   （`charsetHash` 只覆盖字符集，**不含字体路径**，换字体不会自动失效）

2. **多进界面才能烤全字号**：LB 只在游戏实际用到某字号时才烘焙。
   已观测字号：16 18 24 25 27 28 30 31 32 34 36 37 39 40 42 43 45 46 48 56 61 63
   （56/61 来自 `BacklogTextSize[...] * 1.5f` 等动态计算，属正常）

3. **必须正常退出游戏**：`saveCache()` 在 `closeAllSystemsHook` 里调用；
   强杀进程会导致本次烘焙丢失（曾出现 `fontData.bin` 缺失）。

4. 首次启动较慢属正常（重新烘焙）。

---

## 9. 发布前检查清单

- [ ] **字体**：打包 `NotoSansCJKsc-Regular.otf`（SIL OFL，可分发）
      —— **不要**打包 `simhei.ttf`（Windows 系统字体）或 `noto.ttc`（日文且非必需）
- [ ] **不要预置字体缓存**（`font_*.dds` / `fontData.bin`），让用户本地生成，
      可从机制上避免"生成时字符集 ≠ 运行时字符集"
- [ ] **保留 CoZ 许可**：`LICENSE`、`THIRDPARTY.LB.txt`
- [ ] **标注游戏版本**：DLL 内含 `SigScan` 特征码，硬编码游戏 exe 版本；
      Steam 更新后可能失效
- [ ] **标注分支来源**：基于 CoZ LanguageBarrier，注明本分支修改内容
- [ ] 确认 §5 的 `9` 问题是否已修（否则 TIPS/日期中的 9 全缺）
- [ ] 实机回归：对话、口袋电脑、存档读档、TIPS、邮件、地图各走一遍

---

## 10. 已知未修 / 待观察

| # | 问题 | 状态 |
|---|---|---|
| 1 | 数字 `9` 不显示 | **根因已查明，方案已定，待实施**（§5） |
| 2 | 来电人名无滚动动效 | CoZ 功能缺失，非 bug（§6） |
| 3 | 30 个 PUA 图标字（U+E001–E01E）不显示 | 字体无此字形，被过滤。隔壁方案用 1044 个 PUA 槽参考 `Build-RNDZhNativeMapFont.ps1` |
| 4 | 偶发 Runtime Error 崩溃 | 清理字体/缓存后**暂未复现**，未确证。已排除：大图集超限、烘焙卡顿、码表错配、签名失败 |
| 5 | 离线扫描仍报约 980 处"越界字形" | 大概率是扫描器对控制码长度处理不准的假象；其中 344 处集中在废弃文件 `rnd_01_01_00-.msb`（patchdef/gamedef 中零引用） |

---

## 11. 排查经验（避免重复踩坑）

1. **不要用视觉判断字符集问题** —— 用脚本解码 msb 看实际字节。多次因"看起来像"而误判。
2. **解析 msb 必须 2 字节严格对齐**，低字节可 `<0x80`。否则产生大量虚假越界。
3. **区分"数据问题"与"渲染问题"**：若同一字符在对话正常、某界面消失，
   先怀疑**渲染路径差异**（LB 接管 vs 游戏原生），而非字体数据。
4. **离线渲染验证**：用游戏真实 `fontData.bin` + DDS 图集离线拼出字符串图像，
   可确认数据链路是否健康，快速缩小范围。
5. **改字符集是"换门牌号"**：msb 存的是索引不是字符，所以必须同步重写 msb 引用。
