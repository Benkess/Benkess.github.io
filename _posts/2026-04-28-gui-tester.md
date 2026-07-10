---
title: 'GUI Tester: A Computer-Use Agent for Testing GUIs'
date: 2026-04-28
permalink: /posts/2026/04/gui-tester/
tags:
  - ai agents
  - gui testing
  - model context protocol
  - software engineering
---

## Abstract

Modern coding agents can generate surprisingly complete browser-based GUIs from short prompts. They can create pages, style components, wire up navigation, and revise code quickly. But there is still a major gap in the workflow: a generated GUI can be syntactically valid, pass automated checks, and still be wrong when someone actually opens it up and uses it.

I kept running into this problem in my own agentic coding workflows. A coding agent would create a website, dashboard, game, or browser-based tool, but I still had to open the result myself, click through the interface, notice issues, take screenshots, and explain the problem back to the agent. The agent could often fix the issue once I described it, but I was still acting as the testing loop.

That is why I built **GUI Tester**, a computer-use subagent that helps coding agents independently test GUIs through browser interaction. The GUI tester opens a web GUI, observes the screen, uses mouse and keyboard actions, records notes, saves screenshot evidence, and produces a structured report that a coding agent can then use to fix the implementation.

The project code and usage documentation are available in the public [GUI Tester repository](https://github.com/Benkess/gui-tester).

See my accompanying presentation/demo on [YouTube](https://youtu.be/_UleWr2skwg) or below:

[embed the video presentation here: [https://youtu.be/_UleWr2skwg](https://youtu.be/_UleWr2skwg). Title: "GUI Tester Final Project Presentation". Use visible playback controls and do not autoplay.]

## Motivation: The Interaction Gap

For me, a common agentic development workflow looks like this: ask a coding agent to build a GUI, run the result, then manually inspect the interface and explain what went wrong.

That last step is often where the workflow becomes less autonomous. The coding agent may have written reasonable code, and it may even be able to run syntax checks, test suites, or code-level inspections. But the user still has to perform the GUI interaction: open the page, click through the interface, notice visual or functional problems, take screenshots, describe the issue, and ask for a revision. The agent can often fix the problem once it is prompted, but the human is still acting as the testing loop.

I think of this as an **interaction gap** in agentic software development. Coding agents can read files, edit code, run test suites, inspect logs, and reason over images. But many GUI failures are not primarily code or logic-level failures. They are user-facing failures that require interacting with the rendered interface.

GUI Tester is my attempt to close that gap. Instead of relying on a human to repeatedly test the rendered result, the coding agent can call a specialized GUI-use subagent, receive a report with screenshot-backed findings, modify the implementation, and test again.

[Create a highlighted callout element here: **The goal:** enable coding agents to independently test and validate the GUIs they build.]

## System Design

The system is organized into three main layers: a computer-use agent, a GUI testing agent, and a Model Context Protocol (MCP) integration layer. Each layer adds a different piece of functionality.

[include image of slide 3 of presentation as a wide figure here, or recreate with website formatting. No caption needed.]

### Computer-Use Agent

[include `final_project/report_figures/computer_use_agent_architecture.jpg` as a large figure here. Caption: "Computer-Use Agent Architecture."]

GUI Tester is built on a **computer-use agent**, meaning it observes the browser visually and performs actions similar to how a human tester would interact with the interface.

The base computer-use agent follows a ReAct-like loop:

1. observe the current browser state,
2. reason about what needs to be checked next,
3. choose an action,
4. execute the action,
5. observe the result,
6. continue until the test is complete.

The core interaction layer supports mouse-and-keyboard-style actions such as clicking, typing, dragging, and navigating. For browser-based testing, these actions are executed through Playwright.

The computer-use agent uses LangGraph and LangChain as the agentic framework for the agent graph, context management, and tool execution. The current default configuration uses `gpt-5.4` as the vision-language model (VLM), but I also experimented with `qwen3-VL`. The framework maintains a sliding context window over recent observations, reasoning steps, and tool results, while preserving static task context such as the system prompt and user instructions.

The computer-use layer is intentionally general. It provides the basic ability to operate a GUI.

### GUI Testing Agent

[include `final_project/report_figures/GUI_testing_agent_details.jpg` as a large figure here. Caption: "GUI Testing Agent Details."]

The second layer is the GUI testing agent. This agent inherits the observation, reasoning, and action capabilities of the computer-use agent, but adds tools specifically designed for GUI evaluation.

The most important addition is a GUI testing tool with two main capabilities: note-taking and report generation. During a test, the agent can save observations as notes. These notes can optionally include screenshots, so that each finding is tied to visual evidence. At the end of a test, the agent consolidates its notes into a final structured report.

This design is important because the goal is not only to let the agent use a GUI, but also to create artifacts that a coding agent can use later. A screenshot alone can show what happened, but the coding agent also needs a written explanation of what is wrong and why it matters. Thus, the GUI testing agent produces both visual and textual outputs. For example, it can record that a page mostly matches the requested dark theme, but that a contact link is missing from the visible sidebar or that a paragraph is clipped at the bottom of the viewport.

The GUI testing agent is also given the test instructions and the expected GUI description. This allows it to compare the rendered interface against the specification rather than only making general comments about visual quality. For example, if a personal-site specification requires Email, ORCID, GitHub, and LinkedIn contact links, the tester can explicitly check whether all of those items are visible and usable in the rendered page.

### MCP Integration

The final layer exposes GUI Tester as a Model Context Protocol (MCP) server. This lets coding agents such as Claude Code or Codex call the tester as a tool without needing browser-control capabilities themselves.

The MCP tool accepts four main inputs:

- `url`: the URL or local path of the GUI to test,
- `gui_description`: a description of the intended GUI contents and behavior,
- `test_instructions`: instructions describing what the GUI tester should verify,
- `report_dir`: the directory where output artifacts should be saved.

After running the GUI testing agent, the MCP tool returns the path to the final test report and the associated artifacts, including screenshots and saved notes. This allows the coding agent to immediately read the findings, view the screenshots, and make targeted edits.

The key design decision was to make GUI Tester a separate subagent rather than building the entire workflow into one coding agent. The coding agent remains responsible for implementation, while the GUI testing agent specializes in interacting with and evaluating the user-facing result. Because the tester is exposed through MCP, the same testing capability can be reused across different coding agents and projects.

The resulting loop is simple: the coding agent calls the GUI tester through the MCP tool; the GUI tester interacts with the GUI and returns notes, screenshots, and a report; the coding agent uses those artifacts to patch the implementation or decide that the GUI is acceptable. This loop can repeat until the interface satisfies the specification, reducing the need for me to manually inspect the GUI and explain each issue.

[include `final_project/report_figures/MCP_integration_flow.jpg` as a wide figure here. Caption: "MCP integration flowchart."]

## Evaluation

I evaluated the system through a small set of GUI testing experiments. The goal was to see whether the GUI tester subagent could identify real visual and functional issues that coding agents failed to catch on their own, and whether the coding agents could then use the MCP output to fix the interface.

### Test GUIs

For the test cases, I used four buggy GUIs. These GUIs were generated by Codex from initial specifications for working interfaces and included real bugs introduced during the initial generation process.

The main example was a personal website template for a fictional person. The intended site included a landing page, blog page, and resume page. It included profile information, contact links, navigation, and page-specific content. Across the experimental GUIs, the issue types included:

- content not fitting cleanly inside the target viewport,
- sections or paragraphs being cut off,
- missing navigation links on some pages,
- missing or hidden icons and contact links,
- and layouts that technically rendered but did not satisfy the requested visual specification.

Below are images of a valid home page and visually incorrect GUIs, with issues annotated in red boxes. All of these examples were generated with Codex. The valid home page example is the result of manual reprompting.

[include `final_project/report_figures/00_valid_personal_webpage_index.png` as a large figure here. Caption: "Intended personal website interface."]

[include `final_project/report_figures/annotated_incorrect_guis.png` as a large figure here. Caption: "Red-box annotated examples of visually incorrect GUIs."]

### Individual Testing of the GUI Tester

I first tested the GUI testing agent by itself, outside of the full coding-agent loop. In this setting, the tester was given the GUI, the expected description, and instructions for what to inspect. It then opened the page, navigated through the interface, saved observations, and generated a report.

This individual testing verified that the GUI testing agent’s own tool use was reliable. In particular, it confirmed that the agent could capture screenshots, save notes, and identify visible problems in the interface. The screenshots were also useful for checking that the tester had actually observed the issue it described.

In the personal website example, the GUI tester found that the main content was clipped near the bottom of the viewport at the target 1280×720 size. It also found that the LinkedIn contact link was not visible in the left contact rows, even though the specification expected contact links such as Email, ORCID, GitHub, and LinkedIn to be available. I have included an example note and final report output from one trial below.

[include `final_project/report_figures/note_001.png` as a large figure here. Caption: "Example saved note with screenshot from the GUI tester."]

[include `final_project/report_figures/final_report_top_half.png` and `final_project/report_figures/final_report_bottom_half.png` together as a report-preview figure here. Caption: "Example GUI Tester report output."]

### MCP Testing With Coding Agents

After testing the GUI tester independently, I evaluated it as an MCP tool used by coding agents. I tested the MCP integration with both Claude Code and Codex.

The baseline result was that neither coding agent reliably identified the GUI bugs on its own. Without using the GUI tester, the agents could inspect the source code and reason about what the page was supposed to do, but they did not independently notice the rendered-interface issues. This matched the motivation for the project: the bugs were not primarily syntax or logic errors, but user-facing problems that became clear only after interacting with the GUI.

When given access to the MCP-based GUI tester, however, the coding agents were able to use the tool to catch the issues. The workflow was:

1. the coding agent called the GUI tester through the MCP tool,
2. the GUI tester inspected the interface and produced a report,
3. the coding agent read the report and patched the source code,
4. the coding agent ran the GUI tester again,
5. the loop repeated until the tester reported that the issues were resolved.

This iterative process worked especially well for the personal website example. The tester first reported the missing or hidden LinkedIn contact row and the clipped landing-page content. The coding agent then modified the layout and spacing. On the first fix attempt, the tester could still identify remaining issues. The coding agent then made additional changes and tested again. Eventually, the GUI tester reported that the implementation met the specification, and the final rendered page no longer had the original cut-off content or missing contact link.

Below are before and after images for a trial using Codex as the coding agent:

[include `final_project/report_figures/04_annotated.png` as a figure here. Title: "A failing GUI example:"]

[include `codex_fixed_gui.png`, as a figure here. Try to fit it next to the incorrect version. Title: "Fixed personal website."]

### Conclusion

I tested the system on four buggy GUIs and observed whether the GUI tester could find the intended issues and whether coding agents could use the tester to repair the GUIs. The experiments provide useful evidence that the approach works in the intended setting. The important result is not simply that the GUI tester could identify a bug, but that it could become part of an autonomous development loop.

The coding agents failed to catch the issues independently, then succeeded once they used the GUI tester through MCP. This supports the main claim of the project: a specialized GUI testing subagent can give coding agents the interactive feedback they need to fix problems that would otherwise require human inspection.

The main thing I learned is that even a small amount of interactive feedback can significantly improve an agentic coding workflow. The bugs in my test GUIs were not especially complex, but they were exactly the kinds of issues that interrupt real GUI development. These issues are easy for humans to notice, but they create friction when the coding agent has to wait for the user to point them out. The GUI tester subagent reduces that friction by letting the agent inspect its own work.

## Takeaway

GUI Tester was a small project, but it captured an idea I care about: useful AI agents need feedback from the environments they act in. For GUI development, that means more than reading code. It means opening the interface, using it, noticing what a user would notice, and turning those observations into actionable development feedback.

By wrapping a GUI-use testing agent as an MCP tool, I was able to give coding agents a reusable way to inspect and improve the interfaces they build. The result is a step toward more autonomous agentic development workflows, where agents can not only generate software, but also test the user-facing behavior of what they generated.
