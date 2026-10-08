# Engineer-two

Engineer-two is a skill inspired by Google's published paper [Scientist-two](http://scientist-two.github.io/). 
It adapts the Scientist-two workflow to pursue broader engineering and research objectives, 
rather than focusing exclusively on writing novel research papers.

When you run the skill for the first time, it will ask you to describe your research goal. 
For example, you might want to develop and benchmark an algorithm that outperforms existing alternatives.

Your goal will be saved in `goal.md`, which you can revise at any time.

Building on the strengths of Scientist-two, Engineer-two avoids generating ideas that are merely incremental improvements.
It also learns from previously unsuccessful ideas and uses those lessons to generate new ideas that are more likely to make meaningful progress toward your goal in fewer trials.

## Installation

Copy the `engineer-two` folder into your coding agent's skills directory. For example:
- `.claude/skills` for Claude Code.
- `.codex/skills` for OpenAI Codex.

## Usage

Type `/engineer-two` in the agent's chat box to start the workflow. You will then be prompted to answer several questions.

You can stop at any time to refine your goal or provide additional instructions by chatting with your agent normally.
The agent will then continue the workflow using your updated instructions.

Throughout the process, the skill will incrementally maintain:

- `goal.md`, which contains your research or engineering goal
- `project-setup.md`, which contains project-specific rules and constraints that are not part of the goal itself

Files documenting the ideas and attempted approaches will be generated throughout the process.

----

Keywords: engineertwo, engineer2, scientisttwo, scientist2
