# SWE-agent Taxonomy Classification Report

This document provides a comprehensive multi-label classification of the SWE-agent according to the provided taxonomy framework spanning agent architecture, cognitive strategies, memory systems, and tool capabilities.

---

## Agent Architecture

**Labels:** SINGLE-AGENT

**Justification:** SWE-agent operates as a single autonomous agent that works independently on software engineering tasks. The core implementation in `sweagent/agent/agents.py` shows the `DefaultAgent` class executing a single agent loop without multi-agent coordination, delegation, or interaction with other agents. While the `AskColleagues` action sampler in `sweagent/agent/action_sampler.py` samples multiple completions from the same model to simulate colleague opinions, this is not true multi-agent coordination but rather a self-consistency technique within a single agent's decision-making process.

---

## Agent Workflow

**Labels:** ITERATIVE, SELF-IMPROVING

**Justification:** 

1. **ITERATIVE:** The agent implements a continuous thought-action-observation loop as evidenced in `sweagent/agent/agents.py`. The `step()` method (line 1235-1263) and `run()` method (line 1265-1294) show the agent executes actions, observes results, and uses feedback to plan the next step in a repeating cycle (`while not step_output.done: step_output = self.step()`). The ThoughtActionParser in `sweagent/tools/parsing.py` explicitly parses model output into thoughts and actions, implementing the classic ReAct-style iterative workflow.

2. **SELF-IMPROVING:** The agent demonstrates self-improvement through its retry mechanism with the `RetryAgent` class (line 257-441 in `agents.py`) and reviewer system (`sweagent/agent/reviewer.py`). The `RetryAgent` can retry solving issues multiple times, learning from previous failed attempts. The `forward_with_handling()` method (line 1062-1218) shows error handling with requerying capabilities—when the agent makes mistakes (format errors, blocked actions, bash syntax errors), it receives feedback and attempts to correct itself, maintaining a record of these mistakes in the trajectory for learning.

---

## Agent Planning

**Labels:** DIRECT-EXECUTION, REPLANNING, INTERACTIVE

**Justification:**

1. **DIRECT-EXECUTION:** The agent primarily operates in direct execution mode without an explicit hierarchical planning phase. As shown in `sweagent/agent/agents.py`, the `forward()` method (line 1006-1060) queries the model and immediately executes the returned action without creating a formal multi-step plan or exploring alternatives upfront. The thought-action loop is reactive rather than planning multiple steps ahead.

2. **REPLANNING:** The agent dynamically adjusts its approach when conditions change through error handling and retry mechanisms. The `forward_with_handling()` method (line 1062-1218) demonstrates replanning—when actions fail or produce errors, the agent requeries the model with error feedback to generate corrected actions. The `RetryAgent` class also enables replanning at a higher level by allowing complete retries of the solution approach when submissions are rejected by the reviewer.

3. **INTERACTIVE:** The agent supports interactive planning with human input through the `HumanModel` and `HumanThoughtModel` classes in `sweagent/agent/models.py` (line 466-534). The human mode allows users to provide thoughts and actions at each step, enabling human-in-the-loop planning and decision-making. This is configured via the human mode configurations in `config/human/`.

---

## Agent Reasoning

**Labels:** REACT, CHAIN-OF-THOUGHT, SELF-EVALUATION, SELF-REFINE, RETRIEVAL-AUGMENTED-PLANNING

**Justification:**

1. **REACT:** The agent implements the ReAct (Reasoning and Acting) pattern as its core reasoning strategy. The `ThoughtActionParser` in `sweagent/tools/parsing.py` explicitly parses model responses into separate thought and action components. The agent loop alternates between reasoning (thought generation), acting (command execution), and observing (collecting feedback), as seen in the thought-action-observation cycle in `agents.py` line 1045-1050 and the trajectory structure that records thought, action, and observation for each step.

2. **CHAIN-OF-THOUGHT:** The agent employs chain-of-thought reasoning by generating explicit intermediate reasoning steps before each action. The `ThoughtActionParser` expects the model to provide a "thought" explaining its reasoning before the action, and the system template in `config/default.yaml` instructs "Your thinking should be thorough and so it's fine if it's very long," encouraging detailed step-by-step reasoning.

3. **SELF-EVALUATION:** The agent includes self-evaluation capabilities through the reviewer system in `sweagent/agent/reviewer.py`. The `ScoreRetryLoop` and `ChooserRetryLoop` classes evaluate the quality of generated solutions, scoring submissions and selecting the best among multiple attempts. The `BinaryTrajectoryComparison` action sampler (line 96-237 in `action_sampler.py`) also implements self-evaluation by comparing multiple trajectory options and selecting the most promising one.

4. **SELF-REFINE:** The agent demonstrates self-refinement through its retry and requery mechanisms. When the reviewer rejects a submission or when errors occur, the agent generates improved solutions based on feedback. The `RetryAgent` workflow shows iterative refinement across multiple attempts, and the error handling in `forward_with_handling()` shows immediate refinement when syntax errors or blocked actions are detected.

5. **RETRIEVAL-AUGMENTED-PLANNING:** The agent supports retrieval-augmented planning through demonstrations. The `TemplateConfig` in `agents.py` (line 90-96) includes a `demonstrations` field that loads past successful trajectories from disk. The `add_demonstrations_to_history()` method (line 617-656) incorporates these demonstrations into the agent's context, allowing it to reference successful past solutions when tackling new problems.

---

## Agent Memory

**Labels:** EPHEMERAL, EPISODIC, PROCEDURAL

**Justification:**

1. **EPHEMERAL:** The agent's primary memory is ephemeral, lasting only for the current session. The `history` attribute in `DefaultAgent` (line 481) stores the conversation context but is reset between different problem instances via the `setup()` method (line 561-606). The `HistoryProcessor` classes in `sweagent/agent/history_processors.py` manage this ephemeral context window, with processors like `LastNObservations` eliding older observations to fit within context limits.

2. **EPISODIC:** The agent maintains episodic memory through trajectory storage. Each step is recorded as a `TrajectoryStep` in `sweagent/types.py` with timestamped actions, observations, and states (line 1220-1233 in `agents.py`). Trajectories are persisted to disk as `.traj` files (line 779-787) with complete records of when actions occurred and in what context. The reviewer can access past attempt histories to make decisions about retries.

3. **PROCEDURAL:** The agent stores procedural knowledge through demonstrations and tool configurations. Demonstrations (stored in `trajectories/demonstrations/`) encode "how-to" knowledge as executable step sequences for specific task types. The tools in the `tools/` directory (edit commands, search commands, etc.) represent procedural knowledge about how to perform specific operations. The `ToolHandler` in `sweagent/tools/tools.py` manages this procedural knowledge and makes it available to the agent.

---

## Agent Tool

**Labels:** FILE-MANAGEMENT, CODE-EDITING, STRUCTURAL-RETRIEVAL, VERSION-CONTROL, PYTHON-TOOLS, TESTING-TOOLS, LINTING-VALIDATION, DATABASE-TOOLS, SYSTEM-UTILITIES, TEXT-PROCESSING, SHELL-SCRIPTING, PACKAGE-MANAGEMENT, BUILD-TOOLS, WEB-TOOLS, NAVIGATION-TOOLS, ENVIRONMENT-VARIABLES, TERMINAL-CONTROL

**Justification:**

The SWE-agent provides comprehensive tool support as evidenced by the extensive tool bundles in the `tools/` directory:

1. **FILE-MANAGEMENT:** The registry tools in `tools/registry/bin/` provide basic file system operations. The agent can navigate directories, list files, and manage file structures through bash commands enabled via `enable_bash_tool: true` in configurations.

2. **CODE-EDITING:** Multiple sophisticated editing tools are available:
   - `tools/edit_anthropic/bin/str_replace_editor` - Anthropic-style string replacement editor
   - `tools/windowed_edit_replace/` - Windowed editing with replacement
   - `tools/windowed_edit_rewrite/` - Windowed editing with rewriting
   - `tools/windowed/bin/` includes `open`, `create`, `goto`, `scroll_up`, `scroll_down` for code navigation and editing
   - All provide specialized interfaces for viewing and modifying source code files with context awareness

3. **STRUCTURAL-RETRIEVAL:** Search capabilities are provided through:
   - `tools/search/bin/search_file` - Search for patterns within files using grep
   - `tools/search/bin/search_dir` - Search across directories
   - `tools/search/bin/find_file` - Find files by name
   - These enable pattern matching and code structure discovery based on syntax rather than semantics

4. **VERSION-CONTROL:** Git operations are available through bash execution. The submission mechanism in `agents.py` (line 856) uses `git add -A && git diff --cached` to create patches. The `tools/diff_state/bin/_state_diff_state` tool tracks changes.

5. **PYTHON-TOOLS:** The agent operates in Python environments with access to Python interpreters and package managers. The `config/default.yaml` shows environment variables like `PIP_PROGRESS_BAR: 'off'` and `TQDM_DISABLE: '1'` controlling Python tools.

6. **TESTING-TOOLS:** While not visible in dedicated tool binaries, the agent can execute test commands through bash. Configuration files reference test execution as part of the workflow.

7. **LINTING-VALIDATION:** The `tools/edit_anthropic/bin/str_replace_editor` includes linting support via `USE_LINTER` registry variable (line 30), with lint warnings shown after edits (line 44-55).

8. **DATABASE-TOOLS:** Database interaction is available through bash commands, allowing the agent to use clients like `mysql`, `psql`, `sqlite3`.

9. **SYSTEM-UTILITIES:** Full access to system utilities through bash scripting, including process control, environment management, and system information commands.

10. **TEXT-PROCESSING:** Text manipulation tools are accessible via bash (`sed`, `awk`, `grep`, etc.).

11. **SHELL-SCRIPTING:** The `enable_bash_tool: true` configuration and `guard_multiline_input()` method in `tools.py` enable full bash scripting capabilities with support for control structures and complex commands.

12. **PACKAGE-MANAGEMENT:** Package installation is available through bash access to `apt-get`, `pip`, `conda`, etc.

13. **BUILD-TOOLS:** Build systems are accessible through bash (e.g., `make`, compilation commands).

14. **WEB-TOOLS:** The `tools/web_browser/bin/` directory includes comprehensive web interaction tools:
    - `open_site`, `close_site` - Site navigation
    - `click_mouse`, `double_click_mouse`, `drag_mouse`, `move_mouse` - Mouse interactions
    - `press_keys_on_page`, `scroll_on_page` - Keyboard and scroll interactions
    - `screenshot_site` - Visual capture
    - `execute_script_on_page`, `get_console_output` - JavaScript execution
    - `run_web_browser_server` - Browser server management

15. **NAVIGATION-TOOLS:** The filemap tool (`tools/filemap/bin/filemap`) and windowed navigation tools (`goto`, directory traversal) enable efficient codebase navigation. The `USE_FILEMAP` registry variable enables structured workspace exploration.

16. **ENVIRONMENT-VARIABLES:** Extensive environment configuration is supported via `tools.env_variables` in configurations (e.g., `PAGER: cat`, `MANPAGER: cat`, `LESS: -R` in `config/default.yaml`).

17. **TERMINAL-CONTROL:** Terminal control is managed through the SWE-ReX deployment layer that handles terminal sessions, as described in `docs/background/architecture.md`.

---

## Summary

SWE-agent is a **single-agent** system with **iterative and self-improving workflows** that uses **direct execution with replanning and interactive capabilities**. Its reasoning combines **ReAct, chain-of-thought, self-evaluation, self-refinement, and retrieval-augmented planning**. The agent maintains **ephemeral, episodic, and procedural memory** and has access to a **comprehensive suite of 17 tool categories** spanning file management, code editing, version control, testing, web interaction, and system utilities—making it a highly capable autonomous software engineering agent.

---

## Novel Features

No novel features requiring new taxonomy labels were identified. All observed capabilities map clearly to existing taxonomy categories.
