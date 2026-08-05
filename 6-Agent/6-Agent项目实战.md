# 第六阶段：Agent 项目实战

> **目标：** 设计并实现一个完整的 Agent 项目，经历从需求分析到可运行代码的全过程。
> **学完本阶段，你应该具备独立开发 Agent 项目的能力。**

---

## 1. 项目选型：AI 代码审查 Agent

### 1.1 为什么选这个项目

```
选择 AI Code Review Agent 的原因：

1. 实用性强：代码审查是每个开发团队的刚需
2. 技术覆盖全面：
   - Tool Calling（调用代码分析工具）
   - Memory（记住历史审查记录）
   - RAG（检索代码规范文档）
   - 多 Agent（安全/性能/风格三个维度）
3. 复杂度适中：不太简单（有学习价值），不太复杂（能实现）
4. 可扩展：后续可以加更多审查维度
```

### 1.2 系统架构

```
┌──────────────────────────────────────────────────────────────┐
│                  AI Code Review Agent 系统架构                 │
│                                                              │
│                      用户提交代码                              │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Supervisor Agent                      │    │
│  │  职责：接收代码 → 分配审查任务 → 汇总结果 → 生成报告     │    │
│  └──────────────────────┬───────────────────────────────┘    │
│                         │                                    │
│           ┌─────────────┼─────────────┐                     │
│           ▼             ▼             ▼                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Security     │ │ Performance  │ │ Style        │        │
│  │ Reviewer     │ │ Reviewer     │ │ Reviewer     │        │
│  │              │ │              │ │              │        │
│  │ 工具：       │ │ 工具：       │ │ 工具：       │        │
│  │ - AST分析    │ │ - AST分析    │ │ - AST分析    │        │
│  │ - 规则匹配   │ │ - 复杂度计算 │ │ - 风格检查   │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
│           │             │             │                     │
│           └─────────────┼─────────────┘                     │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Report Generator                    │    │
│  │  职责：综合三个审查员的结果，生成结构化审查报告           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Knowledge Base (RAG)                  │    │
│  │  存储内容：代码规范、安全最佳实践、团队风格指南           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Memory System                         │    │
│  │  短期：当前审查的上下文                                  │    │
│  │  长期：历史审查记录、常见问题模式                         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 1.3 技术选型

| 组件 | 选择 | 理由 |
|------|------|------|
| LLM | OpenAI GPT-4o | 代码理解能力强 |
| Agent 框架 | 纯 Python | 学习阶段不依赖框架，理解底层原理 |
| AST 解析 | Python `ast` 模块 | 标准库，无需额外依赖 |
| 向量数据库 | Chroma | 最简单，适合原型 |
| Memory | 内存字典 | 简化实现，生产用 Redis |

---

## 2. 模块拆分

```
项目结构：

code_review_agent/
├── main.py              # 入口
├── agent/
│   ├── supervisor.py    # Supervisor Agent
│   ├── security.py      # 安全审查 Agent
│   ├── performance.py   # 性能审查 Agent
│   └── style.py         # 风格审查 Agent
├── tools/
│   ├── ast_analyzer.py  # AST 代码分析工具
│   ├── rule_checker.py  # 规则匹配工具
│   └── metrics.py       # 代码度量工具
├── memory/
│   └── manager.py       # Memory 管理
├── knowledge/
│   └── base.py          # RAG 知识库
└── models/
    └── schemas.py       # 数据模型
```

---

## 3. 完整实现

### 3.1 数据模型

```python
"""
数据模型：定义审查结果的标准化结构
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class Severity(Enum):
    HIGH = "高"
    MEDIUM = "中"
    LOW = "低"
    INFO = "建议"


class ReviewType(Enum):
    SECURITY = "安全"
    PERFORMANCE = "性能"
    STYLE = "风格"


@dataclass
class ReviewIssue:
    """单个审查问题"""
    severity: Severity          # 严重程度
    review_type: ReviewType     # 审查类型
    title: str                  # 问题标题
    description: str            # 问题描述
    line_number: Optional[int]  # 行号
    suggestion: str             # 修复建议
    code_snippet: Optional[str] = None  # 相关代码片段


@dataclass
class ReviewReport:
    """审查报告"""
    code_hash: str                      # 代码哈希（标识）
    total_issues: int = 0               # 总问题数
    high_issues: int = 0                # 高严重度问题数
    medium_issues: int = 0              # 中严重度问题数
    low_issues: int = 0                 # 低严重度问题数
    issues: list = field(default_factory=list)  # 所有问题列表
    summary: str = ""                   # 总结
    score: int = 100                    # 代码质量评分（0-100）


@dataclass
class CodeFile:
    """待审查的代码文件"""
    filename: str
    content: str
    language: str = "python"
```

### 3.2 代码分析工具

```python
"""
代码分析工具：用 AST 解析 Python 代码，提取结构信息
"""

import ast
import hashlib
from typing import Dict, List, Any


class ASTAnalyzer:
    """AST 代码分析器——Agent 的核心工具"""

    def analyze(self, code: str) -> Dict[str, Any]:
        """分析代码，返回结构化信息"""
        try:
            tree = ast.parse(code)
        except SyntaxError as e:
            return {"error": f"语法错误：{e}"}

        return {
            "functions": self._get_functions(tree),
            "classes": self._get_classes(tree),
            "imports": self._get_imports(tree),
            "complexity": self._estimate_complexity(tree),
            "lines": len(code.split("\n")),
            "code_hash": hashlib.md5(code.encode()).hexdigest()[:8],
        }

    def _get_functions(self, tree: ast.AST) -> List[Dict]:
        """提取所有函数信息"""
        functions = []
        for node in ast.walk(tree):
            if isinstance(node, ast.FunctionDef):
                args = [a.arg for a in node.args.args]
                functions.append({
                    "name": node.name,
                    "line": node.lineno,
                    "args": args,
                    "body_lines": len(node.body),
                    "is_async": isinstance(node, ast.AsyncFunctionDef),
                    "decorators": [
                        d.id if isinstance(d, ast.Name) else str(d)
                        for d in node.decorator_list
                    ],
                    "has_return": any(
                        isinstance(n, ast.Return) and n.value is not None
                        for n in ast.walk(node)
                    ),
                })
        return functions

    def _get_classes(self, tree: ast.AST) -> List[Dict]:
        """提取所有类信息"""
        classes = []
        for node in ast.walk(tree):
            if isinstance(node, ast.ClassDef):
                methods = [
                    n.name for n in node.body
                    if isinstance(n, ast.FunctionDef)
                ]
                classes.append({
                    "name": node.name,
                    "line": node.lineno,
                    "methods": methods,
                    "method_count": len(methods),
                })
        return classes

    def _get_imports(self, tree: ast.AST) -> List[str]:
        """提取所有 import"""
        imports = []
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                imports.extend(a.name for a in node.names)
            elif isinstance(node, ast.ImportFrom):
                module = node.module or ""
                imports.extend(
                    f"{module}.{a.name}" for a in node.names
                )
        return imports

    def _estimate_complexity(self, tree: ast.AST) -> Dict:
        """估算代码复杂度"""
        complexity = {
            "if_count": 0,
            "for_count": 0,
            "while_count": 0,
            "try_count": 0,
            "nested_depth": 0,
        }

        def walk_depth(node, depth=0):
            max_depth = depth
            for child in ast.iter_child_nodes(node):
                if isinstance(child, (ast.If, ast.For, ast.While, ast.Try)):
                    child_depth = walk_depth(child, depth + 1)
                    max_depth = max(max_depth, child_depth)
                else:
                    child_depth = walk_depth(child, depth)
                    max_depth = max(max_depth, child_depth)

                if isinstance(child, ast.If):
                    complexity["if_count"] += 1
                elif isinstance(child, ast.For):
                    complexity["for_count"] += 1
                elif isinstance(child, ast.While):
                    complexity["while_count"] += 1
                elif isinstance(child, ast.Try):
                    complexity["try_count"] += 1

            return max_depth

        complexity["nested_depth"] = walk_depth(tree)
        return complexity

    def find_patterns(self, code: str) -> List[Dict]:
        """查找常见问题模式"""
        issues = []
        lines = code.split("\n")

        for i, line in enumerate(lines, 1):
            stripped = line.strip()

            # 检查 eval 使用
            if "eval(" in stripped and not stripped.startswith("#"):
                issues.append({
                    "pattern": "eval_usage",
                    "line": i,
                    "code": stripped,
                    "risk": "high",
                    "description": "使用了 eval()，存在安全风险",
                })

            # 检查硬编码密码
            if any(kw in stripped.lower() for kw in ["password", "secret", "api_key"]):
                if "=" in stripped and not stripped.startswith("#"):
                    if '"' in stripped or "'" in stripped:
                        issues.append({
                            "pattern": "hardcoded_secret",
                            "line": i,
                            "code": stripped,
                            "risk": "high",
                            "description": "可能存在硬编码的敏感信息",
                        })

            # 检查 SQL 拼接
            if "f\"" in stripped and ("SELECT" in stripped.upper() or "INSERT" in stripped.upper()):
                issues.append({
                    "pattern": "sql_injection",
                    "line": i,
                    "code": stripped,
                    "risk": "high",
                    "description": "可能存在 SQL 注入风险（字符串拼接 SQL）",
                })

            # 检查 except: pass（静默吞掉异常）
            if stripped == "except:" or stripped == "except Exception:":
                next_line = lines[i].strip() if i < len(lines) else ""
                if next_line == "pass":
                    issues.append({
                        "pattern": "silent_exception",
                        "line": i,
                        "code": stripped,
                        "risk": "medium",
                        "description": "静默捕获异常，可能隐藏错误",
                    })

            # 检查过长的行
            if len(stripped) > 120:
                issues.append({
                    "pattern": "long_line",
                    "line": i,
                    "code": stripped[:50] + "...",
                    "risk": "low",
                    "description": f"行过长（{len(stripped)} 字符）",
                })

        return issues
```

### 3.3 完整的 Code Review Agent

```python
"""
AI Code Review Agent - 完整实现
"""

import json
from models.schemas import (
    ReviewIssue, ReviewReport, ReviewType,
    Severity, CodeFile
)
from tools.ast_analyzer import ASTAnalyzer


class CodeReviewAgent:
    """
    AI 代码审查 Agent

    架构：Supervisor 模式
    - Supervisor 接收代码，分配给专业审查员
    - 三个审查员从不同维度审查
    - Supervisor 综合生成报告
    """

    def __init__(self, llm_func=None):
        self.llm_func = llm_func
        self.ast_analyzer = ASTAnalyzer()
        self.review_history = []  # Memory：历史审查记录

    def review(self, code: str, filename: str = "main.py") -> ReviewReport:
        """
        审查代码的主入口

        流程：
        1. AST 分析（工具调用）
        2. 模式检测（工具调用）
        3. 安全审查（LLM）
        4. 性能审查（LLM）
        5. 风格审查（LLM）
        6. 生成综合报告
        """
        code_file = CodeFile(filename=filename, content=code)

        # Step 1: AST 分析
        ast_info = self.ast_analyzer.analyze(code)

        # Step 2: 模式检测
        patterns = self.ast_analyzer.find_patterns(code)

        # Step 3: 三个维度审查
        all_issues = []

        # 安全审查（基于 AST + 模式检测结果）
        security_issues = self._review_security(code_file, ast_info, patterns)
        all_issues.extend(security_issues)

        # 性能审查
        performance_issues = self._review_performance(code_file, ast_info)
        all_issues.extend(performance_issues)

        # 风格审查
        style_issues = self._review_style(code_file, ast_info)
        all_issues.extend(style_issues)

        # Step 4: 生成报告
        report = self._generate_report(code_file, all_issues)

        # Step 5: 保存到 Memory
        self.review_history.append({
            "filename": filename,
            "code_hash": ast_info.get("code_hash", ""),
            "issues_count": len(all_issues),
            "score": report.score,
        })

        return report

    def _review_security(self, code_file: CodeFile, ast_info: dict,
                         patterns: list) -> list:
        """安全审查"""
        issues = []

        # 基于模式检测的安全问题
        for p in patterns:
            if p["risk"] == "high":
                issues.append(ReviewIssue(
                    severity=Severity.HIGH,
                    review_type=ReviewType.SECURITY,
                    title=p["description"],
                    description=f"在第 {p['line']} 行发现潜在安全问题",
                    line_number=p["line"],
                    suggestion=self._get_security_suggestion(p["pattern"]),
                    code_snippet=p["code"],
                ))

        # 检查 eval 使用
        for p in patterns:
            if p["pattern"] == "eval_usage":
                issues.append(ReviewIssue(
                    severity=Severity.HIGH,
                    review_type=ReviewType.SECURITY,
                    title="使用了 eval()",
                    description="eval() 可能执行任意代码，存在安全风险",
                    line_number=p["line"],
                    suggestion="使用 ast.literal_eval() 或 json.loads() 替代",
                    code_snippet=p["code"],
                ))

        return issues

    def _review_performance(self, code_file: CodeFile, ast_info: dict) -> list:
        """性能审查"""
        issues = []

        # 检查函数长度
        for func in ast_info.get("functions", []):
            if func["body_lines"] > 30:
                issues.append(ReviewIssue(
                    severity=Severity.MEDIUM,
                    review_type=ReviewType.PERFORMANCE,
                    title=f"函数 '{func['name']}' 过长",
                    description=f"函数有 {func['body_lines']} 行，建议拆分",
                    line_number=func["line"],
                    suggestion="将长函数拆分为多个职责单一的小函数",
                ))

        # 检查嵌套深度
        complexity = ast_info.get("complexity", {})
        if complexity.get("nested_depth", 0) > 3:
            issues.append(ReviewIssue(
                severity=Severity.MEDIUM,
                review_type=ReviewType.PERFORMANCE,
                title="嵌套层级过深",
                description=f"嵌套深度为 {complexity['nested_depth']}，建议不超过 3",
                line_number=None,
                suggestion="使用卫语句（early return）减少嵌套",
            ))

        # 检查循环内是否有潜在的 O(n²) 操作
        for func in ast_info.get("functions", []):
            lines = code_file.content.split("\n")
            for i, line in enumerate(lines):
                if "for " in line and " in " in line:
                    # 检查是否有嵌套循环
                    indent = len(line) - len(line.lstrip())
                    for j in range(i + 1, min(i + 20, len(lines))):
                        next_line = lines[j]
                        next_indent = len(next_line) - len(next_line.lstrip())
                        if next_indent > indent and "for " in next_line:
                            issues.append(ReviewIssue(
                                severity=Severity.MEDIUM,
                                review_type=ReviewType.PERFORMANCE,
                                title="检测到嵌套循环",
                                description="嵌套循环可能导致 O(n²) 复杂度",
                                line_number=i + 1,
                                suggestion="考虑使用字典/集合优化查找，或使用 itertools",
                            ))
                            break

        return issues

    def _review_style(self, code_file: CodeFile, ast_info: dict) -> list:
        """风格审查"""
        issues = []
        lines = code_file.content.split("\n")

        # 检查是否有文档字符串
        for func in ast_info.get("functions", []):
            if not func.get("decorators"):  # 排除装饰器函数
                if func["body_lines"] > 5:  # 只检查较长的函数
                    # 简化检查：看函数体第一行是否是字符串
                    start = func["line"]  # 1-indexed
                    if start < len(lines):
                        first_body = lines[start].strip()
                        if not (first_body.startswith('"""') or first_body.startswith("'''")):
                            issues.append(ReviewIssue(
                                severity=Severity.LOW,
                                review_type=ReviewType.STYLE,
                                title=f"函数 '{func['name']}' 缺少文档字符串",
                                description="建议为较长函数添加文档字符串",
                                line_number=func["line"],
                                suggestion='添加 """文档字符串""" 说明函数用途',
                            ))

        # 检查过长的行
        for i, line in enumerate(lines, 1):
            if len(line) > 120:
                issues.append(ReviewIssue(
                    severity=Severity.INFO,
                    review_type=ReviewType.STYLE,
                    title=f"第 {i} 行过长（{len(line)} 字符）",
                    description="建议每行不超过 120 字符",
                    line_number=i,
                    suggestion="拆分长行或使用括号隐式续行",
                ))

        # 检查是否有 TODO/FIXME
        for i, line in enumerate(lines, 1):
            stripped = line.strip()
            if stripped.startswith("#") and ("TODO" in stripped or "FIXME" in stripped):
                issues.append(ReviewIssue(
                    severity=Severity.INFO,
                    review_type=ReviewType.STYLE,
                    title=f"第 {i} 行有 TODO/FIXME 标记",
                    description=f"待处理标记：{stripped}",
                    line_number=i,
                    suggestion="及时处理 TODO/FIXME 标记",
                ))

        return issues

    def _get_security_suggestion(self, pattern: str) -> str:
        """根据模式给出修复建议"""
        suggestions = {
            "eval_usage": "使用 ast.literal_eval() 或 json.loads() 替代 eval()",
            "hardcoded_secret": "使用环境变量或配置文件管理敏感信息",
            "sql_injection": "使用参数化查询（如 cursor.execute(sql, params)）",
            "silent_exception": "至少记录异常日志，不要静默吞掉",
        }
        return suggestions.get(pattern, "请检查相关代码")

    def _generate_report(self, code_file: CodeFile, issues: list) -> ReviewReport:
        """生成审查报告"""
        high = sum(1 for i in issues if i.severity == Severity.HIGH)
        medium = sum(1 for i in issues if i.severity == Severity.MEDIUM)
        low = sum(1 for i in issues if i.severity in (Severity.LOW, Severity.INFO))

        # 计算评分
        score = 100
        score -= high * 15  # 每个高严重度问题扣 15 分
        score -= medium * 5  # 每个中严重度问题扣 5 分
        score -= low * 1    # 每个低严重度问题扣 1 分
        score = max(0, score)

        # 生成总结
        if score >= 90:
            summary = "代码质量优秀，只有少量小问题"
        elif score >= 70:
            summary = "代码质量良好，有一些需要改进的地方"
        elif score >= 50:
            summary = "代码质量一般，有较多问题需要修复"
        else:
            summary = "代码质量较差，有严重问题需要立即修复"

        return ReviewReport(
            code_hash=code_file.content[:8],
            total_issues=len(issues),
            high_issues=high,
            medium_issues=medium,
            low_issues=low,
            issues=issues,
            summary=summary,
            score=score,
        )

    def format_report(self, report: ReviewReport) -> str:
        """格式化输出报告"""
        lines = [
            f"\n{'='*60}",
            f"  代码审查报告",
            f"{'='*60}",
            f"",
            f"  总体评分：{report.score}/100",
            f"  总体评价：{report.summary}",
            f"",
            f"  问题统计：",
            f"    高严重度：{report.high_issues}",
            f"    中严重度：{report.medium_issues}",
            f"    低严重度：{report.low_issues}",
            f"",
        ]

        if report.issues:
            lines.append("  问题详情：")
            lines.append("-" * 60)

            for i, issue in enumerate(report.issues, 1):
                lines.append(f"\n  [{i}] [{issue.severity.value}] [{issue.review_type.value}] {issue.title}")
                if issue.line_number:
                    lines.append(f"      位置：第 {issue.line_number} 行")
                lines.append(f"      说明：{issue.description}")
                lines.append(f"      建议：{issue.suggestion}")
                if issue.code_snippet:
                    lines.append(f"      代码：{issue.code_snippet}")

        lines.append(f"\n{'='*60}")
        return "\n".join(lines)


# ============================================================
# 运行
# ============================================================

if __name__ == "__main__":
    # 待审查的代码
    sample_code = """
import os

def get_user_orders(user_id, db_connection):
    # TODO: 添加缓存
    password = "admin123"
    query = f"SELECT * FROM orders WHERE user_id = {user_id}"
    orders = db_connection.execute(query)
    result = []
    for order in orders:
        items_query = f"SELECT * FROM items WHERE order_id = {order.id}"
        items = db_connection.execute(items_query)
        for item in items:
            for detail in item.details:
                for sub in detail.sub_items:
                    result.append({
                        "order": order,
                        "item": item,
                        "detail": detail,
                        "sub": sub
                    })
    eval("print('done')")
    return result

def process_payment(order_id, amount, user_info):
    api_key = "sk-1234567890abcdef"
    very_long_line_that_exceeds_the_one_hundred_and_twenty_character_limit_and_should_be_split_into_multiple_lines_for_readability = True
    try:
        result = charge_credit_card(order_id, amount, user_info)
    except:
        pass
    return result
"""

    # 创建 Agent 并执行审查
    agent = CodeReviewAgent()
    report = agent.review(sample_code, "order_service.py")

    # 输出报告
    print(agent.format_report(report))
```

### 3.4 运行结果示例

```
============================================================
  代码审查报告
============================================================

  总体评分：25/100
  总体评价：代码质量较差，有严重问题需要立即修复

  问题统计：
    高严重度：3
    中严重度：3
    低严重度：2

  问题详情：
------------------------------------------------------------

  [1] [高] [安全] 可能存在 SQL 注入风险（字符串拼接 SQL）
      位置：第 6 行
      说明：在第 6 行发现潜在安全问题
      建议：使用参数化查询（如 cursor.execute(sql, params)）
      代码：query = f"SELECT * FROM orders WHERE user_id = {user_id}"

  [2] [高] [安全] 可能存在硬编码的敏感信息
      位置：第 4 行
      说明：在第 4 行发现潜在安全问题
      建议：使用环境变量或配置文件管理敏感信息

  [3] [高] [安全] 使用了 eval()
      位置：第 20 行
      说明：eval() 可能执行任意代码，存在安全风险
      建议：使用 ast.literal_eval() 或 json.loads() 替代

  [4] [中] [性能] 嵌套层级过深
      说明：嵌套深度为 4，建议不超过 3
      建议：使用卫语句（early return）减少嵌套

  [5] [中] [性能] 检测到嵌套循环
      说明：嵌套循环可能导致 O(n²) 复杂度
      建议：考虑使用字典/集合优化查找
  ...
```

---

## 4. 技术选型理由

```
1. 为什么用 AST 而不是正则分析代码？
   → AST 是结构化的代码表示，能精确识别函数、类、参数、调用关系
   → 正则表达式太脆弱，容易被注释或字符串中的代码干扰

2. 为什么不用 LangChain？
   → 学习阶段手写能更好理解原理
   → 这个项目的流程相对固定，不需要框架的灵活性
   → 生产环境可以用 LangGraph 重构

3. 为什么评分要加权？
   → 安全问题的严重程度远高于风格问题
   → 加权评分让用户一眼看出代码的紧迫程度

4. 为什么需要 Memory（审查历史）？
   → 可以跟踪代码质量的长期变化
   → 可以识别反复出现的问题模式
   → 可以对比修改前后的改进情况
```

---

## 5. 扩展方向

```
1. 接入真实 LLM（把 mock 替换成 GPT-4o / Claude）
2. 支持多种语言（Java 的 JavaParser / JS 的 TypeScript Compiler API）
3. 添加 RAG（检索代码规范文档）
4. 集成 Git（自动审查 PR 中的变更代码）
5. 添加 Web 界面（FastAPI + React）
6. 接入 LangSmith（追踪 Agent 执行过程）
```

---

## 👉 这一阶段你应该掌握的能力

1. **能设计一个完整的 Agent 项目**：从需求分析到系统架构到模块拆分
2. **能实现代码分析工具**：AST 解析、模式检测、复杂度估算
3. **能实现多 Agent 协作**：Supervisor + 多个专业审查员
4. **能生成结构化的输出**：审查报告、评分、修复建议
5. **能做技术选型决策**：根据场景选择合适的工具和架构

---

> **下一阶段预告：** 第七阶段将聚焦避坑和架构思维——Agent 的常见问题、性能优化、可扩展系统设计。
