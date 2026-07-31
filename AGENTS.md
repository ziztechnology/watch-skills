# AGENTS.md

## Writing Rules

- Skills must always be written in English.
- Use direct and explicit directive sentences to describe the intended behavior, applicable conditions, and expected results. For example: “Components must be placed in separate files.” “Output only the modified code.”
- By default, describe only the practices that should be followed and provide only correct examples that align with the expected behavior.
- Add negative instructions such as “prohibited” and “must not” only when hard boundaries are involved, positive instructions cannot eliminate ambiguity, or the model repeatedly makes a specific error.
- When using negative instructions, clearly define their scope; when a viable alternative exists, also state the practice that should be followed.
- Provide incorrect examples only when correct examples are insufficient to eliminate common ambiguity; incorrect examples should be clearly labeled, kept concise, and placed immediately next to the corresponding correct examples.
- Choose wording according to the force of the rule: “must,” “prohibited,” and “must not” indicate hard constraints; “should” indicates a default rule; “prefer” indicates the preferred approach; and “may” indicates permission without requirement.
- Each Skill must be self-contained and independently provide the rules, process, and context required to accomplish its goal.
- Each Skill must be self-contained and independently executable. Do not instruct the AI in `description` or SKILL.md to read, invoke, or depend on other Skills. An Agent may independently select multiple Skills as needed for the task; any supplementary content required by the current Skill must be placed in its own `references`.
- When improving the AI's ability to recognize and trigger a Skill, refine that Skill's own `description` to clearly describe its applicable scenarios, trigger conditions, and capability boundaries.
