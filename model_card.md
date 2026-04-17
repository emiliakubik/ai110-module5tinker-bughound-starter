# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound - "A tiny code analysis agent with built-in safety guardrails"  
**Version:** 0.1.0 (Educational prototype)

**Purpose:** Analyze a Python snippet for code quality, reliability, and maintainability issues, propose a fix using either AI-powered analysis or heuristic rules, and run multi-layered reliability checks (syntax validation, risk assessment) before suggesting whether the fix should be auto-applied. The system implements a 5-step agentic workflow: Plan → Analyze → Act → Test → Reflect.

**Intended users:** Students learning agentic workflows and AI reliability concepts, particularly those exploring how to build safe AI systems with fallback mechanisms, guardrails, and transparent decision-making. Best suited for developers who want to understand the tradeoffs between heuristic rules and LLM-powered analysis in production-like scenarios.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).  
Include what is done by heuristics vs what is done by Gemini (if enabled).


BugHound operates through a 5-step agentic workflow that adapts based on whether Gemini API is available or if it needs to use offline heuristics:

### The 5-Step Workflow:

**1. PLAN** - The agent logs its intention to scan the code and propose fixes. This step is identical in both modes.

**2. ANALYZE** - Detect issues in the code
- **Heuristic mode**: Uses regex patterns to detect:
  - Print statements (Low severity)
  - Bare `except:` blocks (High severity)
  - TODO comments (Medium severity)
- **Gemini mode**: Sends code to the API with a prompt asking for JSON-formatted issue analysis, then validates the response has proper severity levels ("Low", "Medium", "High") and non-empty messages. Falls back to heuristics if API fails, returns invalid JSON, or fails quality checks.

**3. ACT** - Propose a fix for detected issues
- **Heuristic mode**: Applies simple transformations:
  - Replaces `print(` with `logging.info(`
  - Replaces bare `except:` with `except Exception as e:`
  - Adds `import logging` if needed
- **Gemini mode**: Sends the code and issues to the API requesting a full rewrite, strips markdown code fences, validates syntax using Python's `compile()`, and rejects fixes with syntax errors. Falls back to heuristics on errors.

**4. TEST** - Run risk assessment (same for both modes)
- Calculates a risk score (0-100) based on:
  - Issue severity (-40 for High, -20 for Medium, -5 for Low)
  - Structural changes (penalizes if code length drops >50%, return statements removed)
  - **Rewards fixing bare excepts (+5 bonus)** for improved error handling
- Assigns risk level: low (75+), medium (40-74), high (<40)
- Validates fixed code has valid Python syntax (guardrail)

**5. REFLECT** - Decide whether to auto-apply
- Sets `should_autofix: true` only if risk level is "low" (score ≥ 75)
- Otherwise recommends human review
- Same logic for both modes

### Key Safety Features:
- **Triple fallback**: API errors → JSON parsing failures → quality validation → heuristics
- **Syntax validation guardrail**: Rejects any fix that doesn't compile as valid Python
- **Quality checks**: Validates AI output has proper severity levels and messages before accepting
- **Transparent logging**: Every step and decision is logged for debugging
---

## 3) Inputs and outputs

**Inputs:**

- What kind of code snippets did you try?
- What was the “shape” of the input (short scripts, functions, try/except blocks, etc.)?

**Outputs:**

- What types of issues were detected?
- What kinds of fixes were proposed?
- What did the risk report show?

**Inputs:**

I tested BugHound on the following types of code snippets:

1. **print_spam.py** - Simple function with multiple print statements (7 lines)
   - Shape: Single function with conditional logic and multiple print calls
   
2. **flaky_try_except.py** - File I/O with bare exception handling (9 lines)
   - Shape: Function with try/except block using bare `except:`
   
3. **mixed_issues.py** - Multiple issue types in one snippet (9 lines)
   - Shape: Function combining TODO comment, print statement, and bare except
   
4. **cleanish.py** - Well-written code with proper logging (6 lines)
   - Shape: Function using logging module correctly (negative test case)

All inputs were **short functions (6-9 lines)** representing common code patterns students write. The snippets focused on error handling blocks, debugging statements, and incomplete implementations.

**Outputs:**

**Issues Detected:**

| Issue Type | Severity | Example from |
|------------|----------|--------------|
| Print statements | Low | print_spam.py, mixed_issues.py |
| Bare `except:` blocks | High | flaky_try_except.py, mixed_issues.py |
| TODO comments | Medium | mixed_issues.py |
| No issues | N/A | cleanish.py |

**Fixes Proposed:**

- **Print statements** → Replaced with `logging.info()` calls, added `import logging`
- **Bare except** → Changed to `except Exception as e:` with error handling comment
- **Combined fixes** → Applied multiple transformations in correct order (imports first, then replacements)
- **Clean code** → Returned unchanged (no unnecessary modifications)

**Risk Report Examples:**

*High severity issue (bare except):*
- **Score:** 65 (after fix bonus)
- **Level:** Medium
- **Reasons:** "High severity issue detected", "Bare except was fixed - improved error handling"
- **Auto-fix:** No (score < 75)

*Low severity issue (prints only):*
- **Score:** 95
- **Level:** Low  
- **Reasons:** "Low severity issue detected"
- **Auto-fix:** Yes (score ≥ 75)

*Multiple issues (mixed_issues.py):*
- **Score:** ~50-60 (varies by mode)
- **Level:** Medium
- **Reasons:** High + Medium + Low severity issues, structural changes
- **Auto-fix:** No (requires human review due to multiple changes)

**Key Observation:** The risk scorer correctly distinguished between low-risk cosmetic changes (print→logging) and higher-risk error handling modifications (bare except fixes), even though fixing bare excepts is actually a good practice.

---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

- What does the rule check?
- Why might that check matter for safety or correctness?
- What is a false positive this rule could cause?
- What is a false negative this rule could miss?

---

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.



---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

### What Gemini can detect that heuristics miss:

**Gemini's advantages (when working properly):**
- **Nuanced code quality issues**: Variable naming (e.g., `x`, `y` instead of `dividend`, `divisor`), missing docstrings, unclear logic flow
- **Context-aware analysis**: Can understand when print statements are intentional (CLI output) vs debugging artifacts
- **Additional patterns**: Magic numbers, overly complex conditionals, inefficient algorithms
- **Semantic understanding**: Might catch logic errors like mutable default arguments or scope issues

**However:** Gemini's quality depends heavily on prompt engineering and can be inconsistent. With quality validation enabled, poorly-formatted or invalid responses trigger fallback to heuristics.

### What heuristics catch consistently:

**Heuristic mode's strengths:**
- ✅ **100% reliable pattern detection**: Always catches print statements, bare `except:`, and TODO comments
- ✅ **Instant execution**: No API latency or rate limits
- ✅ **Deterministic**: Same input always produces same output
- ✅ **No cost**: Works offline, no API quota consumption

**Limitation:** Only catches the 3 hardcoded patterns - misses everything else.

### How proposed fixes differ:

| Aspect | Heuristic Mode | Gemini Mode |
|--------|---------------|-------------|
| **Approach** | Mechanical string replacement | Context-aware rewrite |
| **Print fix** | `print(` → `logging.info(` | May add proper log levels, format strings |
| **Bare except** | `except:` → `except Exception as e:` + comment | May add specific exception types, proper error handling |
| **Code style** | Minimal changes only | May refactor broader code structure |
| **Reliability** | Predictable but shallow | Smarter but can hallucinate or break syntax |
| **Safety** | Syntax validation guardrail catches errors | Same guardrail + quality validation |

**Example difference on mixed_issues.py:**

*Heuristic fix:*
```python
import logging

# TODO: Replace this with real input validation
def compute_ratio(x, y):
    logging.info("computing ratio...")
    try:
        return x / y
    except Exception as e:
        # [BugHound] log or handle the error
        return 0
---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

### Scenario: Function Signature Changes

**When to refuse auto-fix:** Any time the fixed code modifies a function's signature (parameter names, parameter count, or return type changes).

**Why this matters:** Changing a function's API breaks all calling code. Even if the fix improves the function internally, it requires coordinated updates across the entire codebase. This is too risky for automatic application.

**Example that should block auto-fix:**

```python
# Original
def calculate(x, y):
    return x / y

# Fixed (Gemini adds type hints and validation)
def calculate(dividend: float, divisor: float) -> float:
    if divisor == 0:
        raise ValueError("Cannot divide by zero")
    return dividend / divisor


---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

### Improvement: Line-Level Change Percentage Guardrail

**Problem:** BugHound currently has no protection against overly aggressive rewrites. Gemini might rewrite 80% of a function when only fixing a single print statement, introducing unnecessary risk.

**Solution:** Add a simple guardrail that measures what percentage of lines actually changed and blocks auto-fix if modifications exceed 30%.

**Implementation:**
```python
def _calculate_change_percentage(original: str, fixed: str) -> float:
    """Calculate what percentage of lines were modified."""
    import difflib
    
    orig_lines = original.strip().splitlines()
    fixed_lines = fixed.strip().splitlines()
    
    diff = difflib.unified_diff(orig_lines, fixed_lines, lineterm='')
    changed_lines = sum(1 for line in diff if line.startswith('+') or line.startswith('-'))
    
    total_lines = max(len(orig_lines), len(fixed_lines))
    return (changed_lines / total_lines) * 100 if total_lines > 0 else 0

# Add to risk_assessor.py
change_pct = _calculate_change_percentage(original_code, fixed_code)
if change_pct > 30:
    score -= 25
    reasons.append(f"Too many lines changed ({change_pct:.0f}%) - may be over-rewriting.")
