# Cambright One HRMS 代码健康审查报告

审查日期：2026-10-18  
审查对象：本仓库（远程 `zrzcool/hrms`，应用版本 `17.0.0-dev`，基线分支 `develop`，提交 `d6b465392`）  
审查方式：只读。未修改任何应用代码。

本仓库是 Frappe HR（`hrms`）的分支，代码内没有 Cambright 品牌或独立业务层。本报告把当前树视为 Cambright One HRMS 的实现，并按 Frappe 应用（Python DocType + ERPNext 集成 + Vue PWA / 排班前端）来评估。

## 1. 总体健康分

**6 / 10**

主干业务是可维护的成熟 HRMS，而不是失管项目。薪资单、工资批次、请假申请有数千行集成测试；按仓库锁定的 Ruff 0.3.7 检查 Python 全部通过；`python3 -m compileall hrms` 通过。架构也符合 Frappe 惯例：HR / Payroll 两模块、`hooks.py` 接入 ERPNext、白名单 API 服务移动端。

分数停在 6，是因为安全和工程链路已经拖后腿：

- 多份薪资报表把筛选条件用 `%` / `.format()` 拼进 SQL。
- 订阅相关接口对访客开放，订阅密钥出现在 URL 查询串里。
- 前端依赖 `yarn audit` 报出严重漏洞；ESLint 在当前 Node 上无法启动；排班应用 `tsc --noEmit` 失败。
- CI 忽略 `.js` 变更、不为 PR 上传覆盖率、没有前端构建或依赖审计。

这还没有到“不能继续开发”的程度，但安全与工具链应进入近期迭代，而不是当作背景债。

## 2. 分项评分

| 领域 | 评分 | 简要判断 |
| --- | ---: | --- |
| 架构与结构 | 7 | 模块边界清楚，与 Frappe / ERPNext 的接入方式稳定；薪资与 API 文件过大 |
| 代码质量与重复 | 6 | Ruff 通过，类型注解在 API 上有约束；前后端代理逻辑复制，存在错误的实时事件 DocType |
| 依赖 | 4 | 相对 2026-10 的登记版本明显落后；前端审计有严重项 |
| 安全 | 5 | 大部分 SQL 已参数化，仓库内无生产密钥；报表注入、访客接口和 `v-html` 是实质问题 |
| 测试与覆盖率 | 6 | 核心薪资 / 请假测试很厚；`hrms/api` 与两个前端没有测试；PR 不采集覆盖率 |
| 构建 / Lint / 类型检查 | 5 | Python 编译与 Ruff 通过；前端 ESLint 配置无法加载，排班 `tsc` 失败 |
| 性能 | 5 | 工资批次、请假考勤、自动考勤按员工/按日逐条 `get_doc`；PWA 员工列表无实际上限 |
| 错误处理与日志 | 6 | 批量工资有失败记录；补丁和订阅逻辑吞掉异常；没有结构化日志 |
| 文档 | 4 | 用户文档在站外；仓库内 DocType README 多为占位，没有架构说明，前端无 README |
| CI 与配置 | 5 | Python 并行测试和 Semgrep 在位；路径过滤、缓存键、Python 版本和前端缺口明显 |

## 3. 审查范围与已执行检查

覆盖了架构、重复代码、依赖版本与 `yarn audit`、认证与注入、测试布局、性能热点、错误处理、文档和 `.github/workflows`。

在本环境实际跑过：

| 检查 | 结果 |
| --- | --- |
| `python3 -m compileall -q hrms` | 通过 |
| `ruff 0.3.7 check hrms`（与 `.pre-commit-config.yaml` 的版本一致） | `All checks passed!` |
| `frontend`：`yarn install --frozen-lockfile` 后 `yarn eslint --ext .js,.vue src` | 失败。`frontend/package.json` 为 `"type": "module"`，`.eslintrc.js` 使用 `module.exports`，Node 22 报 `module is not defined` |
| `roster`：`yarn tsc --noEmit` | 失败。`src/main.ts`、`src/router.ts` 无法解析 `.vue` 模块。`roster/index.d.ts` 只声明了 `frappe-ui` |
| `frontend`：`yarn audit --groups dependencies` | 276 条：严重 10、高 140、中 98、低 28。严重项合并后是 3 个包，见下文 |
| Frappe `bench run-tests` | **未运行**。本环境没有 Frappe / ERPNext bench 和 MariaDB 站点，无法启动应用级测试 |

npm 登记版本取自 2026-10-18 的 `npm view`，用于和锁文件对照，不是建议立刻跳到的目标主版本。

## 4. 架构与结构

应用声明依赖 ERPNext，版本窗口在 `pyproject.toml`：

```72:74:pyproject.toml
[tool.bench.frappe-dependencies]
frappe = ">=17.0.0-dev,<18.0.0"
erpnext = ">=17.0.0-dev,<18.0.0"
```

`hrms/hooks.py` 把 Employee、Timesheet、Payment Entry 等类覆盖和 `doc_events` 接到 ERPNext，并把网站路由 `/hrms`、`/hr` 指到 PWA 与排班壳。Python 业务在 `hrms/hr` 与 `hrms/payroll`；移动端 API 集中在 `hrms/api/__init__.py`（828 行）；`frontend/` 是 Ionic + Vue 3 PWA，`roster/` 是 Vue 3 + TypeScript 排班界面。两者构建产物进入 `hrms/public`。

这个结构适合 Frappe 应用，问题是热点文件已经变成上帝对象：

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `hrms/payroll/doctype/salary_slip/salary_slip.py` | 2807 | 薪资计算、税务、公式、福利、缺勤、邮件与 PDF |
| `hrms/payroll/doctype/payroll_entry/payroll_entry.py` | 1828 | 批次创建、提交、取消、会计凭证 |
| `hrms/hr/doctype/leave_application/leave_application.py` | 1521 | 请假校验、台账、考勤同步、通知 |
| `hrms/hr/utils.py` | 1042 | 假期、考勤、校验的共享函数 |
| `hrms/api/__init__.py` | 828 | 约 30 个以上 PWA 白名单方法 |

根 `package.json` 写的是 `"aworkspaces"`，不是 Yarn 的 `"workspaces"`。安装靠 `postinstall` 分别进入 `frontend` 和 `roster`，两个前端没有真正共享包。

## 5. 主要问题

### 5.1 高：薪资报表把筛选值拼进 SQL

`hrms/payroll/report/provident_fund_deductions/provident_fund_deductions.py` 的 `get_conditions` 用旧式 `%` 格式化：

```57:79:hrms/payroll/report/provident_fund_deductions/provident_fund_deductions.py
def get_conditions(filters):
	conditions = [""]

	if filters.get("department"):
		conditions.append("sal.department = '%s' " % (filters["department"]))
	# branch、company、month、year、mode_of_payment 同样拼接
	return " and ".join(conditions)
```

随后原样插入查询：

```113:118:hrms/payroll/report/provident_fund_deductions/provident_fund_deductions.py
	salary_slips = frappe.db.sql(
		""" select sal.name from `tabSalary Slip` sal
		where docstatus = 1 %s
		"""
		% (conditions),
		as_dict=1,
	)
```

同一函数被另外两份报表导入：

- `hrms/payroll/report/professional_tax_deductions/professional_tax_deductions.py`（约第 8、51–62 行，并用 `# nosemgrep` 压掉了 SQL 规则）
- `hrms/payroll/report/salary_payments_based_on_payment_mode/salary_payments_based_on_payment_mode.py`（约第 10、73–80 行）

`hrms/payroll/report/salary_payments_via_ecs/salary_payments_via_ecs.py` 自己复制了一份同样的拼接（约第 67–85、113–117 行），同时 `frappe.get_list("Employee")` 不加过滤，把银行账号、IFSC 载入内存（约第 91–95 行）。

`hrms/hr/report/employee_hours_utilization_based_on_timesheet/employee_hours_utilization_based_on_timesheet.py` 第 142–151 行把 `from_date` / `to_date` 用 f-string 放进 SQL，并用 `nosemgrep` 跳过规则。

能打开这些报表的用户就可以用筛选值打断 SQL 字符串。报表权限不等于数据库权限。应改成 Query Builder 或 `%(name)s` 占位符。仓库里大部分查询已经是这种写法，例如请假和部门审批。

### 5.2 高：访客可调用的订阅接口，密钥在 URL 里

```30:69:hrms/subscription_utils.py
@frappe.whitelist(allow_guest=True)
def get_add_on_details(plan: str) -> dict[str, int]:
	# ...
	active_employees = get_active_employees()

@frappe.whitelist(allow_guest=True)
def subscription_updated(app: str, plan: str):
	if app in ["hrms", "erpnext"] and plan:
		update_erpnext_access()
```

`get_add_on_details` 不校验调用方，返回按在职员工数算出的计费附加人数。`get_active_employees` 是 `frappe.db.count("Employee", {"status": "Active"})`。

`subscription_updated` 意图在收到订阅变更后改 ERPNext 工作区和角色，但 `update_erpnext_access(user_input: dict | None)` 的参数没有默认值，这里却零参数调用，当前会先 `TypeError`。函数体内也不使用 `user_input`，并且只在站点 URL 以 `.frappehr.com` 结尾时继续。这是一条未鉴权、且目前调用本身就是坏的入口。`hooks.py` 第 104 行的安装向导完成回调同样指向这个函数。

```175:177:hrms/subscription_utils.py
def has_subscription(secret_key) -> bool:
	url = f"https://frappecloud.com/api/method/press.api.developer.marketplace.get_subscription_status?secret_key={secret_key}"
	response = requests.request(method="POST", url=url, timeout=5)
```

密钥在查询串中，会进入代理日志和异常信息。主机是固定的 `frappecloud.com`，不是开放 SSRF。密钥来自 `frappe.conf`，没有写进仓库。

其他 `allow_guest=True`：`hrms/api/oauth.py`（只返回授权 URL，不返回 client secret）、`hrms/api/system_settings.py`（是否关闭密码登录）、`hrms/utils/__init__.py` 的 `get_country`。这些更接近登录页需要的公开信息。

### 5.3 高：前端依赖过旧，审计有严重项

锁文件版本对照 2026-10-18 的 npm 最新版：

| 包 | 本仓库锁定 | 登记最新版 | 位置 |
| --- | --- | --- | --- |
| eslint | 8.41.0 | 10.13.0 | `frontend/yarn.lock` |
| prettier | 2.8.8 | 3.9.10 | `frontend` |
| vite | 5.4.15 | 8.3.4 | `frontend`、`roster` |
| firebase | 10.8.0 | 13.0.0 | `frontend` |
| @ionic/vue | 7.4.3 | 9.0.8 | `frontend` |
| frappe-ui | 0.1.105（精确钉死） | 1.0.2 | `frontend`、`roster` |
| postcss | 8.4.23（直接依赖范围） | 随 8.5 安全修复 | `frontend` |
| ruff（pre-commit） | 0.3.7 | 0.17.1 | `.pre-commit-config.yaml` |

`yarn audit --groups dependencies`（`frontend`，808 个包）汇总为严重 10、高 140、中 98、低 28。严重告警合并后是 3 个问题：

- `protobufjs`（经 `firebase > @firebase/firestore > @grpc/proto-loader`）：任意代码执行，修复版本 `>=7.5.5`。
- `websocket-driver`（经 `firebase > @firebase/database > faye-websocket`）：协议长度头可破坏消息，修复版本 `>=0.7.5`。
- `@babel/traverse`（经 `vite-plugin-pwa > workbox-build`）：恶意代码编译时的任意代码执行，修复版本 `>=7.23.2`。这条在构建链上，不在浏览器运行时。

直接依赖上的高危项包括锁定的 `postcss@8.4.23`（`sourceMappingURL` 导致的任意文件读取，修复版本 `>=8.5.12` / `>=8.5.18`）和 `vite@5.4.15`（`server.fs.deny` 绕过，修复版本 `>=6.4.3`，主要影响开发服务器）。高危数量里有大量 `minimatch`、`brace-expansion`、`semver` 的 ReDoS，被传递依赖重复计数，不能把 140 理解成 140 个独立生产漏洞。

CI 没有任何 `yarn audit` / `pip-audit` 步骤。

### 5.4 高：前端检查是坏的，CI 也不跑前端

`frontend/.eslintrc.js` 第 1 行是 `module.exports`，而 `frontend/package.json` 第 5 行是 `"type": "module"`。在 Node 22.14 上 ESLint 8.41.0 无法读取配置。配置还 `extends` 了 `plugin:prettier/recommended`，`package.json` 却没有 `eslint-plugin-prettier` / `eslint-config-prettier`，也没有 `lint` 脚本。

`roster/tsconfig.json` 打开了 `strict` 并包含 `src/**/*.vue`，但没有 `declare module "*.vue"`。`tsc --noEmit` 在 `src/main.ts:5` 和 `src/router.ts:7` 失败。排班目录没有 ESLint。

`.github/workflows/ci.yml` 第 4–12 行对 PR 忽略 `**.js`、`**.css`、`**.md`、`**.html`。只改桌面端 JS 不会跑 Python CI；`.vue` 会触发 Python 测试，但没有任何 job 执行 `yarn build`、ESLint 或类型检查。`.pre-commit-config.yaml` 第 25–34 行把整个 `frontend/` 排除在 Prettier 之外。

### 5.5 中：PWA API 把对象级权限完全交给 DocPerm

`hrms/api/__init__.py` 的白名单方法要求登录，但没有把 `employee` 绑定到当前会话员工。`get_current_employee()` 已经存在（约第 81–85 行），列表接口仍接受调用方传入的员工：

- `get_all_employees`（约第 63–78 行）对 Employee 使用 `limit=999999`。
- `get_leave_applications`、`get_expense_claims` 按传入的 `employee` 过滤。
- `get_attachments` / `upload_base64_file` / `delete_attachment` / `_download_pdf`（约第 720–787 行）按客户端给出的 DocType 和单号读写附件或工资单 PDF。

Frappe 的 `get_list` / `insert` / `delete_doc` 默认检查 DocType 权限，所以这不是“任意访客即可下载工资单”。风险在权限被配宽时没有第二道“只能看自己或自己审批的单据”的约束，工资单 PDF 和附件尤其敏感。`hrms/api/` 下没有 `test_*.py`。

`hrms/hr/doctype/employee_checkin/employee_checkin.py` 第 159–210 行的 `add_log_based_on_employee_field` 允许客户端指定 `employee_fieldname`（默认 `attendance_device_id`），再用该字段查找员工并 `insert` 打卡。这是考勤机集成的入口，字段名应做成白名单，并限制为集成用户。

### 5.6 中：通知 HTML 与公式 `eval`

`hrms/mixins/pwa_notifications.py` 第 27、42–44 行用 `frappe.bold()` 把员工姓名、单据名拼进 `message`。`frontend/src/views/Notifications.vue` 第 73 行 `v-html="item.message"`。姓名若含 HTML，会在审批人或员工的 PWA 里执行。排班界面还有两处 `v-html`：`roster/src/components/MonthViewTable.vue` 约第 116 行（假期说明）、`roster/src/components/ShiftAssignmentDialog.vue` 第 117 行。

`hrms/payroll/doctype/salary_slip/salary_slip.py` 第 2744–2764 行的 `_safe_eval` 对薪资公式调用 `eval`。文件明确写了不要在别处使用。它清掉了 `__builtins__`，并做了属性名和 AST 检查，公式来自薪资组件配置而不是终端用户。残余风险在能改薪资组件的管理员。这是有意偏离 `frappe.safe_eval` 的实现，应保持隔离并补恶意公式测试。

### 5.7 中：批量薪资与请假的逐条写入

`hrms/payroll/doctype/payroll_entry/payroll_entry.py` 第 1597–1599 行对每个员工 `frappe.get_doc(...).insert()`，第 1680 行起再逐张 `get_doc` + `submit`。取消路径（约第 199–234 行）同样逐张加载工资单和凭证。

`hrms/hr/doctype/leave_application/leave_application.py` 的 `update_attendance`（第 323–355 行）按日期循环 `frappe.db.exists` 和 `get_doc`。`hrms/hr/doctype/shift_type/shift_type.py` 的自动考勤对员工和日期做类似循环。

这些路径在员工多、请假跨度长时会放大成数据库往返。查询构建器已经用在部分薪资查询上，批量创建还没有。

### 5.8 中：错误被吃掉，实时事件发错 DocType

`except Exception` 在 `hrms/**/*.py` 中约 45 处。静默 `pass` 有三处：

- `hrms/patches/post_install/move_tax_slabs_from_payroll_period_to_income_tax_slab.py` 第 111–112、139–140 行：迁移失败不写 Error Log。
- `hrms/subscription_utils.py` 第 119–120 行：禁用角色失败被忽略。同文件第 107–108 行只 `frappe.clear_messages()`。

`hrms/hr/doctype/leave_control_panel/leave_control_panel.py` 第 91–95 行和第 132–136 行把批量请假的 `publish_realtime` 发到 DocType `"Bulk Salary Structure Assignment"`。订阅请假分配完成事件的客户端收不到这条消息。

`hrms/payroll/doctype/salary_slip/salary_slip.py` 第 2622–2624 行在 `frappe.throw` 之后还有不可达的 `raise`。

全库 Python 没有 `logging` / `structlog`。运行时可见性依赖 `frappe.log_error`（约 21 处）、`msgprint` 和 realtime。工资批次失败有 `log_payroll_failure`，这是较好的局部模式。

### 5.9 中：测试厚在核心单据，空在 API 和前端

| 指标 | 数量 |
| --- | ---: |
| `hrms` 下 Python 文件 | 659 |
| `test_*.py` | 124 |
| 有控制器且体积大于 200 字节、但没有同目录测试的 DocType | 57（多数是子表） |
| `frontend` / `roster` 的 `*.spec.*` / `*.test.*` | 0 |

厚的测试：`test_salary_slip.py` 3038 行，`test_leave_application.py` 1512 行，`test_payroll_entry.py` 1273 行。

没有对应测试文件的重要模块：`hrms/api/__init__.py`、`hrms/api/roster.py`、`hrms/hr/utils.py`、`hrms/overrides/`、`hrms/setup.py`。

`codecov.yml` 要求 PR 补丁覆盖率 85%，但 `.github/workflows/ci.yml` 第 29、118、122、131 行只在非 PR（每日定时）时采集并上传覆盖率。PR 上看不到这条门禁。

### 5.10 低到中：CI 与文档的具体缺陷

- `.github/workflows/ci.yml` 第 97 行和 `patch.yml` 仍使用已废弃的 `::set-output`。`run-individual-tests.yml` 已改用 `$GITHUB_OUTPUT`。
- 三份测试工作流用 `hashFiles('**/package-lock.json')` 做 npm 缓存。仓库只有 `yarn.lock`，没有 `package-lock.json`，该缓存键无效。
- `.github/workflows/docs_checker.yml` 使用 Python 3.8。项目 `requires-python` 是 `>=3.10`，主 CI 和 linter 使用 3.14。3.8 已停止维护。
- MariaDB 镜像不一致：主 CI `mariadb:11.8`，单独测试工作流 `10.6`，`docker/docker-compose.yml` 为 `10.8`。
- `.releaserc` 只发布 `version-14`，而树版本是 `17.0.0-dev`。发布工作流和开发分支的对应关系需要人工核对，否则 develop 上的修复不会走这条发布链。
- 仓库没有 `docs/` 架构说明（本报告是新增文件）。`hrms/hr/README.md` 是条目列表。`hrms/payroll/doctype/salary_slip/README.md` 等是一行说明。`frontend/`、`roster/` 没有 README。
- 开发凭据：`docker/init.sh` 使用 MariaDB 密码 `123`、管理员密码 `admin`；`.github/helper/site_config.json` 是 CI 夹具（`admin` / `root` / `test_frappe`）。这些是本地和 CI 默认值，不是生产密钥。仓库中未发现 `.env` 或云厂商私钥。

重复代码的具体例子：`frontend/vite.config.js` 第 92–128 行与 `roster/vite.config.js` 第 37–73 行的代理函数几乎相同；两边各有一份 `Link.vue` 和 dayjs 初始化。

## 6. 优先建议

工作量按改动范围估计：小 = 局部文件即可落地；中 = 跨几个模块或需要补测试；大 = 要拆分核心单据并回归薪资。

### 高

1. **把薪资报表筛选改为参数化查询。** 先改 `provident_fund_deductions.get_conditions`，再改 ECS 报表和工时报表里的 f-string。去掉对应的 `nosemgrep`，改完用恶意筛选值做一次报表测试。工作量：小。
2. **收紧订阅接口。** `get_add_on_details` 去掉 `allow_guest` 或校验共享密钥。`subscription_updated` 补上签名校验，并修正对 `update_erpnext_access` 的错误调用。`has_subscription` 把 `secret_key` 放到请求头，避免进 URL。工作量：小。
3. **升级运行时依赖并固定审计。** 优先处理 `firebase` 传递的 `protobufjs`、`websocket-driver`，以及直接依赖 `postcss`。`vite` 的 `server.fs.deny` 问题在升级主版本时一并处理。CI 增加 `yarn audit`（至少对 `frontend` 生产依赖的严重级别失败）。不要在同一次改动里把 Ionic、frappe-ui、Vite 全部跳到最新主版本。工作量：中。
4. **让前端检查能在本地和 CI 跑起来。** 把 `frontend/.eslintrc.js` 改成可被 `"type": "module"` 加载的配置，补上 Prettier 相关依赖和 `lint` 脚本；给 `roster` 增加 `*.vue` 模块声明或改用 `vue-tsc`。CI 对 `frontend` 和 `roster` 至少执行安装、lint、类型检查和 `yarn build`。同时删掉 CI 里对 `**.js` 的一刀切忽略，或改成只忽略翻译和快照。工作量：中。

### 中

5. **给 PWA API 加会话约束和测试。** 自助接口默认使用 `get_current_employee()`；审批人接口显式检查审批关系；`_download_pdf`、附件上传和删除先确认调用方对目标单据有打印或写权限。为 `hrms/api` 增加权限矩阵测试。工作量：中。
6. **打卡字段名白名单。** `add_log_based_on_employee_field` 只允许 `attendance_device_id` 等固定字段，并限制角色。工作量：小。
7. **通知改为转义后的文本。** PWA 消息不要把未转义的姓名放进 HTML；`Notifications.vue` 用文本插值或先转义再允许有限标签。工作量：小。
8. **批量写入批量化。** 工资批次创建/提交和请假考勤同步避免逐日 `get_doc`。先加批量查询和失败隔离，保持现有 `test_payroll_entry.py`、`test_leave_application.py` 为回归网。工作量：中。
9. **修实时事件 DocType，并停止吞迁移异常。** `leave_control_panel.py` 的 `publish_realtime` 改为请假相关 DocType。税务迁移补丁的 `except Exception: pass` 改为 `frappe.log_error`。工作量：小。
10. **PR 覆盖率与核心 API 测试对齐。** 让 PR 也产生 coverage artifact，否则 `codecov.yml` 的 85% 补丁门槛不会生效。优先补 `hrms/api` 和报表 SQL 的测试，而不是再堆子表测试。工作量：中。

### 低

11. **升级预提交工具。** Ruff 从 0.3.7 升到当前 0.17 线，先只在 CI 观察新增规则，再收紧 `pyproject.toml` 里被忽略的 `E501`、`F401`、`B904`。工作量：小。
12. **对齐 CI 杂项。** 文档检查改为 Python 3.12+；`::set-output` 改为 `$GITHUB_OUTPUT`；npm 缓存改为 `yarn.lock` 或删除；统一 MariaDB 主版本。工作量：小。
13. **抽出前端共享的 Vite 代理。** `frontend` 与 `roster` 的 `getProxyOptions` 只留一份。顺手修正根 `package.json` 的 `aworkspaces`。工作量：小。
14. **补内部文档。** 说明 HR / Payroll / `hrms/api` / 两个前端的边界、工资计算入口和本地测试如何跑。DocType 下一行 README 可以保持，不要指望它们承载架构。工作量：小到中。
15. **拆分 `salary_slip.py`。** 按税务、公式、考勤扣款、邮件 PDF 拆模块，用现有 3000 行测试兜住行为。这是可维护性投资，不先于上面的安全和工具链。工作量：大。

## 7. 未覆盖项

- 没有安装 Frappe 17 与 ERPNext，因此没有跑 `bench run-parallel-tests`，也不能在浏览器里点登录、请假和工资单。
- 没有对生产站点的权限配置、反向代理日志或 Frappe Cloud 网络策略做验证。报表注入和访客接口的可利用性还取决于谁能打开报表、站点域名是不是 `*.frappehr.com`。
- `yarn audit` 对传递依赖重复计数。严重项已按包合并；高危总数不宜直接当成待修缺陷数。
- 本报告不包含应用代码改动。
