COSMIX · 余弦概率像素散射
COSMIX 是一种可逆图像混淆算法：以确定性伪随机过程重排像素位置并打散像素数值，
使输出图像外观呈噪点，同时全部原始信息无损保留，可用同一算法逐位还原。
整体管线：
```
输入 RGBA ──► 余弦概率 Feistel 置换（位置重排）──► 逐像素值扩散（数值打散）──► PNG
噪点 PNG  ──► 反值扩散 ──► 逆 Feistel 置换（位置还原）──► 指纹自检 ──► 原始 RGBA
```
---
1. 密钥派生（KDF）
主密钥由 `BLAKE2b(固定种子 + "|" + 变体名)` 派生（`deriveMaster`）。
子密钥通过 `HMAC-SHA256` 按标签派生：`perm/phase`、`perm/round/NN`、`diff/rotate`、`diff/xor`。
像素指纹 `sum = BLAKE2b(像素)` 前 16 字节，用于还原后校验像素是否被改动。
2. 余弦概率 Feistel 置换
以像素总数 `n` 决定分组半长（`permutationParams` 取 2 的幂）。
每轮置换表 `makeRoundTable` 结合余弦权重：
`cosineWeight(t, phase) = 0.5·(1 + cos(2π(t + phase)))`，
将 BLAKE2b 输出的局部相位与全局相位混合后调制置换倾向，使打散在统计上更均匀。
采用 cycle-walking 保证落在 `[0, n)` 内且严格双射，因此可精确逆向（`feistelDecrypt`）。
轮数默认 8，范围 4–32。
3. 逐像素值扩散
由 `lowbias32` 计数器密钥流（`keystream`）生成两类序列：通道旋转量 `rot`
（在 R/G/B 之间循环移位）与异或掩码 `ks`。
混淆：`mixed[t] = V[P[t]]` 先置换，再对每像素做通道旋转 + 异或；还原为对称逆运算。
处理按块进行（每块 `2^16` 像素）。
4. PNG 编解码
解码 `decodePNG`：支持 8-bit 非隔行的灰度/RGB/调色板/带 Alpha 等颜色类型，
完整还原 5 种行过滤器（None/Sub/Up/Average/Paeth）。
编码 `encodePNG`：统一输出 RGBA（color type 6），原图辅助块原样搬回，
头信息以 `tEXt` 文本块写入。
zlib 压缩/解压借助 `CompressionStream` / `DecompressionStream`。
5. 头信息
混淆时写入 `cosmix` 文本块，字段包括：`magic`、`v`（版本）、`w/h`（尺寸）、
`variant`（变体）、`rounds`（轮数）、`alpha`（是否参与通道旋转）、`sum`（像素指纹）、
`src_format`、`meta`。还原时据此确定变体、轮数与版本，并校验像素指纹。
