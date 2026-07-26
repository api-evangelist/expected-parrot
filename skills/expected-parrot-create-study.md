---
name: create-study
description: Design complete surveys from free text requirements - generates a Python script with Survey, ScenarioList, and AgentList definitions
allowed-tools: Read, Glob, Skill, AskUserQuestion, Write
arguments: research_question
---

# Design Study

This skill generates a Python script with EDSL objects (Survey, ScenarioList, AgentList) based on a free text description of what the user wants the study to accomplish.


Example:
```
/create-study Do LLMs exibit anchoring bias?
```

## Workflow

### 1. Parse the User's Description

Extract from the free text:
- **Survey Goal**: What the survey is trying to measure
- **Questions needed**: Topics and question types implied
- **Scenarios**: Any variables that should vary across runs
- **Agents**: Any respondent personas mentioned
- **Rules**: Any branching or skip logic implied

If the description is ambiguous or missing key details, use AskUserQuestion to clarify before generating code.

### 2. Ask About Output Destination

Use AskUserQuestion to ask where the user wants the code:

```
Question: "Where would you like the survey code?"
Header: "Output"
Options:
  1. "Write to file (Recommended)" - "Save to a Python file with an appropriate name based on the survey topic"
  2. "Display only" - "Show the code in the conversation without saving"
```

If the user chooses to write to a file:
- Generate a snake_case filename based on the survey topic (e.g., `mafia_exit_survey.py`, `food_preferences_survey.py`)
- Write to the current working directory
- Inform the user of the filename after writing

### 3. Reference Other Skills

Read the consolidated reference skill for detailed implementation guidance:

| Skill | When to Read |
|-------|--------------|
| **edsl-survey-reference** | Question types, templating, rules, memory, helpers, visualization |

Use the Skill tool to invoke `edsl-survey-reference`, or Read the SKILL.md file from `.claude/skills/edsl-survey-reference/SKILL.md`.

### 4. Design the Survey Structure

Based on requirements, determine:

1. **Questions needed** and their types
2. **Scenario variables** for parameterization
3. **Agent traits** for respondent personas
4. **Rules** for branching/skip logic
5. **Memory configuration** for context

### 5. Generate the Code

Produce a Python script that defines:
- `Survey` with all questions and rules
- `ScenarioList` if variables are needed
- `AgentList` if personas are needed

Do NOT include code to run or analyze the survey.

## Example: Full Survey Design

### User Request
"I want to survey people about their food preferences. I want to ask about 5 different cuisines, get their favorite dish from each, and then ask follow-up questions based on their top choice."

### Generated Code

```python
from edsl import (
    Survey,
    QuestionMultipleChoice,
    QuestionFreeText,
    QuestionLinearScale,
    Scenario,
    ScenarioList,
    Agent,
    AgentList
)

# === QUESTIONS ===

# Initial preference question
q_cuisine = QuestionMultipleChoice(
    question_name="favorite_cuisine",
    question_text="Which cuisine do you enjoy most?",
    question_options=["Italian", "Japanese", "Mexican", "Indian", "Thai"]
)

# Follow-up about favorite dish (uses piping)
q_dish = QuestionFreeText(
    question_name="favorite_dish",
    question_text="What is your favorite {{ favorite_cuisine.answer }} dish?"
)

# Rating question
q_frequency = QuestionLinearScale(
    question_name="frequency",
    question_text="How often do you eat {{ favorite_cuisine.answer }} food?",
    question_options=[1, 2, 3, 4, 5],
    option_labels={1: "Rarely", 5: "Very Often"}
)

# Why they like it
q_why = QuestionFreeText(
    question_name="why_favorite",
    question_text="Why do you particularly enjoy {{ favorite_cuisine.answer }} cuisine?"
)

# === SURVEY ===

survey = (Survey([q_cuisine, q_dish, q_frequency, q_why])
    .set_full_memory_mode())  # Each question sees prior answers

# === SCENARIOS (if parameterizing) ===

# Not needed here since we use piping, but example:
# scenarios = ScenarioList([
#     Scenario({"cuisine": "Italian"}),
#     Scenario({"cuisine": "Japanese"}),
# ])

# === AGENTS (respondent personas) ===

agents = AgentList([
    Agent(traits={"persona": "health-conscious millennial", "age": 28}),
    Agent(traits={"persona": "traditional home cook", "age": 55}),
    Agent(traits={"persona": "adventurous foodie", "age": 35}),
    Agent(traits={"persona": "busy professional", "age": 42}),
])

# === READY TO RUN ===
# To execute: results = survey.by(agents).run()
```

## Design Patterns

### Pattern 1: Simple Survey (No Branching)

```python
from edsl import Survey, QuestionFreeText, QuestionMultipleChoice

questions = [
    QuestionFreeText(question_name="q1", question_text="Question 1?"),
    QuestionMultipleChoice(question_name="q2", question_text="Question 2?",
                          question_options=["A", "B", "C"]),
    QuestionFreeText(question_name="q3", question_text="Question 3?"),
]

survey = Survey(questions)
```

### Pattern 2: Branching Survey

```python
from edsl import Survey, QuestionMultipleChoice, QuestionFreeText

q_branch = QuestionMultipleChoice(
    question_name="path",
    question_text="Which topic interests you?",
    question_options=["Technology", "Nature", "Arts"]
)
q_tech = QuestionFreeText(question_name="tech_q", question_text="Tech follow-up?")
q_nature = QuestionFreeText(question_name="nature_q", question_text="Nature follow-up?")
q_arts = QuestionFreeText(question_name="arts_q", question_text="Arts follow-up?")
q_final = QuestionFreeText(question_name="final", question_text="Final thoughts?")

survey = (Survey([q_branch, q_tech, q_nature, q_arts, q_final])
    .add_rule("path", "{{ path.answer }} == 'Technology'", "tech_q")
    .add_rule("path", "{{ path.answer }} == 'Nature'", "nature_q")
    .add_rule("path", "{{ path.answer }} == 'Arts'", "arts_q")
    .add_skip_rule("tech_q", "{{ path.answer }} != 'Technology'")
    .add_skip_rule("nature_q", "{{ path.answer }} != 'Nature'")
    .add_skip_rule("arts_q", "{{ path.answer }} != 'Arts'"))
```

### Pattern 3: Parameterized Survey with Scenarios

```python
from edsl import Survey, QuestionFreeText, Scenario, ScenarioList

q = QuestionFreeText(
    question_name="opinion",
    question_text="What do you think about {{ scenario.topic }}?"
)

survey = Survey([q])

scenarios = ScenarioList([
    Scenario({"topic": "artificial intelligence"}),
    Scenario({"topic": "climate change"}),
    Scenario({"topic": "remote work"}),
])

# To run: results = survey.by(scenarios).run()
```

### Pattern 4: Agent-Based Survey (Personas)

```python
from edsl import Survey, QuestionFreeText, Agent, AgentList

q = QuestionFreeText(
    question_name="perspective",
    question_text="As a {{ agent.role }}, what's your view on automation?"
)

survey = Survey([q])

agents = AgentList([
    Agent(traits={"role": "factory worker", "experience": "20 years"}),
    Agent(traits={"role": "tech executive", "experience": "15 years"}),
    Agent(traits={"role": "policy maker", "experience": "10 years"}),
])

# To run: results = survey.by(agents).run()
```

### Pattern 5: Full Factorial Design (Scenarios × Agents)

```python
from edsl import Survey, QuestionLinearScale, Scenario, ScenarioList, Agent, AgentList

q = QuestionLinearScale(
    question_name="trust",
    question_text="How much do you trust {{ scenario.source }} for {{ scenario.topic }} news?",
    question_options=[1, 2, 3, 4, 5],
    option_labels={1: "Not at all", 5: "Completely"}
)

survey = Survey([q])

scenarios = ScenarioList([
    Scenario({"source": "social media", "topic": "political"}),
    Scenario({"source": "traditional news", "topic": "political"}),
    Scenario({"source": "social media", "topic": "scientific"}),
    Scenario({"source": "traditional news", "topic": "scientific"}),
])

agents = AgentList([
    Agent(traits={"age_group": "18-30", "education": "college"}),
    Agent(traits={"age_group": "31-50", "education": "college"}),
    Agent(traits={"age_group": "51+", "education": "college"}),
])

# To run (4 scenarios × 3 agents = 12 responses):
# results = survey.by(scenarios).by(agents).run()
```

## Output

Generate a complete Python script that:

1. **Imports** all necessary EDSL classes
2. **Defines Questions** with appropriate types
3. **Creates Survey** with any rules, memory, or skip logic
4. **Defines ScenarioList** if parameterization is needed
5. **Defines AgentList** if respondent personas are specified

**Do NOT include**:
- Code that runs the survey (`.run()` calls)
- Code that analyzes results
- Code that saves or loads data

The script should define the objects and be ready for the user to run separately.

## Checklist Before Generating

- [ ] All question names are valid Python identifiers
- [ ] Question types match the data being collected
- [ ] Piping references only reference earlier questions
- [ ] Skip rules cover all branches appropriately
- [ ] Memory mode is appropriate for the survey flow
- [ ] Scenarios cover all required variations
- [ ] Agent traits are realistic and relevant
- [ ] Code includes proper imports
