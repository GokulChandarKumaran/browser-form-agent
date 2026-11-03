# Google Harness Extra Credit: Evaluating My Browser Agent

**Student:** Gokul Chandar Kumaran
**Agent tested:** [`browser_form_agent.py`](../browser_form_agent.py), the browser form agent from the earlier homework. It fills out and submits the practice form at https://onyourmark.github.io/browser-agent-form/ with Name `Alex Student`, Email `alex@example.com`, Program `MS Information Systems`, and the agreement checked.
**Model:** `qwen3:8b` through Ollama, driving Chromium through Playwright.

## Step 1: The connection to Google's harness idea

The course separates the model from the harness around it: the model thinks and asks for actions, and the harness does everything else. It puts that as "A decision doesn't change a file. A tool does." My browser agent has the same split. Qwen3 only writes a JSON proposal such as `{"action": "fill_email", "argument": "alex@example.com"}`. The Python program is the harness. It keeps the conversation as state, parses the reply, validates it against an allowed-action list, runs exactly one Playwright action, reads the browser back with `observe_form()`, sends that observation to the model, and enforces a 12-step budget. The course's module on tools and flow (a control loop with a retry limit) matches my loop and budget. Its security module (permissions enforced in software, and retrieved content treated as data) matches my validator and the `BROWSER OBSERVATION` label on page data. Its evaluation module says clear requirements should be checked by deterministic code, which is how my program decides success.

## Step 2: How my program decides success

The model cannot declare success. Choosing `done` is only a request to stop. When the model proposes `done`, Python calls `check_final_success()`, and the run counts as a success only if **all** of these are true in the real browser:

1. The form was submitted. `form_submitted` is set only after a `submit_form` action whose check found the result section visible.
2. The text `FORM SUBMITTED SUCCESSFULLY` is visible on the page.
3. The displayed submitted Name is exactly `Alex Student`.
4. The displayed submitted Email is exactly `alex@example.com`.
5. The displayed submitted Program is exactly `MS Information Systems`.
6. The displayed submitted Agreement is `Checked`.

If any check fails, the program tells the model why and the loop continues. If 12 steps pass without a successful final check, the run stops with `budget_exceeded`. Each action is also checked as it happens: after `fill_name`, for example, Python reads the Name field back and compares it with the expected value. These are exact string comparisons in ordinary Python, the deterministic kind of evaluation the course recommends for clear requirements.

## Step 3: Five runs

I ran the same unchanged program five times in a row on November 2, 2026, with the command `python browser_form_agent.py`. Each run launches a new Chromium browser and opens the form fresh, so every run started from an empty form. The program file had the same SHA-256 hash for all five runs (`D5108C6A…3071B414`). The full terminal output of each run is in this folder.

| Run | Success or Failure | Notes |
|---|---|---|
| 1 | Success | 7 of 12 steps. `open_form` → `fill_name` → `fill_email` → `select_program` → `check_agreement` → `submit_form` → `done`. Every per-step check passed, then the final check passed. Took about 62 seconds, because Ollama loaded the model into memory on the first call. Log: [`run1.log`](run1.log) |
| 2 | Success | Same 7 actions with the same arguments. No parse or validation errors. About 21 seconds. Log: [`run2.log`](run2.log) |
| 3 | Success | Same 7 actions. Final check: "FORM SUBMITTED SUCCESSFULLY is visible and all submitted values are correct." About 21 seconds. Log: [`run3.log`](run3.log) |
| 4 | Success | Same 7 actions. About 21 seconds. Log: [`run4.log`](run4.log) |
| 5 | Success | Same 7 actions. About 22 seconds. Log: [`run5.log`](run5.log) |

**Result: 5 of 5 runs succeeded.** In every run the model's final explanation was the same: "Form submitted successfully with the correct values."

## Step 4: What I learned

The agent was completely consistent: five runs, the same seven actions in the same order, no rejected proposals, and a final check that passed on Python's evidence rather than on the model's word. The main limitation is that these five runs only tested the easy path. The request, the form and the values were identical every time, and the model never sent malformed JSON, an unknown action or a wrong value. So the parser's error branch, the validator's rejections, a failed field check, and the budget limit were never exercised, and five successes say nothing about whether those safety parts work in a live run. Repeating an identical task five times also mostly measures repeatability, not reliability across situations. A stronger evaluation would vary the inputs (a different program, an email with a typo in the goal), break the page on purpose (a missing field, or a submit button that doesn't work), and include a page with hidden instructions, to check that the final success check still refuses a false `done`.

## Files in this folder

- `README.md`: this report
- `run1.log` to `run5.log`: the complete terminal output of each run, including every model reply, parsed action, validation result, browser observation, check result and the printed trace
