# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

BugHound is a lightweight agentic tool designed to help beginner-to-intermediate Python developers catch common code quality and reliability issues before they reach production. It is intentionally minimal so the guardrail logic and agentic loop are easy to study, modify, and extend. The system is built for educational use inside AI engineering courses where students learn how multi-step AI workflows make decisions and self-correct.

---

## 2) How does it work?

BugHound runs a five-step agentic loop: it first **plans** by logging its intent, then **analyzes** the code for issues either via pattern-matching heuristics or a Gemini LLM call, then **acts** by rewriting the code to address those issues, then **tests** the fix by running it through the risk assessor, and finally **reflects** by deciding whether the fix is safe enough to auto-apply. Heuristics handle offline mode or when Gemini fails — they catch bare `except:` blocks, `print` statements, and TODO comments using simple regex and string checks. Gemini mode sends the code to the API with a structured prompt and parses the JSON response, enabling it to catch a wider range of issues such as mutable default arguments, None-safety violations, and logic errors that heuristics would miss.

---

## 3) Inputs and outputs

**Inputs:**

- Short Python functions and scripts, typically 10–40 lines, submitted as plain text through the Streamlit UI or directly via the agent's `run()` method.
- Tested shapes included: bare `except:` blocks, functions with `print` statements mixed with business logic, TODO-heavy scaffolding code, and scripts with missing `None` checks on dictionary lookups.

**Outputs:**

- Issue lists as JSON arrays with `type`, `severity`, `line`, `msg`, and `fix_hint` fields identifying what went wrong and where.
- A rewritten version of the input code with targeted fixes applied — for example, `except:` narrowed to `except Exception as e:` and `print(...)` replaced with `logging.info(...)`.
- A risk report containing a numeric score (0–100), a level (`low` / `medium` / `high`), a list of reasons for score deductions, and a `should_autofix` boolean that gates automatic application of the fix.

---

## 4) Reliability and safety rules

**Rule 1 — Return statement removal (`score -= 30`):** This rule checks whether `return` appears in the original code but not in the fixed code, which would mean the fixer silently dropped a return value. This matters because removing a return statement changes the function's output from the caller's perspective, which is a silent correctness regression — the code still runs but produces `None` instead of the intended value. A false positive occurs when the original `return` was inside a dead code branch that the fixer legitimately removed; a false negative occurs when the fixer moves a `return` inside a newly added `try/except` block, making it unreachable under certain error conditions.

**Rule 2 — New try/except blocks introduced (`score -= 15 per block`):** This rule penalizes the fixer for adding `try:` blocks that did not exist in the original, because adding exception handling is a structural change that can suppress new errors rather than fix existing ones. It matters for safety because a fixer that wraps side-effectful code in a silent `try/except` changes the program's failure behavior — errors that should propagate now disappear. A false positive happens when the fixer legitimately adds a `try/except` around a known-risky operation like file I/O; a false negative happens when the fixer restructures existing exception handling without adding a new `try:` keyword but still changes the scope of what is caught.

---

## 5) Observed failure modes

**Failure 1 — Over-editing with the heuristic fixer:** When code contained both a bare `except:` and a `print` statement, the heuristic fixer prepended `import logging` and replaced every `print(` globally — including `print(` calls inside string literals or comments — which produced invalid or unintended code. The fix was technically mechanical but not semantically aware, so it changed lines it was never supposed to touch.

**Failure 2 — Gemini returning unparseable output:** When the code snippet was very short (2–3 lines with no real issues), Gemini occasionally returned a JSON object instead of a JSON array, or wrapped the array in a markdown code fence despite the system prompt explicitly forbidding it. The agent's fallback to heuristics handled this gracefully, but it meant the LLM's richer analysis was silently discarded without any indication to the user that the model's output was malformed.

---

## 6) Heuristic vs Gemini comparison

Heuristics reliably caught the same three patterns — bare `except:`, `print` statements, and TODO comments — every time, regardless of code shape, because they rely on string matching rather than interpretation. Gemini detected a broader range of issues, including mutable default arguments (`def f(x=[]):`), missing `None` checks after `.get()` calls, and unused imports, which heuristics completely missed. However, Gemini's proposed fixes sometimes changed more code than necessary — for example, reformatting entire functions or renaming variables — while the heuristic fixer made only the narrowest mechanical change, making its output easier to audit even if less complete.

---

## 7) Human-in-the-loop decision

If the fixed code introduces a new `import` statement that was not present in the original — especially `os`, `subprocess`, or `sys` — BugHound should refuse to auto-apply and require human review, because importing these modules enables file system access, shell execution, or process control that could introduce a security risk the original code never had. This trigger would be implemented in `risk_assessor.py` as an additional structural check, keeping the logic centralized alongside the other guardrail rules. The tool should display: *"Fix adds a new system-level import. Auto-apply blocked — please review the rewritten code before proceeding."*

---

## 8) Improvement idea

Add a **line-level diff check** to the risk assessor that counts how many lines actually changed between the original and fixed code, and penalizes the score if more than 30% of lines were modified — because a targeted fix should touch only the lines directly related to the detected issues. This would catch the failure mode where Gemini rewrites entire functions when only one bare `except:` needed narrowing, without requiring any changes to the prompt or the LLM call. The check is low-complexity to implement (a simple `difflib.ndiff` line count) and would make the `should_autofix` gate meaningfully stricter for over-eager fixes.
