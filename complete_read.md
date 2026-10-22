# 细粒度测评：Linux完整使用与DeepSeek交接文档

更新日期：2026-10-22。协议版本：`fg-eval/1.1`。仓库：[hitmhj/fine-grained-eval](https://github.com/hitmhj/fine-grained-eval)，当前为私有。

本文可以直接交给公司Linux环境中的DeepSeek。所有命令面向Linux Bash，在仓库根目录执行，不依赖原聊天记录，也不需要远程控制公司电脑。

**当前状态：`gen_truth.py`、`evaluate.py`、`gen_html.py`及严格协议已经实现；`get_html.py`是兼容入口。`change.py`尚未实现，须在现场对接原始Qwen结果。平台SDK、账号、实际模型名称及版本须由现场提供。已通过15项模拟SDK测试，尚未完成真实平台API联调，没有新的真实模型能力成绩。**

## 1. 给现场DeepSeek的任务

以下文字可直接作为交接提示词：

```text
请接手本仓库在当前公司Linux主机上的部署、小样本联调及结果适配。
先阅读complete_read.md、PROTOCOL.md、contracts.py、common.py及三个入口，
检查实际文件和环境，不假设缺失文件已经存在。

目标：评价Qwen3.5-35B-A3B的原始图片回答，分别测摘要、AI解读、
菜品组合和当地常用语；强模型生成参考，评审模型提供逐项标签，
程序计算分数并生成简洁HTML。

依次完成：
1. 检查仓库版本、Python、公司SDK、图片、原始results.json、
   实际prompt YAML和运行参数。优先复用已能调用平台SDK的Python环境。
2. 用平台已经验证的初始化代码接入SDK；不猜包名、安装源、模型别名或认证流程。
3. 根据真实YAML及实际追加内容填写task_config.json。
4. 实现本文第8节的change.py。适配只改变文件结构，不修改模型回答，
   不猜造提示词、图片身份、token数量或结束原因。
5. 先通过不调用真实API的检查，再以1～2张图完成参考生成、结果适配、
   实际评分和报告。核对逐项记录后才扩大数据量。
6. 保留命令、版本、错误、状态、原始数据和产物路径。

不得改变fg-eval/1.1的字段、公式或缺失记录计分规则来消除报错。
不得将mock、prepare-only或生成HTML成功称为真实API/模型能力验证成功。
不得上传凭据、公司图片、真实YAML、原始回答或含图片的JSONL/HTML。
缺少材料时列出缺什么、为何需要，并完成不依赖它们的工作，不伪造材料。
如没有终端/文件权限，给出命令或补丁，等待操作者反馈，不声称已经执行。
最后按“已执行、已通过、未通过/待补、产物路径”交接。
```

## 2. 整体流程与现有文件

```text
图片 + 实际任务prompt + 可选当地上下文/可靠资料
                        │
                        ▼
                  gen_truth.py
          独立参考生成、摘要候选、机器复核
                        │
                        ▼
                   truth.jsonl ─────────────────────────┐
                                                        │
既有Qwen推理 → results.json → 待实现change.py             │
                                      │                 │
                                      ▼                 ▼
                             predictions.jsonl → evaluate.py
                                                        │
                                                        ▼
                                                 evaluations.jsonl
                                                        │
                                                        ▼
                                                   gen_html.py
                                                        │
                                                        ▼
                                                   report.html
```

| 程序/文件 | 当前作用 |
|---|---|
| `gen_truth.py` | 输入图片文件夹，生成独立机器参考`truth.jsonl`；可导出无答案输入清单 |
| `evaluate.py` | 输入truth和严格格式的predictions，调用评审API，再由程序计算分数 |
| `gen_html.py` | 从评分JSONL生成离线HTML与可选汇总JSON |
| `get_html.py` | 与`gen_html.py`同参数的兼容入口 |
| `change.py` | **尚未编写**；需将原推理脚本的结果转换成严格JSONL |
| `common.py` | JSONL、原子保存、哈希、SDK调用与重试 |
| `contracts.py` / `schemas/` | 结构、版本及严格校验 |
| `metrics.py` / `prompts.py` | 四项公式及参考/评审提示词 |
| `api_config.example.json` | 不含凭据的接口配置模板 |
| `task_config.example.json` | 四任务prompt结构示例，不代表真实历史输入 |
| `tests/` | 合成SDK测试，不是真实视觉能力测试 |

本仓库不包含Qwen权重、原推理程序、公司SDK、真实账号或测试图片。**评分工具不加载本地模型，不要求GPU、CUDA、H800或PyTorch。** 若需要重新生成Qwen答案，继续使用原推理项目的环境和加载器，并单独核实其兼容性。

| 任务ID | 测什么 | 适用场景 |
|---|---|---|
| `summary` | 正确相关性、必需信息覆盖、相对长度 | 菜单、药品、化妆品、食品标签、路牌等 |
| `interpretation` | 主要含义、必要关系与条件、重要边界 | 同上，不同场景建立不同必需项 |
| `recommendation` | 菜品存在、数量、预算、忌口及金额核算 | 仅菜单 |
| `phrase` | 当地语言、用户意图、菜品数量对应、沟通有效性 | 仅菜单，且有双方可见的目标语言依据 |

四项独立评价原始回答，不重写答案，不合成一个总分。非菜单不进入菜品组合和点餐常用语主指标。

## 3. Linux获取代码与环境准备

### 3.1 获取完整仓库

使用有该私有仓库权限的GitHub账号：

```bash
git clone https://github.com/hitmhj/fine-grained-eval.git
cd fine-grained-eval
git rev-parse HEAD
```

已克隆时，先检查本地改动，再更新：

```bash
git status --short
git pull --ff-only
```

私有仓库认证使用公司允许的Git凭据方式。不要将token写入克隆URL、README或Shell历史。`Repository not found`可能是账号无权限，不能据此把仓库改成公开。

如果公司主机不能访问GitHub，在允许联网的电脑下载**整个仓库ZIP**，按公司规定传入并解压。只下载本文可以给AI理解任务，但不能代替全部Python和schema文件。不要把另一台电脑的代理地址直接复制过来，不关闭证书校验或绕过公司网络限制。

### 3.2 确定Python解释器

需要Python **3.10+**。优先激活已经可以使用公司SDK的环境，再选解释器：

```bash
PYTHON_BIN="$(command -v python3)"
"$PYTHON_BIN" -c 'import sys; print(sys.executable); print(sys.version); assert sys.version_info >= (3, 10), "需要Python 3.10+"'
```

如果SDK在另一个环境，将`PYTHON_BIN`设为该环境解释器的真实绝对路径。后文统一使用`"$PYTHON_BIN"`，不要混用不同环境的`python`、`python3`和`pip`。新开Shell后需要重新设置该变量。

评测核心仅依赖标准库和平台SDK。**不要猜测并执行`pip install hm_aigc`**，应使用平台实际提供的SDK安装方式。未来`change.py`若使用`yaml.safe_load`，其环境还需具备原推理项目使用的`PyYAML`依赖；按公司允许的依赖来源配置。

## 4. 本地目录和图片准备

先以1～2张图联调，不直接批量调用API：

```bash
mkdir -p ./local/smoke/images ./local/qwen_prompts
test -e api_config.local.json || cp api_config.example.json api_config.local.json
test -e task_config.json || cp task_config.example.json task_config.json
```

将联调图片复制到`local/smoke/images/`；若要测试组合与常用语，至少有一张真实菜单。将原始Qwen结果、真实YAML与实际生成参数放在`local/`内，不修改原始记录。

```text
fine-grained-eval/
  api_config.local.json           实际API配置，不提交
  task_config.json                实际任务prompt，不提交
  local/
    qwen_prompts/                 原推理使用的YAML
    results.json                  原始Qwen输出
    prompt_map.json               待实现适配器的显式映射
    smoke/
      images/                     1～2张原图
      input_manifest.jsonl        无答案输入清单
      truth.jsonl                 参考标准
      predictions.jsonl           转换后的原始回答
      evaluations.jsonl           评分与证据
      report.html                 离线报告
      report_summary.json         汇总
```

支持`.jpg/.jpeg/.png/.webp/.bmp`，递归扫描。当前不直接处理PDF、视频、TIFF或多图输入；另行转换时须让Qwen和评测端使用同一份转换后图片。

主键是**含扩展名的完整文件名**，如`menu_001.jpg`，不含目录。所有子目录内不允许重名或仅大小写不同的名称。图片字节必须与推理时一致；不能在推理后偷偷压缩、换图或改名。

调用API前检查：

```bash
"$PYTHON_BIN" - <<'PY'
from gen_truth import scan_images
paths = scan_images("local/smoke/images")
print("图片数量:", len(paths))
for p in paths:
    print(p.name)
PY
```

若原结果包含大数据集，后续适配器根据小样本输入清单筛选，并记录选中/跳过数，不需要手改原`results.json`删图。

## 5. 配置API并接入公司SDK

### 5.1 API配置字段

编辑`api_config.local.json`。模板为空，不包含可用账号或模型名：

```json
{
  "account": "",
  "sdk_module": "hm_aigc",
  "model_agent": "yibu",
  "reference_model": "",
  "reference_version": "",
  "judge_model": "",
  "judge_version": "",
  "timeout": 120,
  "max_retries": 3,
  "retry_wait": 1,
  "request_options": {}
}
```

| 字段 | 实际要求 |
|---|---|
| `account` | 实际平台子账号，不是通用API key字段 |
| `sdk_module` | 可导入的模块名，不带`.py`；本地包装模块可填`hm_client_local` |
| `model_agent` | 用户提供的接口示例为`yibu`；以现场实际平台为准 |
| `reference_model/reference_version` | 能看图、生成独立标准的强模型实际服务名与版本，必须非空 |
| `judge_model/judge_version` | 能看图、逐项判定的模型服务名与版本，也负责参考复核，必须非空 |
| `timeout` | 传给SDK的超时参数，给定接口按秒使用，正整数；实际行为取决于SDK |
| `max_retries` | 当前实现为最多尝试次数，默认3，即含首次最多3次，并非首次外再重试3次 |
| `retry_wait` | 重试等待基数，非负秒数，按尝试序号增加 |
| `request_options` | 平台实际支持的生成选项；不能覆盖`model/messages`，不猜平台特有参数 |

模型服务名称和版本需要向平台核实，不凭空写入某个厂商别名、价格档位或日期冒充版本。原生配置不接受额外`api_key`顶层字段；SDK需要key时通过实际认证机制或本地初始化模块提供。

可用环境变量覆盖：`FG_EVAL_ACCOUNT`、`FG_EVAL_REFERENCE_MODEL`、`FG_EVAL_REFERENCE_VERSION`、`FG_EVAL_JUDGE_MODEL`、`FG_EVAL_JUDGE_VERSION`。环境变量优先于JSON。排错时只看是否设置，不把真实值打印到聊天或公开日志。

### 5.2 SDK必须满足的接口

`common.py`实际调用：

```python
result = client.aigc_managed(
    model_agent=config["model_agent"],
    req_data={"model": model, "messages": messages, **request_options},
    sub_account_name=config["account"],
    model=model,
    timeout=config["timeout"],
)
```

请求包含system和多模态user消息；user内容中有文本JSON及`image_url`类型的base64 data URL。SDK返回值需能读取：

```python
choice = result["choices"][0]
content = choice["message"]["content"]  # 字符串，内含满足本次schema的JSON对象
finish_reason = choice.get("finish_reason")
usage = result.get("usage")            # 未提供时保留None
```

程序将`response_schema`放在文本请求中，不假定服务支持特定厂商的`response_format`。内容应是一个JSON对象字符串，允许完整JSON代码围栏，不允许额外前言或多个对象。参考/评审回包`finish_reason="length"`会被判为截断并重试。

若实际SDK有业务外壳、返回字符串或SDK对象，应在本地SDK包装层明确转换到上述形态，不能伪造`stop`和usage来通过验证。

### 5.3 模块加载的三种方式

`sdk_module`指向的模块满足任一种即可：模块自身有`aigc_managed`；模块有`hm_aigc`对象；模块有`create_client()`工厂返回带该方法的对象。

需要登录/初始化时，推荐根目录新建`hm_client_local.py`并配置`"sdk_module":"hm_client_local"`。文件已被忽略。下面仅是**尚待现场填充的占位，不能原样投入运行**：

```python
# hm_client_local.py：仅保存在公司主机
def create_client():
    # 使用平台已经验证的导入、登录和初始化代码。
    # 从实际环境/凭据存储读密钥，返回带aigc_managed的客户端。
    raise NotImplementedError("请在公司环境实现真实SDK初始化")
```

DeepSeek应参照现场成功调用示例替换函数体，不能猜SDK构造函数。不把包装模块取成原厂SDK同名，防止循环导入。

检查模块导入位置与接口形态：

```bash
"$PYTHON_BIN" - <<'PY'
import importlib
from common import load_api_config
c = load_api_config("api_config.local.json")
m = importlib.import_module(c["sdk_module"])
candidate = getattr(m, "hm_aigc", m)
print("SDK模块:", m.__name__)
print("位置:", getattr(m, "__file__", "平台内置模块"))
print("提供工厂:", callable(getattr(m, "create_client", None)))
print("直接提供接口:", callable(getattr(candidate, "aigc_managed", None)))
PY
```

此检查不调用本工具的`aigc_managed`；SDK自身导入行为由平台实现决定。导入成功不等于账号有效、模型支持图片、工厂初始化完成或回包合规，仍需真实小样本联调。

## 6. 任务prompt、当地语言与外部知识

### 6.1 保存真实输入

`task_config.json`记录Qwen实际收到的完整`system/user`，不是评分器随意重新编的一组题。历史答案必须对应当时YAML和运行时追加内容，不能换成当前模板。

只测摘要/解读时，结构如下；文本是占位，需换成真实输入：

```json
{
  "schema_version": "fg-eval/1.1",
  "context": {"locale": null, "acceptable_languages": []},
  "tasks": {
    "summary": {"prompt": {"system": "实际完整system", "user": "实际摘要user"}},
    "interpretation": {"prompt": {"system": "实际完整system", "user": "实际解读user"}}
  }
}
```

四任务结构见`task_config.example.json`。只保留要采集、要测的任务。确实预期执行但没有输出的任务须保留失败；不要通过删除困难任务抬分。字符串按精确值比较，包括空格和换行，不用`.strip()`伪造匹配。

自然提示词可以是“请你解读一下这张图片”，不需要将评分字段告诉Qwen。不同场景的必需项在参考端建立，避免把评测变成按评分提示抄答案。

### 6.2 联合推荐与常用语

同一次输出同时含组合和常用语时，两个task配置相同的实际prompt，适配器写两条记录，保留同一份完整原文，分别评分。评分器仅在prompt和output都相同的联合记录间自动配对；不同请求的随机组合不能互相替代。

若常用语输出本身含明确推荐，或用户已指定组合，也可据此判对应关系；不能从一句表达反推一个不存在的推荐依据。

### 6.3 语言与逐图覆盖

`context.locale`为双方可见的国家/地区，`acceptable_languages`是接受的语言代码，例如`["fr"]`。如果用户prompt明确要求法语，应登记相同语言。地区背景若用于评分，也必须是Qwen实际已知输入；不能只在评分端隐藏提供。

没有可靠语言依据时用null及空列表，phrase列N/A；不能默认英语，不能因出现欧元推断法国。多语言地区可登记多个适用语言。

逐图差异通过`--manifest ./local/overrides.jsonl`提供，一行一对象：

```json
{"image_name":"menu_001.jpg","context":{"locale":"FR","acceptable_languages":["fr"]}}
```

还可用同`task_config.json`结构的`tasks`覆盖逐图prompt或增加任务。当前覆盖机制没有删除单图任务语法；不同任务集合可分批配置，不杜撰`enabled`等字段。

### 6.4 专业外部资料

可选`--knowledge ./local/knowledge.jsonl`按图片提供实际核定摘录。程序不自动联网检索，不能允许参考模型凭记忆编网址。结构示例如下，所有占位必须换成真实资料：

```json
{"image_name":"medicine_001.jpg","evidence":[{"id":"ext1","source":"external","location":"实际章节/页码","text":"实际核定原文","url":"https://example.org/replace-with-real-source","version":"实际版本或访问日期"}]}
```

模型引用外部证据须逐字段原样匹配输入。没可靠依据的专业推断保留未知。资料不自动注入Qwen；若比较“基础模型”和“补充知识后的系统”，另设run_id及真实输入，不能混算。

## 7. 从无API检查到真实参考生成

### 7.1 合成测试

```bash
"$PYTHON_BIN" -m unittest discover -s tests -v
```

现有基线15项，覆盖结构、公式、故障、重试、缓存、截断和报告。故障注入测试会打印部分`ERROR:root`，最终应为`Ran 15 tests`、`OK`，不能出现`FAILED`。测试使用1像素图片及预设回包，不是真实语义评测。

既有验证环境是Windows Python 3.12；Linux现场应重新执行，不把历史记录当该主机已通过。

### 7.2 检查配置结构

```bash
"$PYTHON_BIN" - <<'PY'
from common import load_api_config
from gen_truth import task_config
api = load_api_config("api_config.local.json")
tasks = task_config("task_config.json")
print("结构有效，账号及SDK可用性尚未验证")
print("启用任务:", ", ".join(tasks["tasks"]))
print("账号已填:", bool(api["account"]))
print("参考模型/版本已填:", bool(api["reference_model"] and api["reference_version"]))
print("评分模型/版本已填:", bool(api["judge_model"] and api["judge_version"]))
PY
```

此检查不调用SDK，不打印真实凭据。结构允许空账号供准备阶段使用，真实调用前仍须填写。

### 7.3 只准备请求，不调用API

```bash
"$PYTHON_BIN" gen_truth.py ./local/smoke/images \
  --output ./local/smoke/truth.jsonl \
  --api-config api_config.local.json \
  --task-config task_config.json \
  --export-inputs ./local/smoke/input_manifest.jsonl \
  --prepare-only
```

得到`truth.requests.jsonl`和`input_manifest.jsonl`，打印`live_calls: 0`。**不会得到真正的truth或分数。** 摘要和复核请求依赖首次真实参考，故准备模式只生成最初的参考请求。这些文件含原图，也需保密。

### 7.4 生成真实标准

确认SDK、图片及prompt后，去掉`--prepare-only`：

```bash
"$PYTHON_BIN" gen_truth.py ./local/smoke/images \
  --output ./local/smoke/truth.jsonl \
  --api-config api_config.local.json \
  --task-config task_config.json \
  --export-inputs ./local/smoke/input_manifest.jsonl
```

实际使用逐图配置/外部资料时，在此及恢复命令中加入相同的`--manifest`、`--knowledge`参数。参考生成不接收Qwen回答，须在评审前冻结。

每图先生成事实/量表，摘要适用时独立生成3份候选，再机器复核。至少2份摘要经复核有效，长度中位数作为该图参考长度。每完成一图原子保存。

避免打印含base64的整行，只检查状态：

```bash
"$PYTHON_BIN" - <<'PY'
from collections import Counter
from common import read_jsonl
from gen_truth import check_truth
rows = read_jsonl("local/smoke/truth.jsonl")
for r in rows:
    check_truth(r)
print("图片数:", len(rows))
print("状态:", dict(Counter(r["status"] for r in rows)))
for r in rows:
    if r["status"] != "ok":
        print("待处理参考:", r["image_name"], "请在本地查看error/verification")
PY
```

验收时核对图片数量、参考状态、事实证据、量表必要性及专业关键项。机器复核不等于人工核验，`human_verified`默认false。不得手改指纹、复核结果或参考长度绕过错误。

`input_manifest.jsonl`含图片名/哈希/路径、实际prompt和context，不含参考答案，供适配器核对，不用于给模型补答案。

## 8. change.py：现场需完成的适配器

### 8.1 原始结果是什么

用户原推理程序保存一个JSON对象，**不是JSONL**：

```json
{
  "/original/path/menu_001.jpg": {
    "config/v04_summary_self.yaml": {
      "prompt_name": "summary_v04",
      "output": "原始模型回答",
      "elapsed_seconds": 1.23
    }
  }
}
```

失败记录仅有`{"error":"实际错误"}`。内层键是原`prompt_file`字符串，可能相对或绝对；`prompt_name`只是YAML的task/version标签。回答文本的内容由实际YAML决定，原脚本没有强制回答JSON字段。

旧文件没有实际system/user、图像哈希、token数、结束原因及完整运行配置，不能从回答或文件名猜造。

### 8.2 目标JSONL完整结构

UTF-8，一行一个JSON对象；以下展开仅为阅读。字段全部必填，拒绝额外顶层字段。64个0是占位，须计算真实图片字节哈希：

```json
{
  "schema_version": "fg-eval/1.1",
  "image_name": "menu_001.jpg",
  "image_sha256": "0000000000000000000000000000000000000000000000000000000000000000",
  "task": "summary",
  "model_id": "Qwen3.5-35B-A3B/实际版本标识",
  "run_id": "experiment-20261022-v1",
  "prompt": {"system": "实际完整system", "user": "实际完整user"},
  "context": {"locale": null, "acceptable_languages": []},
  "status": "ok",
  "output": "未经改写、清洗、纠错、截短的原始回答",
  "error": null,
  "generation": {
    "elapsed_seconds": 1.23,
    "input_tokens": null,
    "output_tokens": null,
    "max_output_tokens": null,
    "finish_reason": null
  }
}
```

| 字段 | 转换要求 |
|---|---|
| `image_name` | 含扩展名的文件名，不含目录；精确匹配清单，拒绝重名 |
| `image_sha256` | 原图文件字节的SHA-256，64位小写；不是路径字符串哈希 |
| `task` | 仅`summary/interpretation/recommendation/phrase`，显式映射 |
| `model_id/run_id` | 操作者确认的实际模型与批次；多prompt或多轮实验不要覆盖同一键 |
| `prompt` | 读取真实YAML及实际追加文本，与清单逐字符相等；不能补空system冒充 |
| `context` | 与清单完全相等，且对应Qwen真实可见背景 |
| `output` | 逐字符保留，不做`.strip()`、翻译、补全、过滤或截断 |
| `status/error` | 成功为`ok/null`；失败为`error/非空错误信息`且output空字符串 |
| `elapsed_seconds` | 原结果可靠值，没有则null |
| `input_tokens/output_tokens` | 未记录则null；不以字符数估算 |
| `max_output_tokens` | 仅根据已知实际`max_new_tokens`填写；不能因脚本默认1024而假定历史每次均如此 |
| `finish_reason` | 未记录则null；有可靠证据达到上限时为`length`，不能凭句子没有结尾判断 |

成功但空输出允许保留，评分时计模型失败。确实截断的回答保留原文，不变成空回答或自动补全。数值必须有限、非负；token等整数不接受布尔值。未知信息用null，不能用0冒充。

唯一键：`image_name + task + model_id + run_id`。同图多任务、多批次分别成行，重复键报错。完整结构见[PROTOCOL.md](PROTOCOL.md)及[schemas/prediction.schema.json](schemas/prediction.schema.json)。

### 8.3 显式prompt映射与跨平台路径

建议现场新增`local/prompt_map.json`。以下仅是未来适配器的建议配置，不是现有评分器读取的配置；示意文件名不保证存在：

```json
{
  "config/v04_summary_self.yaml": {
    "yaml_path": "local/qwen_prompts/v04_summary_self.yaml",
    "tasks": ["summary"]
  },
  "config/v04_interpret_self.yaml": {
    "yaml_path": "local/qwen_prompts/v04_interpret_self.yaml",
    "tasks": ["interpretation"]
  },
  "config/actual_joint_prompt.yaml": {
    "yaml_path": "local/qwen_prompts/actual_joint_prompt.yaml",
    "tasks": ["recommendation", "phrase"]
  }
}
```

左侧键须匹配旧结果内层实际键，`yaml_path`定位当前真实文件。YAML解析须与原脚本一致：顶层有system/user就用顶层，否则按原遍历顺序找含二者的嵌套对象。缺少实际YAML或追加输入时报告材料不足，不从`prompt_name`、文件名或输出猜任务与prompt。

源绝对路径不要求在新Linux存在，允许明确映射到当前图片根目录及YAML位置。适配器应正确解析源路径形式，拒绝文件名冲突，验证当前原图哈希与输入清单一致。由于旧文件没存图像哈希，还需操作者确认传入的是当时原文件；不能声称旧路径本身已证明图像内容一致。

联合组合/表达输出映射成两条task记录时，完整原文和实际prompt保持相同。两份YAML若在同图映射到同一task，应另立run_id或报冲突，不静默覆盖。

### 8.4 建议的适配器CLI（尚不可直接执行）

**先实现change.py再运行。以下是建议接口，不是现有功能。**

```bash
"$PYTHON_BIN" change.py \
  --results ./local/results.json \
  --image-root ./local/smoke/images \
  --mapping ./local/prompt_map.json \
  --input-manifest ./local/smoke/input_manifest.jsonl \
  --model-id "Qwen3.5-35B-A3B/实际版本标识" \
  --run-id "experiment-20261022-v1" \
  --output ./local/smoke/predictions.jsonl
```

若现场更改适配器参数，应同步帮助及说明，但不改变目标JSONL协议。转换应先校验计划行，再原子保存；默认拒绝覆盖已有文件。输出图片/记录/任务/原始失败/跳过数量，不打印密钥或完整图片数据。

需用合成数据测试正常、失败、空输出、重复键、同名图、跨平台路径、YAML缺失、prompt错配、联合输出及未知元数据。不得把失败样本替换成伪造成功回答。

### 8.5 转换后的现有协议检查

以下检查不调用API，不修改答案：

```bash
"$PYTHON_BIN" - <<'PY'
from collections import Counter
from common import read_jsonl
from contracts import check_prediction
from gen_truth import check_truth
by_name = {}
for t in read_jsonl("local/smoke/truth.jsonl"):
    check_truth(t)
    assert t["image_name"] not in by_name, "重复参考图片"
    by_name[t["image_name"]] = t
rows = read_jsonl("local/smoke/predictions.jsonl")
assert rows, "没有输出，不能据此完成小样本验收"
seen = set()
for p in rows:
    check_prediction(p)
    key = (p["image_name"], p["task"], p["model_id"], p["run_id"])
    assert key not in seen, "重复输出键"
    seen.add(key)
    t = by_name[p["image_name"]]
    assert p["image_sha256"] == t["image_sha256"], "图片哈希不一致"
    assert p["prompt"] == t["tasks"][p["task"]]["prompt"], "prompt不一致"
    assert p["context"] == t["context"], "context不一致"
print("协议检查通过，记录数:", len(rows))
print("各任务数量:", dict(Counter(p["task"] for p in rows)))
print("本检查不证明覆盖所有预期记录，也不证明回答语义正确")
PY
```

## 9. 真实评分与报告

### 9.1 先只准备评审请求

需要真正的truth及转换后的predictions，不能把`truth.requests.jsonl`当参考标准：

```bash
"$PYTHON_BIN" evaluate.py \
  ./local/smoke/truth.jsonl ./local/smoke/predictions.jsonl \
  --output ./local/smoke/evaluations.jsonl \
  --api-config api_config.local.json \
  --tasks summary interpretation recommendation phrase \
  --prepare-only
```

生成`evaluations.requests.jsonl`，不产生真实评分。不在该图truth任务配置中的任务跳过；只测摘要/解读时使用`--tasks summary interpretation`。

### 9.2 真正调用评分API

```bash
"$PYTHON_BIN" evaluate.py \
  ./local/smoke/truth.jsonl ./local/smoke/predictions.jsonl \
  --output ./local/smoke/evaluations.jsonl \
  --api-config api_config.local.json \
  --tasks summary interpretation recommendation phrase
```

评审器返回逐项标签、候选原文引用及证据，不直接给总分。程序计算分数；引用必须是候选原文子串，证据ID须存在于冻结标准。不能关闭校验来掩盖错误回包。

检查评分记录：

```bash
"$PYTHON_BIN" - <<'PY'
from collections import Counter
from common import read_jsonl
from gen_html import validate_result
rows = read_jsonl("local/smoke/evaluations.jsonl")
assert rows, "评分文件为空"
for r in rows:
    validate_result(r)
print("记录数:", len(rows))
print("状态:", dict(Counter(r["evaluation_status"] for r in rows)))
print("模型/批次:", sorted({(r["model_id"], r["run_id"]) for r in rows}))
PY
```

| 状态 | score | 含义 |
|---|---|---|
| `scored` | 0～1 | 项级标签通过校验，由程序计算 |
| `missing_prediction` | 0 | 预期输出缺失，保留分母；核对采集/适配完整性 |
| `model_error` | 0 | 原推理失败或空回答 |
| `not_applicable` | null | 冻结标准判不适用，单列数量，不入均值 |
| `reference_error` | null | 标准失败，先处理参考，不归因为模型差 |
| `judge_error` | null | 评分API或回包失败，需重试，不当模型0分 |

退出码为0不等于模型高分。模型缺失/失败可正常计0并完成评分；参考/评审失败会在汇总保留待评并非零退出。必须同时检查退出码、汇总和逐行状态。

### 9.3 生成HTML

```bash
"$PYTHON_BIN" gen_html.py ./local/smoke/evaluations.jsonl \
  --output ./local/smoke/report.html \
  --summary-output ./local/smoke/report_summary.json \
  --title "Qwen3.5-35B-A3B 图片能力测评"
```

也可使用同参数`get_html.py`。报告无前端依赖、无CDN，图片与样式内嵌。Linux无桌面时，通过公司允许方式下载`report.html`，在本机浏览器离线打开即可，不必搭公开Web服务。

报告按模型/批次/任务/场景分类，包含每图输入prompt、原图、原始输出、参考量表、具体计分、证据、问题和结论，以及总体数量、均值、长度和截断记录。分数会按标签重新计算，手改分或数据不一致会被拒绝。

完成小样本后另建`local/batch-实际批次名/`运行全量，不把联调1～2图当总体能力结论。

## 10. 恢复运行、批次与调用量

### 10.1 中断恢复

同一输入中断后，在原命令后加`--resume`。例如评分：

```bash
"$PYTHON_BIN" evaluate.py \
  ./local/smoke/truth.jsonl ./local/smoke/predictions.jsonl \
  --output ./local/smoke/evaluations.jsonl \
  --api-config api_config.local.json \
  --tasks summary interpretation recommendation phrase \
  --resume
```

参考生成同理。成功且签名相同的记录复用，参考/评审失败可重试。参考以图片为恢复单位，失败图会重做整套参考，不保证从某份摘要继续；评分以单条记录为单位。不要两个进程同时写同一文件。

图片、prompt、上下文、资料、参考、API配置或任务范围变化，缓存可能不再匹配。应换新输出路径，不能删除指纹绕过检查。补充/改正predictions后，若签名变更，同样用新的评分输出。

### 10.2 多模型及完全没有输出

不同`model_id/run_id`可在一个predictions文件内，报告分别汇总。完全空的批次无法从输出推断身份，应成对传`--model-id`、`--run-id`，让应有的缺失记0，不造假答案让程序启动。

同图两种真实prompt不能套一条不匹配的truth任务；分别生成对应参考和评分。不能看模型表现后修改gold抬分。量表在生成前已定，难例不可按分数事后剔除。

### 10.3 调用数与资源

适用摘要的图片参考阶段通常5次逻辑调用；摘要不适用时通常2次。每条适用、正常非空的输出评分通常1次。四任务均适用的菜单约5+4=9次；摘要与解读约5+2=7次。重试、失败图重做和SDK自身重试会增加调用。

这里只估调用数，不估费用，价格和图像计费以现场服务为准。先查看小样本实际usage，再按额度扩展。SDK未返回usage则null。

truth及评分文件内嵌原图，后者还会重复保存关联输入。当前JSONL整文件读入内存，超大数据要按明确批次组织并检查磁盘/内存，不声称无限流式处理。

## 11. 指标、长度、截断及结论口径

详细公式见[METRICS.md](METRICS.md)，现场不能为使程序运行而更改主规则。存储分数为0～1，HTML显示百分数。

### 11.1 摘要

令$C_i$为去重输出事实数，$V_i$为正确相关事实数，$G_i$为必需事实数，$c_i$为正确覆盖数，$\lambda_i$为实际长度，$\lambda_i^{ref}$为该图有效参考摘要长度中位数：

$$
P_i=\frac{V_i}{C_i},\quad R_i=\frac{c_i}{G_i},\quad
S_i=\frac{2P_iR_i}{P_i+R_i}\min\left(1,\frac{\lambda_i^{ref}}{\lambda_i}\right).
$$

P检查说出的是否正确相关，R检查应保留的是否完整。参考100字符、实际150字符，长度比1.5，系数2/3；短于参考不扣长度系数，但遗漏降低R。重复不加覆盖，原文仍计长度。长度为Unicode非空白字符，不是token上限；空回答/无有效信息得0，不先裁剪再评分。

### 11.2 AI解读

字段为主要含义、必要关系与条件、重要边界，按图预先固定适用字段及少量必需项。令$C_{if}/K_{if}$为字段正确率，$K_i$为全部必需项数，$H_i$为额外无效解释的去重数：

$$
D_i=\left(\frac1{|\mathcal F_i|}\sum_{f\in\mathcal F_i}\frac{C_{if}}{K_{if}}\right)\frac{K_i}{K_i+H_i}.
$$

每项有命题、限定、适用理由、推理、接受答案、错误边界、证据及关键项标记；正确且必要限定完整得1，错误/遗漏得0。字段取答对率，不是整图全对才得1。同一必需项错误不再重复计H。

食品标签重视每份/每100克换算，路牌重视对象时间及例外，药品重视标签用途、用法条件和证据边界；不机械要求同样细节，也不要求固定三个标题。正确可选补充不加不扣，无依据/无关扩展计入额外错误。

### 11.3 组合与常用语

$$
B_i=\frac{\text{已满足约束数}}{\text{本图适用约束总数}},\qquad
L_i=\frac{\text{成功表达单元数}}{\text{预定表达单元数}}.
$$

组合按固定类别检查全部选择，避免拆大量容易约束稀释违规。未提供的偏好不补设；未知配料/辣度不能认定安全。关键预算/过敏项未通过时，即使均分较高，也不能称方案合规可用。

常用语每预定单元同时满足语言适用、意图、菜品数量对应、沟通有效。按候选实际组合核对，组合超预算不会自动扣语言分；语言自身歪曲条件仍应扣分。

### 11.4 总体与长度诊断

先算项，再算图，最后图片等权平均；按场景、模型、批次、任务分别汇总，不把长图独立项全部混成大分母。

token数量不是API认证token。原推理没记录usage/结束原因就显示未知，不能以字符数估算，也不能将“没记录”说成“没截断”。截断比例分母只包含结束原因已知的成功输出，未知数单列。长度与截断记录不额外增加第五个主指标。

参考/评审失败是待评，不是模型0分；有待评只能报告部分结果。置信区间只反映已评分图片抽样波动，不包括机器gold或评审偏差；同模板/产品变体应按独立来源组织采样。

正式结论需人工子集校准参考及评审器，特别是数字、否定、未知与专业关键项。没有业务批准的最低要求和关键错误规则，不能擅自宣称上线可用。

## 12. 常见故障与排查顺序

按“环境→输入→SDK/权限→回包→参考质量→模型成绩”定位。不要把工程故障归为模型能力差。

| 现象/报错 | 处理 |
|---|---|
| Git认证失败/Repository not found | 检查私有仓库权限、地址及公司认证，不将token写URL，不改公开 |
| Python不存在/版本低 | 找已有3.10+环境并设`PYTHON_BIN`，不重装系统Python |
| No module named hm_aigc | 查实际SDK模块名和安装环境，确认解释器与模块位置，不猜公网包 |
| No module named yaml | 未来适配器的YAML依赖不在当前环境；标准库评分器本身不依赖它 |
| SDK must expose... | 模块是否暴露接口/对象/工厂，是否需要初始化 |
| configure account...version | 账号及当前角色模型/版本是否为空，环境变量是否覆盖 |
| 认证/模型不存在/图像权限错误 | 用平台已成功样例检查实际账号、服务名、版本、图片权限及网络 |
| image directory is empty | 图片目录、扩展名、拷贝位置 |
| duplicate image basename | 跨子目录重名或大小写冲突，统一数据映射后再运行 |
| 图片找不到/哈希不同 | 跨平台路径映射、原图字节、同名换图；不能仅改哈希通过 |
| prompt/context不匹配 | 实际YAML、追加内容、空格换行、逐图背景；不伪造输入来匹配 |
| duplicate prediction key | 同图同任务同模型同批次多份结果，分run_id或拒绝冲突 |
| unknown keys/expected type | 严格字段和类型，JSONL不是JSON数组，不加臆造列 |
| 非法JSON/NaN/重复键 | 文件编码、每行一对象、数值有限性及回包完整性 |
| API output truncated | 参考/评审达到上限，按平台支持的参数调整容量；不修补残缺JSON当成功 |
| message.content非字符串 | SDK响应外壳须明确适配，不伪造usage/结束原因 |
| quote不在原文/未知证据ID | 评审引用有误，保留judge_error，核对请求与模型，不关闭校验 |
| 外部证据不匹配 | 是否实际提供资料、模型是否编来源或改写证据对象 |
| 参考复核失败/有效摘要不足2 | 本地查看verification/issues，检查图和量表质量，再重生成 |
| output exists | 原输入恢复用resume；新实验换路径，不覆盖历史 |
| resume signature mismatch | 输入/配置/范围已改变，另建输出，不删除签名绕过 |
| score/details重算不一致 | 曾改分或标签不一致，从原始数据重新评分 |
| 大量0分 | 区分缺输出、推理失败和真实逐项答错，再看均值 |
| phrase全部N/A | 是否真实菜单、双方可见目标语言充分；不能事后补隐藏国家 |
| token与截断未知 | 历史推理没采集；如实保留，未来运行再增加可靠记录 |

故障记录存`local/`，包含阶段、文件名和必要脱敏错误；不将原始SDK认证异常、环境变量全量或base64整行粘贴到聊天。

## 13. 保密与现场最终交付

当前代码、空配置、schema及合成测试检查未发现有效认证凭据，参见[SECURITY.md](SECURITY.md)。后续实际账号、key、SDK初始化及业务数据都须保密。

API阶段会把图片、prompt、参考资料及模型回答发给所配置的平台；base64是编码，不是脱敏，必须确认该材料允许进入该平台。HTML生成不调用API，但文件内嵌原图和回答，保密级别与原数据相同。

`.gitignore`排除本地配置、`*_local.py`、`local/`、图片、JSONL、HTML和日志。规则不是脱敏，强制添加、改名或在代码硬编码密钥仍可能泄露。可检查：

```bash
git status --short
git check-ignore api_config.local.json hm_client_local.py local/smoke/truth.jsonl local/smoke/report.html
```

未来提交适配器时仅提交无凭据代码、说明和合成测试，不提交真实YAML、prompt映射或模型答案。若有效凭据误入Git，先撤销/轮换，再处理历史；仅删除当前文件不使凭据失效。

现场DeepSeek最终应交付：

| 项目 | 要求 |
|---|---|
| 环境 | 实际仓库提交、Python路径/版本、SDK模块/来源，不含密钥 |
| 实际接口 | 参考/评审模型名与版本、真实调用成功或失败证据；与mock明确区分 |
| 适配器 | change.py实现、CLI帮助、合成测试及协议检查结果 |
| 输入追溯 | 原图、哈希、真实YAML、显式映射与实际语言背景 |
| 产物 | truth、predictions、evaluations、HTML、汇总在公司主机的路径 |
| 完整性 | 预期/实际数量，各任务/场景状态，缺失、N/A、参考/评审失败单列 |
| 质量 | 人工核对若干图、参考、回答与逐项分数的对应关系 |
| 待补 | 未完成的步骤、原因、需要的材料，不伪造验证成功 |

HTML能生成只代表报告渲染完成；输入一致、参考成立、真实评审有效且失败没有隐藏，才可作为一轮完整测评。后续文档：[严格协议](PROTOCOL.md) · [指标公式](METRICS.md) · [保密说明](SECURITY.md) · [既有验证](VALIDATION.md) · [代码来源](SOURCE.md)。
