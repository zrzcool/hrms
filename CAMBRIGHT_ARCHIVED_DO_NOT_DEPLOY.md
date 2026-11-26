# ⚠️ ARCHIVED UPSTREAM MIRROR — DO NOT DEPLOY TO PRODUCTION

本仓库（`HR.Portal`）是上游开源 `frappe/hrms`（`develop` 分支）的零改动镜像副本，**与 Cambright One 生产环境存在底层架构冲突，严禁直接构建或部署至线上环境**。

## 为什么不能部署本仓库
1. **Employee 控制器重写**：Cambright One 在 `cambright_custom/overrides/employee.py` 中将 `Employee` 重写为继承 `frappe.model.document.Document`（绕过 HRMS `EmployeeMaster`），并引入了自定义离职交接态 `"Resign"`。
2. **权限隔离策略不同**：Cambright One 禁止向普通员工自动授予原生 `Employee` / `Salary Slip` DocPerm 读取权限，本仓内置的 `frontend/`（Ionic PWA）与 `roster/` 强依赖原生 HRMS 权限与 `Salary Slip` 表，直接启用会全面报错或破坏权限边界。
3. **马来西亚算薪体系独立**：Cambright 生产算薪与报税运行在自定义 `CB Payroll Entry`、`CB Statutory Rate`、`CB PCB Filing` 上，而非本仓的 `Salary Slip`。

## Cambright 真实人事与员工自助入口
- **全员自助中心（请假、报销、工资单 PDF、资产、通讯录、公告、培训、资料更正）**：统一使用各职能端挂载的同源 `/staff/`（`Staff Hub` + `<cb-staff-bar>`，后端对应 `cambright_custom.app_api.v1.staff_hub.*`）。
- **HR 专员管理工作台（入离职、算薪、法定缴纳、考勤、排班）**：统一使用 Cambright One 后台 `https://one.cambright.edu.my/app/hr-dashboard`（后端对应 `cambright_custom/hr_malaysia/` 与 `services/hr_malaysia/`）。
