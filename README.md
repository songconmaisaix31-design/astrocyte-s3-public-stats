<div align="center">

# astrocyte-s3-public-stats

A 27-line Go package that computes a mean without intermediate overflow, plus the tests that pin its input contract.

一个 27 行的 Go 包：算平均值但不让中间求和溢出，外加钉住输入契约的测试。

![Go](https://img.shields.io/badge/go-1.27.2-00ADD8)
![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)
![Network](https://img.shields.io/badge/network%20calls-none-brightgreen)
![License](https://img.shields.io/badge/license-not%20specified-lightgrey)

</div>

> **How to read this page / 怎么读这页**
> English first, then the same content in Chinese — not a summary, not a separate half.
> 每段英文在前、同一内容的中文紧跟在后；不是摘要，也没有单独的「中文版」。

---

## At a glance · 一眼看懂

| Field · 字段 | Value · 内容 |
|---|---|
| **Problem<br/>问题** | Summing large `float64` values overflows to `+Inf` before the division happens, so the mean of two perfectly representable numbers can come back as `+Inf` or `NaN`.<br/>先求和再相除时，两个很大的 `float64` 会在除法之前就溢出成 `+Inf`，于是一组本来完全可表示的数的平均值会变成 `+Inf` 或 `NaN`。 |
| **Fix<br/>做法** | `Mean` accumulates in `math/big.Rat` — exact rationals — and only converts to `float64` once, at the end.<br/>`Mean` 用 `math/big.Rat`（精确有理数）累加，只在最后一次性转成 `float64`。 |
| **Cost<br/>代价** | Zero dependencies, zero network calls, Go 1.27.2. One exported function and two exported errors.<br/>零依赖、零网络调用，Go 1.27.2。一个导出函数、两个导出错误。 |
| **Not<br/>不是** | Not a general statistics library. There is no `Median` yet, no variance, no streaming API.<br/>不是通用统计库。目前没有 `Median`，没有方差，没有流式接口。 |

## What it does · 它做什么

The whole public surface, from [stats.go](stats.go):

全部公开接口，来自 [stats.go](stats.go)：

| Symbol · 符号 | Signature · 签名 | Behaviour · 行为 |
|---|---|---|
| `Mean` | `Mean(values []float64) (float64, error)` | Exact rational accumulation, one final `float64` conversion. Does not modify the caller's slice.<br/>精确有理数累加，最后做一次 `float64` 转换。不修改调用方传入的切片。 |
| `ErrEmptyInput` | `error` | Returned for `nil` or a zero-length slice.<br/>输入为 `nil` 或长度 0 时返回。 |
| `ErrNonFiniteInput` | `error` | Returned for any `NaN`, `+Inf`, or `-Inf` element, checked before it is accumulated.<br/>任一元素为 `NaN`、`+Inf` 或 `-Inf` 时返回，检查发生在累加之前。 |

Both errors are package-level sentinels, so callers are expected to match with `errors.Is` rather than compare strings — which is exactly what [stats_test.go](stats_test.go) does.

两个错误都是包级哨兵值，调用方应当用 `errors.Is` 匹配而不是比较字符串——[stats_test.go](stats_test.go) 就是这么写的。

## Why exact rationals · 为什么用精确有理数

A naive implementation looks like this: add everything into a `float64`, then divide. With `[]float64{math.MaxFloat64, math.MaxFloat64}` the intermediate sum is already `+Inf`, and `+Inf / 2` stays `+Inf` — a wrong answer for an input whose true mean is perfectly representable.

朴素实现是：全部加进一个 `float64`，再相除。对 `[]float64{math.MaxFloat64, math.MaxFloat64}` 来说，中间和已经是 `+Inf`，`+Inf / 2` 仍是 `+Inf`——可这组输入的真实平均值是完全可表示的，所以这是个错答案。

`Mean` avoids that by never letting a `float64` hold a partial sum: each value becomes a `big.Rat`, the total is divided by the exact integer count, and `Float64()` is called once at the end. The case `{"large", []float64{math.MaxFloat64, math.MaxFloat64}, math.MaxFloat64}` in the test table is the one that would fail on the naive version.

`Mean` 的做法是不让任何 `float64` 持有部分和：每个值先变成 `big.Rat`，总和除以精确的整数个数，最后只调一次 `Float64()`。测试表里 `{"large", []float64{math.MaxFloat64, math.MaxFloat64}, math.MaxFloat64}` 这一条，朴素实现就会挂在这。

That exactness has a hard edge worth knowing: `big.Rat.Float64()` returns a rounded value plus an exactness flag, and the code keeps the value and discards the flag (`result, _ :=`). So the function guarantees no *intermediate* overflow, not a correctly rounded result for every input.

这种精确性有一个该知道的硬边：`big.Rat.Float64()` 返回的是「舍入后的值 + 是否精确」两个返回值，而代码只留了值、丢掉了标志（`result, _ :=`）。所以这个函数保证的是**中间过程**不溢出，而不是对任意输入都给出正确舍入的结果。

## Quickstart · 快速开始

### Requirements · 环境要求

Go 1.27.2 or newer, per [go.mod](go.mod). Nothing else — no third-party modules.

按 [go.mod](go.mod) 需要 Go 1.27.2 或更新版本。别的都不用——没有任何第三方模块。

### 1. Get the code · 拿到代码

```bash
git clone https://github.com/songconmaisaix31-design/astrocyte-s3-public-stats.git
cd astrocyte-s3-public-stats
```

The module line is `module swarmstats`, which is not a fetchable import path, so `go get` will not resolve it. Bring it in as a local `replace` instead.

`go.mod` 里写的是 `module swarmstats`，这不是一个可抓取的导入路径，所以 `go get` 解析不到它。改用本地 `replace` 引入：

```go
// in your own go.mod · 在你自己的 go.mod 里
require swarmstats v0.0.0
replace swarmstats => ../astrocyte-s3-public-stats
```

### 2. Validate · 验证

```bash
go test ./...      # run the shipped tests · 跑自带测试
go build ./...     # compile the package · 编译这个包
git diff --check   # no whitespace damage · 没有空白符破坏
```

These three commands are the ones the repository fixes in its own documentation. They were **not executed during the writing of this page** — this repository was reviewed read-only, without building. Run them yourself before trusting the numbers below.

这三条命令是仓库自己的文档里定死的验证方式。撰写本页时**没有执行**它们——这次是以只读方式审查、没有构建。在相信下面的数字之前，请自己跑一遍。

## Tests as shipped · 自带的测试

Two test functions in [stats_test.go](stats_test.go), stdlib `testing` only, no CI workflow wired to them.

[stats_test.go](stats_test.go) 里有两个测试函数，只用标准库 `testing`，没有 CI 工作流在跑它们。

| Test · 测试 | Cases · 用例 | What it pins · 钉住了什么 |
|---|---|---|
| `TestMeanFiniteInputs` | 7 table cases: `single`, `even`, `negative`, `duplicates`, `mixed`, `large`, `opposite-large`<br/>7 个表驱动用例 | Correct mean, plus a `slices.Clone` comparison asserting the input slice is unchanged.<br/>平均值正确，并用 `slices.Clone` 对比断言输入切片未被改动。 |
| `TestMeanRejectsEmptyAndNonFiniteInputs` | 1 empty case + 3 non-finite values (`NaN`, `+Inf`, `-Inf`)<br/>1 个空输入 + 3 个非有限值 | Exact sentinel errors via `errors.Is`.<br/>通过 `errors.Is` 断言精确的哨兵错误。 |

## The next feature, not implemented · 下一个功能（尚未实现）

The repository documents an approved `Median([]float64) (float64, error)` contract. There is **no `Median` function in [stats.go](stats.go)** at HEAD (`da3bddd`) — the contract below is a specification, not shipped code.

仓库记录了一份已批准的 `Median([]float64) (float64, error)` 契约。在 HEAD（`da3bddd`）的 [stats.go](stats.go) 里**没有 `Median` 函数**——下面这些是规格，不是已交付代码。

| Requirement · 要求 | Status · 状态 |
|---|---|
| Preserve input order<br/>保持输入顺序 | specified · 已写进规格 |
| Handle odd and even lengths<br/>处理奇偶两种长度 | specified · 已写进规格 |
| Accept negative and duplicate values<br/>接受负数与重复值 | specified · 已写进规格 |
| Reject empty input with `ErrEmptyInput`<br/>空输入返回 `ErrEmptyInput` | specified · 已写进规格 |
| Reject non-finite values with `ErrNonFiniteInput`<br/>非有限值返回 `ErrNonFiniteInput` | specified · 已写进规格 |
| No overflow for two equal large finite middle values<br/>两个相等的大有限中间值不溢出 | specified · 已写进规格 |
| Implementation and tests<br/>实现与测试 | **absent · 尚无** |

## Status and limits · 状态与边界

One commit (`da3bddd`, 2026-10-15), four files, one package, third-party dependencies: none. The repository describes itself as a controlled local Git repository holding public, agent-authored source with no private input.

一次提交（`da3bddd`，2026-10-15）、四个文件、一个包，第三方依赖为零。仓库自述是一个受控的本地 Git 仓库，只放公开的、agent 编写的源码，不含私有输入。

Known limits, stated rather than hidden:

边界写出来，而不是藏起来：

- `Median` is specified but not implemented — see the table above.<br/>`Median` 只有规格、没有实现——见上表。
- No CI. There is no workflow badge on this page because there is no `.github/workflows` directory to point at.<br/>没有 CI。本页没有工作流徽章，因为仓库里不存在 `.github/workflows` 目录可指。
- No `LICENSE` file, so no license is granted by default.<br/>没有 `LICENSE` 文件，因此默认不授予许可。
- `Mean` guarantees no intermediate overflow; it does not guarantee correct rounding for inputs where the true mean is not representable as a `float64`.<br/>`Mean` 保证的是中间过程不溢出；当真实平均值无法用 `float64` 表示时，它不保证正确舍入。
- The module path `swarmstats` is local-only. Plan on a `replace` directive, not a published import path.<br/>模块路径 `swarmstats` 只在本地有效。请按 `replace` 指令的用法接入，它不是已发布的导入路径。
- Only `Mean` is covered by tests. Nothing here has been benchmarked, so this page states no performance numbers.<br/>只有 `Mean` 有测试覆盖。这里没有做过任何基准测试，所以本页不写任何性能数字。

## License · 许可

No `LICENSE` file is present in this repository, so no license is granted by default and reuse needs permission from the author.

仓库里没有 `LICENSE` 文件，因此默认不授予任何许可，复用需要作者同意。
