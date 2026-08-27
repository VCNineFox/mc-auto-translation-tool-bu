## 修复 6 个 Bug：JSON 解析、并发限流、多语言检测

### 变更内容

修复了 `translator-core` 模块中的 6 个问题，涉及 JSON 解析、并发速率限制、翻译服务冗余逻辑和多语言字符检测。

---

### Bug 1：`JsonStrings.readStringField` 搜索到字段名作为值时过早返回 null（高）

**文件**：`net/JsonStrings.java`

**问题**：`readStringField` 在 JSON 文本中搜索字段名时，如果先匹配到该名称作为字符串值（而非键），方法直接返回 `null`，不再继续搜索后面的实际字段。

```json
// 这种 JSON 会失败：第一个 "content" 是值不是键
{"note": "content", "content": "真实译文"}
```

**修复**：匹配到非字段位置时，更新 `searchFrom` 继续搜索，而非返回 null。

```diff
  int colon = skipWhitespaceTo(json, field + needle.length(), ':');
  if (colon < 0) {
-     return null;
+     searchFrom = field + needle.length();
+     continue;
  }
```

---

### Bug 2：OpenAI 响应解析用全文搜索代替路径导航（高）

**文件**：`provider/OpenAiChatTranslationProvider.java`

**问题**：OpenAI 兼容 API 返回 `{"choices":[{"message":{"content":"译文"}}]}`，译文嵌套在 `choices[0].message.content`。代码用 `readStringField(response, "content")` 全文搜索第一个 `"content"`，如果响应中其他位置也有 `content` 字段就会取到错误值。

**修复**：优先用 `readStringPath` 按路径精确定位，失败再回退。

```diff
- String translated = JsonStrings.readStringField(response, "content");
+ String translated = JsonStrings.readStringPath(response, "choices[0].message.content");
+ if (translated == null) {
+     translated = JsonStrings.readStringField(response, "content");
+ }
```

**影响范围**：DeepSeek、DashScope、火山方舟、智谱、本地 llama.cpp 等所有 OpenAI 兼容服务。

---

### Bug 3：速率限制在 synchronized 块内 sleep，阻塞所有线程（中）

**文件**：`provider/ResilientTranslationProvider.java`

**问题**：`awaitRateLimit()` 是 `synchronized` 方法，但内部调用了 `Thread.sleep(delay)`。一个线程睡眠时，所有其他翻译线程都被锁在外面无法进入。

**修复**：synchronized 只保护共享状态读写，sleep 移到外面。

```diff
- private synchronized void awaitRateLimit() throws InterruptedException {
-     long now = System.currentTimeMillis();
-     long delay = nextRequestAtMillis - now;
-     if (delay > 0L) {
-         Thread.sleep(delay);
-         now = System.currentTimeMillis();
-     }
-     nextRequestAtMillis = now + minimumIntervalMillis;
- }
+ private void awaitRateLimit() throws InterruptedException {
+     long delay;
+     synchronized (this) {
+         long now = System.currentTimeMillis();
+         delay = nextRequestAtMillis - now;
+         if (delay < 0L) {
+             delay = 0L;
+         }
+         nextRequestAtMillis = now + delay + minimumIntervalMillis;
+     }
+     if (delay > 0L) {
+         Thread.sleep(delay);
+     }
+ }
```

---

### Bug 4：腾讯混元双重速率限制，实际间隔翻倍（中）

**文件**：`provider/TencentHunyuanProvider.java`

**问题**：`TencentHunyuanProvider` 内部有速率限制（`MIN_REQUEST_INTERVAL_MILLIS = 60L`），但创建时会被 `ResilientTranslationProvider` 包装，后者也有 60ms 限速。实际每次请求等待 120ms，是预期值的 2 倍。

**修复**：移除 `TencentHunyuanProvider` 内部的速率限制逻辑（常量、字段、方法），统一由 `ResilientTranslationProvider` 处理。

---

### Bug 5：未检测日语假名和韩语谚文，浪费 API 调用（中）

**文件**：`LanguageHeuristics.java`

**问题**：`shouldTranslate` 只检测中文字符。日语文本目标语言是日语时、韩语文本目标语言是韩语时，文本仍被发送到翻译 API。

**修复**：增加日语假名（Hiragana、Katakana）和韩语谚文（Hangul）检测。

```diff
+ if (isJapaneseTarget(targetLanguage) && (han + japaneseKana) == letters) {
+     return false;
+ }
+ if (isKoreanTarget(targetLanguage) && koreanHangul == letters) {
+     return false;
+ }
```

---

### Bug 6：下载器 User-Agent 版本号不一致（低）

**文件**：`offline/VerifiedDownloader.java`

**问题**：`VerifiedDownloader` 的 User-Agent 是 `MCAutoTranslationTool/1.1`，`HttpJsonClient` 是 `MCAutoTranslationTool/1.3.8`。

**修复**：统一更新为 `1.3.8`。

---

### 测试

`CoreSelfTest.handlesJsonStrings()` 新增测试用例，验证字段名作为值出现时 `readStringField` 能正确跳过并找到实际字段。

### 文件清单

| 文件 | Bug |
|------|-----|
| `net/JsonStrings.java` | #1 |
| `provider/OpenAiChatTranslationProvider.java` | #2 |
| `provider/ResilientTranslationProvider.java` | #3 |
| `provider/TencentHunyuanProvider.java` | #4 |
| `LanguageHeuristics.java` | #5 |
| `offline/VerifiedDownloader.java` | #6 |
| `test/.../CoreSelfTest.java` | #1 测试 |
