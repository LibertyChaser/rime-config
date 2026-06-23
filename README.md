# Rime / Squirrel 输入法配置

本人的 macOS Squirrel / Rime 配置仓库。

基础方案使用 [雾凇拼音 rime-ice](https://github.com/iDvel/rime-ice)，本仓库只保存我自己的个性化覆盖配置。

[toc]

## 最重要的原则

```text
~/Library/Rime/
= 当前电脑真正使用的 Rime 配置目录

GitHub
= 管理 YAML / Lua / 自定义短语等配置文件

iCloud Drive/RimeSync
= Rime 用户词库和输入习惯的同步交换站

*.userdb
= 当前电脑正在使用的本地用户词库，不直接用 Git 管
```

不要把整个 `~/Library/Rime` 直接交给 iCloud 同步。

稳定做法是：**GitHub 管配置，iCloud + Rime Sync 管输入习惯。**

## 常用命令

打开 Rime 配置目录：

```bash
open ~/Library/Rime
```

打开 iCloud RimeSync：

```bash
open ~/Library/Mobile\ Documents/com~apple~CloudDocs/RimeSync
```

查看当前安装信息：

```bash
cat ~/Library/Rime/installation.yaml
```

同步用户词库：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --sync ~/Library/Rime
```

重新部署：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```

清理 build 后重新部署：

```bash
rm -rf ~/Library/Rime/build
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```

重启 Squirrel（之后仍然需要手动点击 Deploy 在右上角的输入法）：

```bash
killall Squirrel || true
open -a Squirrel
```

## 目录分工

### GitHub 管理的内容

本仓库管理的是 Rime 的配置文件，主要放在：

```bash
~/Library/Rime/
```

包括但不限于：

```text
default.custom.yaml
squirrel.custom.yaml
rime_ice.custom.yaml
custom_phrase.txt
double_pinyin_flypy.custom.yaml
lua/
opencc/
others/
```

这些文件控制输入方案、候选词数量、候选框样式、配色、快捷键、模糊音、短语、符号等配置。

### iCloud 管理的内容

iCloud 只负责同步 Rime 的**用户词库**和**输入习惯**交换数据，不直接管理主配置目录。

当前 iCloud 同步目录是：

```bash
~/Library/Mobile Documents/com~apple~CloudDocs/RimeSync
```

在 `~/Library/Rime/installation.yaml` 中通过 `sync_dir` 指定：

```yaml
sync_dir: "/Users/par/Library/Mobile Documents/com~apple~CloudDocs/RimeSync"
```

iCloud 里的 `RimeSync` 通常会包含多个设备 ID 目录，例如：

```text
RimeSync/
├── 15f4928a-6977-4686-bbd9-c618cd725ca0
└── EF175FAF-2A55-4534-9D8C-F04887199BAD
```

这些目录分别代表不同电脑 / 不同 Rime 安装实例的同步身份。

### 同步机制说明

Rime 的同步不是 iCloud 自动合并配置，而是：

```text
iCloud 负责搬运同步目录里的文件
Rime 负责读取 sync_dir，并在本机 userdb 和 RimeSync 之间合并用户词库
```

需要区分三个东西：

```text
*.userdb/
= 当前电脑正在使用的本地用户词库数据库
= Rime 打字时真正读取和更新的地方

RimeSync/
= 多台电脑之间交换用户词库数据的中转站
= 不直接参与日常输入，只在执行 Rime 同步时被读取/写入

iCloud
= 只负责把 RimeSync 文件夹同步到其他电脑
= 不理解 Rime 数据，也不会自动合并用户词库
```

也就是说，平时打字时：

```text
用户输入
    ↓
Rime 学习新词、词频、输入习惯
    ↓
写入本机 ~/Library/Rime/*.userdb/
```

执行同步时：

```text
本机 *.userdb/
    ↓
Rime 导出本机用户词库数据
    ↓
写入 iCloud/RimeSync/本机 installation_id/
```

同时，Rime 也会读取其他设备的同步数据：

```text
iCloud/RimeSync/其他设备 installation_id/
    ↓
Rime 读取其他设备的用户词库数据
    ↓
合并进本机 ~/Library/Rime/*.userdb/
```

所以完整流程可以理解为：

```text
A 电脑输入习惯
    ↓
A 的本地 *.userdb/ 被更新
    ↓
A 执行 Rime 同步
    ↓
A 的用户词库数据写入 iCloud/RimeSync/A设备ID/
    ↓
iCloud 把 RimeSync 同步到 B 电脑
    ↓
B 执行 Rime 同步
    ↓
B 读取 RimeSync/A设备ID/ 的数据
    ↓
B 的本地 *.userdb/ 合并 A 的输入习惯
    ↓
B 同时把自己的 *.userdb/ 导出到 RimeSync/B设备ID/
```

反过来也一样：

```text
B 电脑输入习惯
    ↓
B 执行 Rime 同步
    ↓
写入 iCloud/RimeSync/B设备ID/
    ↓
A 执行 Rime 同步
    ↓
A 的本地 *.userdb/ 合并 B 的输入习惯
```

因此，`RimeSync` 不是最终生效的词库。
 最终真正生效的是本机的：

```text
~/Library/Rime/*.userdb/
```

`RimeSync` 的作用是作为跨设备交换站。
 每次执行：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --sync ~/Library/Rime
```

Rime 都会同时做两件事：

```text
1. 把 RimeSync 里其他设备的数据合并进本机 *.userdb/
2. 把本机 *.userdb/ 的数据更新到 RimeSync 里的本机设备目录
```

所以新设备要继承旧设备的输入习惯，需要满足三个条件：

```text
1. 新设备的 installation.yaml 设置了正确的 sync_dir
2. iCloud 已经把旧设备的 RimeSync 数据同步过来
3. 新设备执行过 rime_deployer --sync
```

同步命令：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --sync ~/Library/Rime
```

同步后建议重新部署一次：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```

一句话总结：

```text
*.userdb 是本机真正使用的用户词库；
RimeSync 是跨设备交换用户词库的中转站；
iCloud 只负责同步 RimeSync 文件夹；
Rime Sync 才负责真正合并用户词库。
```

## 当前个性化修改

### 输入方案

当前基于雾凇拼音 `rime-ice`，并启用小鹤双拼相关方案：

```text
double_pinyin_flypy.schema.yaml
rime_ice.schema.yaml
```

主要目标：

```text
中文输入：雾凇拼音
双拼方案：小鹤双拼
英文混输：保留 melt_eng 相关支持
```

### 候选词数量

候选词数量通过 `default.custom.yaml` 或相关 patch 配置控制。
 用于调整每页显示的候选词数量，避免候选框过长或候选不足。

常见配置位置，如候选词数量：

```yaml
patch:
  menu/page_size: 5
```

### Squirrel 外观和配色

候选框样式、字体、颜色、横排/竖排等控制由：

```text
squirrel.custom.yaml
```

这里用于管理：

```text
候选框颜色
高亮候选词颜色
字体大小
候选框圆角
边框
阴影
候选词排列方式
```

常见配置类似：

```yaml
patch:
  style/color_scheme: custom
  style/horizontal: true
  style/font_point: 16
  style/candidate_format: "%c %@ "
```

具体配色方案放在：

```yaml
preset_color_schemes:
  custom:
    name: Custom
    author: Songqing
    back_color: 0xFFFFFF
    text_color: 0x000000
    hilited_candidate_back_color: 0xD75A00
    hilited_candidate_text_color: 0xFFFFFF
```

注意：Squirrel 的颜色格式通常是 `0xBBGGRR`，不是普通网页里的 `#RRGGBB`。

### 用户词库和输入习惯

本地真正使用的用户词库在：

```text
~/Library/Rime/*.userdb/
```

例如：

```text
rime_ice.userdb/
luna_pinyin.userdb/
```

这些目录是当前电脑正在使用的输入习惯数据库。
 不要手动用 Git 管理这些 `.userdb` 目录，也不要直接把整个 `~/Library/Rime` 放到 iCloud 里同步。

正确做法是：

```text
GitHub 同步配置
iCloud + Rime Sync 同步用户词库和输入习惯
```

### 词库、拼写容错与自定义短语的分工

| 位置 | 适合放什么 | 编码方式 | 例子 |
| --- | --- | --- | --- |
| `rime_ice.dict.yaml` | 正确、高频、主动想加入词库的内容；适合 emoji、专有名词、高频短语、希望稳定出现在候选里的词 | 标准拼音编码 | `点儿  dian er 1` |
| `rime_ice.custom.yaml` 的 `speller/algebra` | 拼写容错规则；负责把规律性错误输入映射到正确拼音 | 正则派生规则 | `xai -> xia` |
| `custom_phrase.txt` | 临时、个人、非标准输入码补丁；适合特别私人的、强制置顶的、或无法靠标准拼音和 `speller/algebra` 稳定解决的内容 | 自定义输入码 | `点儿    dainer  1` |

简单原则：

- 正确内容进词库 `rime_ice.dict.yaml`；
- 规律性手误进 `rime_ice.custom.yaml` 的 `speller/algebra`；
- 特殊非标准短语进 `custom_phrase.txt`。

### 不再使用的本地 sync 目录

如果 `installation.yaml` 里已经设置了：

```yaml
sync_dir: "/Users/par/Library/Mobile Documents/com~apple~CloudDocs/RimeSync"
```

那么：

```text
~/Library/Rime/sync/
```

这个本地 sync 目录就不是当前主要同步目录，可以删除。

## 日常使用

### 快速移动正在输入的拼音光标

在候选框还显示时，拼音还没有真正输入到文本框里。  

这个阶段左右方向键默认是逐字符移动。

如果正在输入较长拼音串，例如：

```text
hai wang cang
```

可以使用：Option + Left / Option + Right 按词或拼音片段快速移动光标。

或者使用 Tab
  - Tab = 从前往后切换拼音片段
  - Shift + Tab = 从后往前切换拼音片段

### 修改配置后

```bash
cd ~/Library/Rime
git add .
git commit -m "Update rime config"
git push

"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```

### 同步输入习惯

在每台电脑上分别执行：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --sync ~/Library/Rime
```

如果刚从另一台电脑同步过来，建议再部署一次：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```

## 新电脑初始化流程

### 1. 安装 Squirrel

```bash
brew install --cask squirrel-app
```

### 2. 安装雾凇拼音 rime-ice

```bash
rm -rf /tmp/rime-ice
git clone --depth=1 https://github.com/iDvel/rime-ice.git /tmp/rime-ice
rsync -av --exclude='.git' /tmp/rime-ice/ ~/Library/Rime/
```

### 3. 覆盖我的自定义配置

```bash
rm -rf /tmp/rime-config
git clone git@github.com:LibertyChaser/rime-config.git /tmp/rime-config
rsync -av /tmp/rime-config/ ~/Library/Rime/
```

### 4. 设置 iCloud RimeSync 目录

先确认 iCloud 目录存在：

```bash
mkdir -p ~/Library/Mobile\ Documents/com~apple~CloudDocs/RimeSync
```

然后检查：

```bash
cat ~/Library/Rime/installation.yaml
```

确保里面有类似：

```yaml
sync_dir: "/Users/par/Library/Mobile Documents/com~apple~CloudDocs/RimeSync"
```

### 5. 同步用户词库

等 iCloud 把 `RimeSync` 文件夹同步完成后，执行：

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --sync ~/Library/Rime
```

这一步会把其他电脑的用户词库和输入习惯合并到当前电脑。

### 6. 重新部署

```bash
"/Library/Input Methods/Squirrel.app/Contents/MacOS/rime_deployer" --build ~/Library/Rime
```
