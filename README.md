# RND_Chinese
Chinese localization of Robotics;Notes DaSH. Based on the patch by Committee of Zero. Muchas gracias. Contact me if there is anything inappropriate.

## sc3tools 版本说明 (IMPORTANT)

仓库里有两套 sc3tools,源码完全相同,唯一区别是编译时内嵌的 `resources/rnd/charset.utf8` 码表。**码表是 rust-embed 编译期打进 exe 的,改码表必须重新 `cargo build --release`,改完直接跑旧 exe 无效。**

- `sc3tools_rndchs/` — **中文版**。码表为汉化简体字码表 (13K, 4499 字, md5 `e62f5ca3...`),用于配合 `cnscript/` 译稿做中文回写 (replace-text)。
- `sc3tools_jp/` — **日文版**。码表为日文原版码表 (8.7K, 3020 字, md5 `6040d18f...`,即 `charset-.utf8` / `charset-备份.utf8`),用于提取日文原版脚本 (extract-text)。

**配对关系:日文包用日文版提取,中文回写用中文版。用错码表会得到大面积错码 (例如 `種子島` → `丽仁亿`,`海翔` → `ΥΦ`)。**

现成的 exe:
- `sc3tools_rndchs/target/release/sc3tools.exe` (934K, md5 `96cf7bb0...`) — 中文版
- `sc3tools_jp/target/release/sc3tools.exe` (921K, md5 `295b1c24...`) — 日文版

用法示例 (提取日文):
```
sc3tools_jp/target/release/sc3tools.exe extract-text "mes00.cpk/*.msb" rnd
```
输出会放在输入文件同级的 `txt/` 子目录。
