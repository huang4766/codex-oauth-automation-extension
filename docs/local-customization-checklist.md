# 本地定制核对清单

这份清单用于维护当前 fork 的本地定制点。

适用场景：

- 从上游仓库拉取新更新后，想快速确认本地关键修复有没有被覆盖
- 合并 `master` / `dev` / 其他上游分支后，想快速判断重点回归风险
- 打包发版前，想确认关键逻辑仍然存在

建议用法：

1. 每次合并上游后，先看这份清单
2. 优先检查文中列出的关键文件和关键关键词
3. 至少跑文末列出的最小测试集合

## 1. 最需要保住的本地定制点

当前 fork 最关键、最容易在上游同步时被冲掉的逻辑有 4 类：

1. `email_in_use -> 回步骤 7 重新获取邮箱`
2. `步骤 3 / 4 / 8` 的已进入后续页面兜底，不再误判失败
3. `2925 + Cloudflare 转发` 场景下的邮箱匹配与提码
4. `2925` 中文 OpenAI 临时验证码正文提取

如果只做最小检查，优先围绕这 4 类核对。

## 2. 注册 / 登录主链路重点文件

### 2.1 `background/message-router.js`

重点关注：

- `STEP_COMPLETE`
- `STEP_ERROR`
- `logged_in_home`
- `skipProfileStep`
- 自动运行中过期步骤消息的忽略逻辑

这是很多“页面其实走到了后面，但后台状态还停在前面”的收口点。

### 2.2 `background/signup-flow-helpers.js`

重点关注：

- 步骤 3 收尾确认
- ChatGPT 首页 / 资料页 fallback
- signup entry 稳定等待逻辑

这是步骤 2 / 3 / 4 状态衔接的关键文件。

### 2.3 `background/steps/fetch-signup-code.js`

重点关注：

- 步骤 4 的验证码页准备与回滚逻辑
- `skipProfileStep`
- 手机验证后切回邮箱验证的处理
- 2925 的 `filterAfterTimestamp`

### 2.4 `background/steps/fetch-login-code.js`

重点关注：

- `STEP8_EMAIL_IN_USE`
- `email_in_use` 后回步骤 7 的逻辑
- `step8VerificationTargetEmail`
- 步骤 8 的邮箱轮询参数

### 2.5 `background/verification-flow.js`

重点关注：

- `mail2925MatchTargetEmail`
- 邮箱轮询 payload
- 步骤 4 / 8 提交后的 fallback
- `invalidCode`
- `oauth_consent_page`
- `add_phone_page`

### 2.6 `content/signup-page.js`

重点关注：

- `logged_in_home`
- `email_in_use`
- `skipProfileStep`
- 步骤 5 / 8 页面状态识别

这是很多页面状态最终判定的来源。

## 3. 2925 专项重点文件

### 3.1 `content/mail-2925.js`

这是当前 fork 最值得重点防守的本地定制文件。

重点关注这些关键词是否还在：

- `shouldAllowCloudflarePreviewTargetFallback`
- `cfbounces`
- `代发`
- 中文临时验证码提取
- 排除域名数字误识别
- 轮询增强日志

当前本地预期行为：

1. 普通 `2925 receive` 模式下，如果列表预览目标邮箱与当前轮不一致，直接跳过
2. 只有在 `Cloudflare` 转发痕迹明确存在时：
   - 预览包含 `cfbounces` 或 `代发`
   - 且预览仍能看出当前目标邮箱所在域名
   - 才允许“预览不一致但继续打开正文确认”
3. 中文正文如 `输入此临时验证码以继续` 时，应优先提取正文真实验证码
4. 不能把 `hl4766902.asia` 这类域名中的数字误提成验证码

### 3.2 `background/mail-2925-session.js`

重点关注：

- 2925 登录态恢复
- 账号池切换
- 当前邮箱页与目标邮箱一致性检查

## 4. 关键词快速核对法

每次合并上游后，建议先搜下面这些关键词：

- `email_in_use`
- `logged_in_home`
- `skipProfileStep`
- `mail2925MatchTargetEmail`
- `cfbounces`
- `代发`

如果这些关键词相关逻辑整体消失，说明本地定制很可能被覆盖了。

## 5. 推荐 diff 方式

每次 pull / merge 后，优先看下面这些文件的 diff：

```bash
git diff <上次稳定点>..HEAD -- \
  background/message-router.js \
  background/signup-flow-helpers.js \
  background/steps/fetch-signup-code.js \
  background/steps/fetch-login-code.js \
  background/verification-flow.js \
  content/signup-page.js \
  content/mail-2925.js
```

如果只想快速看 2925：

```bash
git diff <上次稳定点>..HEAD -- \
  content/mail-2925.js \
  background/mail-2925-session.js \
  background/verification-flow.js
```

## 6. 最小测试集合

每次上游同步后，建议至少跑这几组：

```bash
node --test tests/mail-2925-content.test.js
node --test tests/background-step7-recovery.test.js
node --test tests/background-step4-filter-window.test.js
node --test tests/verification-flow-polling.test.js
```

含义：

- `mail-2925-content`：2925 提码与 Cloudflare 转发边界
- `background-step7-recovery`：`email_in_use` 回步骤 7
- `background-step4-filter-window`：步骤 4 过滤窗口与 `skipProfileStep`
- `verification-flow-polling`：2925 轮询 payload、步骤 4 / 8 轮询分支

## 7. 当前维护建议

当前 fork 后续继续同步上游时，优先注意：

1. `content/mail-2925.js` 是高风险冲突文件，尽量逐段看 diff，不要只看是否 merge 成功
2. 如果上游改了步骤 4 / 8 的邮箱轮询入口，必须复核 `2925` 和 `email_in_use` 两条链路
3. 如果上游继续增强 `operation delay`，要确认 mail polling 相关 bundle 仍然没有被错误包裹

## 8. 一句话记忆版

以后每次拉上游，只要先确认这 4 点还在，基本就不会把最关键的本地定制丢掉：

1. `email_in_use -> 回步骤 7`
2. `logged_in_home / skipProfileStep` 兜底
3. `2925 + Cloudflare cfbounces` 预览与正文边界
4. `2925` 中文临时验证码提取
