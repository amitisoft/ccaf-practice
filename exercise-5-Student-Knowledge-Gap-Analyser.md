# Coding Exercise: Student Knowledge Gap Analyser

**Domains:** 2 (Tool Design & MCP Integration) + 5 (Context Management & Reliability)
**Scenario:** Education — adaptive tutoring agent
**Time:** ~1 hour with Claude Code
**Stack:** Python 3.10+ + Anthropic SDK

---

## Concepts you will practise

This exercise maps directly to exam task statements. Each part makes a specific
concept concrete through implementation.

### Domain 2 — Tool Design & MCP Integration

| Concept | Where in this exercise |
|---|---|
| **Tool descriptions as selection mechanism** — minimal descriptions cause misrouting | Part 1: two similar-sounding tools with differentiated descriptions |
| **Input formats, examples, edge cases in descriptions** — what good descriptions contain | Part 1: writing tool descriptions that prevent ambiguity |
| **Structured error responses** — `errorCategory`, `isRetryable`, human-readable descriptions | Part 2: mock tools return typed errors the agent can act on |
| **Transient vs validation vs permission errors** — different recovery paths | Part 2: each mock tool demonstrates a different error type |
| **Tool distribution** — giving agents only the tools relevant to their role | Part 2: the tutor agent gets 4 tools, not all 7 defined |
| **`tool_choice` configuration** — `"auto"`, `"any"`, forced selection | Part 2: force `get_student_history` as the first call every session |
| **MCP `isError` flag pattern** — how tool failures are communicated back to the agent | Part 2: all mock tools use the `isError` pattern |

### Domain 5 — Context Management & Reliability

| Concept | Where in this exercise |
|---|---|
| **Persistent "case facts" block** — transactional data kept outside summarised history | Part 3: student facts block preserved across all turns |
| **Trimming verbose tool outputs** — keep only relevant fields before accumulation | Part 3: `trim_tool_result()` strips unused assessment fields |
| **Escalation triggers** — customer request, policy gap, inability to progress | Part 3: three distinct escalation conditions with examples |
| **Structured handoff summaries** — self-contained for a human who lacks transcript access | Part 3: `build_escalation_handoff()` |
| **Error propagation** — structured error context, partial results, what was attempted | Part 3: assessment tool timeout propagated with context |
| **Information provenance** — linking recommendations back to their source assessments | Part 3: every resource recommendation carries `source_assessment_id` |
| **Confidence-based human review routing** — low-confidence extractions go to humans | Stretch: route diagnostic sessions below confidence threshold |

---

## Scenario

You are building an AI tutoring assistant for a secondary school. When a student
starts a session, the assistant:

1. Retrieves the student's prior assessment history
2. Runs a diagnostic to identify current knowledge gaps
3. Recommends targeted learning resources for each gap
4. Logs the session progress for the teacher's dashboard
5. Escalates to a human tutor when the student is significantly struggling,
   when policy does not cover the student's situation, or when the student
   requests it

The system has access to a mock school database through tool calls.

---

## Setup (5 min)

```bash
mkdir tutor-pipeline && cd tutor-pipeline
python -m venv .venv && source .venv/bin/activate
pip install anthropic
export ANTHROPIC_API_KEY=your_key_here
touch tutor.py school_db.py
```

Open in Claude Code:
```bash
claude
```

---

## Sample data

Paste this into `school_db.py`. These are the mock data stores your tool
implementations will query.

```python
# school_db.py

STUDENTS = {
    "S001": {
        "id": "S001",
        "name": "Maya Chen",
        "grade": 10,
        "subject": "Python Programming",
        "teacher": "Ms. Rodriguez",
        "learning_plan": "standard"
    },
    "S002": {
        "id": "S002",
        "name": "Jordan Williams",
        "grade": 10,
        "subject": "Python Programming",
        "teacher": "Ms. Rodriguez",
        "learning_plan": "standard"
    },
    "S003": {
        "id": "S003",
        "name": "Sam Okonkwo",
        "grade": 10,
        "subject": "Python Programming",
        "teacher": "Ms. Rodriguez",
        "learning_plan": "accelerated"
    },
    "S004": {
        "id": "S004",
        "name": "Priya Sharma",
        "grade": 10,
        "subject": "Python Programming",
        "teacher": "Ms. Rodriguez",
        "learning_plan": "support"   # IEP student — escalation required for any plan changes
    }
}

# Topic scores out of 100 from most recent assessments
ASSESSMENT_HISTORY = {
    "S001": {
        "variables_and_types": 88,
        "loops": 76,
        "functions": 91,
        "lists_and_tuples": 82,
        "dictionaries": 47,   # clear gap
        "file_handling": 39,  # clear gap
        "classes_and_oop": 31 # clear gap
    },
    "S002": {
        "variables_and_types": 55,
        "loops": 42,          # gap
        "functions": 38,      # gap
        "lists_and_tuples": 61,
        "dictionaries": 29,   # gap
        "file_handling": 22,  # gap
        "classes_and_oop": 18 # gap — Jordan is significantly struggling
    },
    "S003": {
        "variables_and_types": 97,
        "loops": 94,
        "functions": 96,
        "lists_and_tuples": 98,
        "dictionaries": 91,
        "file_handling": 88,
        "classes_and_oop": 85
    },
    "S004": {
        "variables_and_types": 72,
        "loops": 65,
        "functions": 58,      # slight gap
        "lists_and_tuples": 70,
        "dictionaries": 45,   # gap
        "file_handling": 40,  # gap
        "classes_and_oop": 35 # gap
    }
}

# Learning resources catalogue
RESOURCES = {
    "dictionaries": [
        {
            "id": "R_DICT_01",
            "title": "Python Dictionaries — Beginner Guide",
            "url": "https://school.example.com/python/dicts-beginner",
            "difficulty": "beginner",
            "duration_minutes": 20
        },
        {
            "id": "R_DICT_02",
            "title": "Dictionary Methods and Comprehensions",
            "url": "https://school.example.com/python/dicts-intermediate",
            "difficulty": "intermediate",
            "duration_minutes": 35
        }
    ],
    "file_handling": [
        {
            "id": "R_FILE_01",
            "title": "Reading and Writing Files in Python",
            "url": "https://school.example.com/python/files-intro",
            "difficulty": "beginner",
            "duration_minutes": 25
        }
    ],
    "classes_and_oop": [
        {
            "id": "R_OOP_01",
            "title": "Introduction to Classes and Objects",
            "url": "https://school.example.com/python/oop-intro",
            "difficulty": "beginner",
            "duration_minutes": 40
        },
        {
            "id": "R_OOP_02",
            "title": "Inheritance and Polymorphism",
            "url": "https://school.example.com/python/oop-advanced",
            "difficulty": "intermediate",
            "duration_minutes": 50
        }
    ],
    "loops": [
        {
            "id": "R_LOOP_01",
            "title": "For and While Loops — Practice Set",
            "url": "https://school.example.com/python/loops-practice",
            "difficulty": "beginner",
            "duration_minutes": 30
        }
    ],
    "functions": [
        {
            "id": "R_FUNC_01",
            "title": "Functions, Parameters, and Return Values",
            "url": "https://school.example.com/python/functions-guide",
            "difficulty": "beginner",
            "duration_minutes": 25
        }
    ]
}

# Simulated tool failure scenarios (for Part 2 testing)
# Set to True to trigger each failure type during testing
SIMULATE_FAILURES = {
    "assessment_timeout": False,   # transient — retryable
    "invalid_topic": False,        # validation — not retryable
    "iep_permission": False        # permission — not retryable, escalate
}
```

---

## Part 1 — Tool Interface Design (15 min)

### Concepts: tool descriptions as selection mechanism, input formats and boundaries, differentiating similar tools

### The problem with bad descriptions

Your tutoring agent needs to distinguish between two assessment-related tools:
- One that retrieves **historical** scores from the database
- One that **runs a fresh diagnostic** quiz right now

If both have minimal descriptions, the agent will misroute. Your first task is to
write descriptions that make the boundary clear, then wire up the tool definitions.

**Key exam concept:** Tool descriptions are the primary mechanism LLMs use for tool
selection. A description should include: what the tool does, what inputs it expects,
what it returns, and critically — when to use it *instead of* a similar alternative.

```python
# tutor.py
import json
import time
import anthropic
from school_db import STUDENTS, ASSESSMENT_HISTORY, RESOURCES, SIMULATE_FAILURES

client = anthropic.Anthropic()


# ── TOOL DEFINITIONS ─────────────────────────────────────────────────────────
# Part 1: Your job is to write the descriptions.
# Each description should include:
#   - What the tool does
#   - What inputs it accepts and in what format
#   - What it returns
#   - When to use it vs the similar alternative (the critical differentiator)

TOOL_GET_STUDENT_HISTORY = {
    "name": "get_student_history",
    "description": (
        # TODO: Write a description that makes clear this tool retrieves
        # HISTORICAL scores already stored in the database from PAST assessments.
        # It does NOT run a new assessment. It does NOT generate questions.
        # Use it when you need to know what the student has scored before,
        # NOT when you want to test the student right now.
        # Include: input format (student_id as "SXXX"), what it returns,
        # and explicitly state the contrast with run_diagnostic_assessment.
        "TODO: replace with your description"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "student_id": {
                "type": "string",
                "description": "Student ID in format 'SXXX' (e.g. 'S001')"
            }
        },
        "required": ["student_id"]
    }
}

TOOL_RUN_DIAGNOSTIC = {
    "name": "run_diagnostic_assessment",
    "description": (
        # TODO: Write a description that makes clear this tool ACTIVELY TESTS
        # the student right now with a set of questions on a specific topic.
        # It generates a score from the current interaction — not from history.
        # Use it when you want to assess a student's CURRENT understanding,
        # NOT when you just need their past scores.
        # Include: what topic values are valid, what it returns,
        # and the contrast with get_student_history.
        "TODO: replace with your description"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "student_id": {
                "type": "string",
                "description": "Student ID in format 'SXXX'"
            },
            "topic": {
                "type": "string",
                "description": (
                    "Topic to assess. Valid values: "
                    "'variables_and_types', 'loops', 'functions', "
                    "'lists_and_tuples', 'dictionaries', 'file_handling', "
                    "'classes_and_oop'"
                )
            }
        },
        "required": ["student_id", "topic"]
    }
}

TOOL_RECOMMEND_RESOURCE = {
    "name": "recommend_learning_resource",
    "description": (
        # TODO: Write a description for a tool that looks up resources
        # from the school's catalogue for a given topic and difficulty level.
        # Make clear it returns a specific resource with URL and duration.
        # Include when to use it: AFTER identifying a gap, not for browsing.
        "TODO: replace with your description"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "topic": {"type": "string"},
            "difficulty": {
                "type": "string",
                "enum": ["beginner", "intermediate"],
                "description": "Match to student's current level: beginner if score < 60, intermediate if 60-75"
            }
        },
        "required": ["topic", "difficulty"]
    }
}

TOOL_ESCALATE_TO_TUTOR = {
    "name": "escalate_to_human_tutor",
    "description": (
        # TODO: Write a description that is clear about WHEN to escalate.
        # Include the three trigger conditions:
        # 1. Student explicitly asks for a human tutor
        # 2. Student is significantly struggling (4+ topics below 50%)
        # 3. Student has a support learning plan (IEP) requiring plan changes
        # Make clear this should NOT be used for every difficult question —
        # only for the specific conditions above.
        "TODO: replace with your description"
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "student_id": {"type": "string"},
            "reason": {
                "type": "string",
                "enum": ["student_request", "significant_struggle", "iep_policy_gap"]
            },
            "summary": {
                "type": "string",
                "description": "Self-contained summary for the human tutor including student name, gaps identified, and recommended next action."
            }
        },
        "required": ["student_id", "reason", "summary"]
    }
}

# The agent gets only these 4 tools — not the full set of 7+ you might define.
# Key exam concept (Task 2.3): giving an agent too many tools degrades
# selection reliability. Scope tool access to the agent's actual role.
AGENT_TOOLS = [
    TOOL_GET_STUDENT_HISTORY,
    TOOL_RUN_DIAGNOSTIC,
    TOOL_RECOMMEND_RESOURCE,
    TOOL_ESCALATE_TO_TUTOR
]
```

**Reflection question (write your answer as a comment):**
> Your tool list includes both `get_student_history` and `run_diagnostic_assessment`.
> Without clear descriptions, what is the most likely misrouting failure? What
> phrase in each description is doing the most work to prevent it?

---

## Part 2 — Structured Error Responses + Tool Choice (20 min)

### Concepts: structured error responses, transient vs validation vs permission errors, isRetryable, tool_choice configuration

### Your task

Implement the four mock tool functions that simulate your school database. Each
function must return structured error responses using the MCP `isError` pattern —
not raw Python exceptions. The agent needs this structure to decide how to recover.

**Key exam concept:** Generic error messages like `"Operation failed"` prevent the
agent from making intelligent recovery decisions. Structured errors with `errorCategory`
and `isRetryable` tell the agent: should I retry? Should I escalate? Should I explain
to the student and move on?

**Error categories to implement:**

| Category | `isRetryable` | Example in this system |
|---|---|---|
| `transient` | `True` | Assessment database timed out |
| `validation` | `False` | Invalid topic name passed to diagnostic tool |
| `permission` | `False` | IEP student — plan changes require human approval |

```python
# ── MOCK TOOL IMPLEMENTATIONS ─────────────────────────────────────────────────

def tool_get_student_history(student_id: str) -> dict:
    """
    Retrieves historical assessment scores for a student.
    Returns structured data or a structured error.
    """
    # Simulate transient failure (for testing)
    if SIMULATE_FAILURES.get("assessment_timeout"):
        # TODO: return a structured error response using the isError pattern
        # Include:
        #   isError: True
        #   errorCategory: "transient"
        #   isRetryable: True
        #   message: a human-readable explanation
        #   attempted: what was being attempted when the error occurred
        return {"isError": True}  # TODO: fill this in

    if student_id not in STUDENTS:
        # TODO: return a validation error — student not found
        # isRetryable should be False (the student ID is just wrong)
        return {"isError": True}  # TODO: fill this in

    student = STUDENTS[student_id]
    scores = ASSESSMENT_HISTORY.get(student_id, {})

    # Return only the fields relevant to the tutor agent
    # Key exam concept (Task 5.1): trim verbose results before they accumulate
    return {
        "isError": False,
        "student_name": student["name"],
        "learning_plan": student["learning_plan"],
        "assessment_scores": scores,
        "topics_below_60": [t for t, s in scores.items() if s < 60],
        "topics_below_50": [t for t, s in scores.items() if s < 50]
    }


def tool_run_diagnostic(student_id: str, topic: str) -> dict:
    """
    Simulates running a live diagnostic quiz on a specific topic.
    In production this would interact with an assessment engine.
    """
    valid_topics = [
        "variables_and_types", "loops", "functions",
        "lists_and_tuples", "dictionaries", "file_handling", "classes_and_oop"
    ]

    if topic not in valid_topics:
        # TODO: return a validation error — invalid topic
        # Include the list of valid topics in the message so the agent can self-correct
        # isRetryable: False (the topic name itself is wrong)
        return {"isError": True}  # TODO: fill this in

    if SIMULATE_FAILURES.get("invalid_topic"):
        return {
            "isError": True,
            "errorCategory": "validation",
            "isRetryable": False,
            "message": f"Topic '{topic}' is not in the assessment catalogue.",
            "validTopics": valid_topics
        }

    # Simulate a diagnostic score slightly different from history
    # (represents current performance which may differ from past assessments)
    import random
    base_score = ASSESSMENT_HISTORY.get(student_id, {}).get(topic, 50)
    diagnostic_score = max(0, min(100, base_score + random.randint(-10, 10)))

    return {
        "isError": False,
        "student_id": student_id,
        "topic": topic,
        "diagnostic_score": diagnostic_score,
        "questions_asked": 5,
        "assessment_type": "diagnostic",
        # Provenance: every result carries its own identifier
        # Key exam concept (Task 5.6): link downstream recommendations to this source
        "assessment_id": f"DIAG_{student_id}_{topic}_{int(time.time())}"
    }


def tool_recommend_resource(topic: str, difficulty: str) -> dict:
    """
    Looks up a learning resource from the school catalogue.
    """
    if topic not in RESOURCES:
        return {
            "isError": True,
            "errorCategory": "validation",
            "isRetryable": False,
            "message": f"No resources found for topic '{topic}'.",
            "availableTopics": list(RESOURCES.keys())
        }

    resources_for_topic = [
        r for r in RESOURCES[topic] if r["difficulty"] == difficulty
    ]

    if not resources_for_topic:
        # Valid query, valid topic — just no resources at this difficulty
        # Key exam concept (Task 2.2): distinguish access failures from valid empty results
        return {
            "isError": False,
            "found": False,
            "message": f"No {difficulty} resources available for '{topic}'. Try 'beginner'.",
            "topic": topic,
            "difficulty_requested": difficulty
        }

    resource = resources_for_topic[0]
    return {
        "isError": False,
        "found": True,
        "resource_id": resource["id"],
        "title": resource["title"],
        "url": resource["url"],
        "duration_minutes": resource["duration_minutes"],
        "topic": topic
    }


def tool_escalate_to_tutor(student_id: str, reason: str, summary: str) -> dict:
    """
    Flags the student for human tutor intervention.
    For IEP students (learning_plan == "support"), this always succeeds.
    """
    if SIMULATE_FAILURES.get("iep_permission"):
        return {
            "isError": True,
            "errorCategory": "permission",
            "isRetryable": False,
            "message": "IEP students require escalation through the SENCO coordinator, not the general tutor queue.",
            "redirectTo": "SENCO"
        }

    student = STUDENTS.get(student_id, {})
    return {
        "isError": False,
        "escalated": True,
        "student_name": student.get("name"),
        "assigned_to": student.get("teacher"),
        "reason": reason,
        "ticket_id": f"ESC_{student_id}_{int(time.time())}"
    }


# ── TOOL DISPATCHER ───────────────────────────────────────────────────────────

def execute_tool(tool_name: str, tool_input: dict) -> dict:
    """Routes tool calls to their implementations."""
    if tool_name == "get_student_history":
        return tool_get_student_history(tool_input["student_id"])
    elif tool_name == "run_diagnostic_assessment":
        return tool_run_diagnostic(tool_input["student_id"], tool_input["topic"])
    elif tool_name == "recommend_learning_resource":
        return tool_recommend_resource(tool_input["topic"], tool_input["difficulty"])
    elif tool_name == "escalate_to_human_tutor":
        return tool_escalate_to_tutor(
            tool_input["student_id"],
            tool_input["reason"],
            tool_input["summary"]
        )
    else:
        return {
            "isError": True,
            "errorCategory": "validation",
            "isRetryable": False,
            "message": f"Unknown tool: {tool_name}"
        }
```

**Now implement the forced first-call pattern:**

```python
# Key exam concept (Task 2.3): forced tool_choice guarantees the agent
# always retrieves student history FIRST before doing anything else.
# This is deterministic — a prompt instruction like "always start with
# get_student_history" has a non-zero failure rate.

def start_session(student_id: str) -> dict:
    """
    Forces get_student_history as the first tool call of every session.
    Returns the student history for use as the session's "case facts" block.
    """
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=500,
        tools=AGENT_TOOLS,
        # TODO: set tool_choice to force get_student_history specifically
        # Hint: {"type": "tool", "name": "get_student_history"}
        tool_choice=None,  # TODO
        messages=[{
            "role": "user",
            "content": f"Begin a tutoring session for student {student_id}."
        }]
    )

    # TODO: extract the tool_use block and execute it
    # Return the result of execute_tool("get_student_history", ...)
    pass
```

**Test your error handling:**

```python
if __name__ == "__main__":
    # Test 1: normal flow
    result = tool_get_student_history("S001")
    print("Normal:", json.dumps(result, indent=2))

    # Test 2: transient error
    SIMULATE_FAILURES["assessment_timeout"] = True
    result = tool_get_student_history("S001")
    print("\nTransient error:", json.dumps(result, indent=2))
    SIMULATE_FAILURES["assessment_timeout"] = False

    # Test 3: validation error
    result = tool_run_diagnostic("S001", "quantum_mechanics")
    print("\nValidation error:", json.dumps(result, indent=2))

    # Test 4: valid empty result (not an error)
    result = tool_recommend_resource("classes_and_oop", "intermediate")
    print("\nEmpty result:", json.dumps(result, indent=2))

    # Test 5: forced first call
    history = start_session("S002")
    print("\nSession start (forced tool):", json.dumps(history, indent=2))
```

**Check your work:**
- Transient errors have `isRetryable: True`, validation and permission errors have `isRetryable: False`
- The empty resource result has `isError: False` — it is a successful query that found nothing
- `start_session` calls `get_student_history` regardless of what the prompt says

---

## Part 3 — Context Management + Escalation + Error Propagation (25 min)

### Concepts: persistent case facts, trimming tool outputs, escalation triggers, structured handoffs, error propagation with provenance

### Your task

Build the full tutoring session loop. The agent must:
1. Keep a `student_facts` block that persists across all turns (never summarised away)
2. Trim verbose tool outputs before they accumulate
3. Apply explicit escalation criteria
4. Propagate errors with enough context for recovery
5. Attach `source_assessment_id` to every resource recommendation

```python
# ── CONTEXT MANAGEMENT ────────────────────────────────────────────────────────

def trim_tool_result(tool_name: str, result: dict) -> dict:
    """
    Strips fields not needed downstream before the result accumulates in context.

    Key exam concept (Task 5.1): tool results accumulate token-by-token across
    turns. A full assessment result has many fields the tutor agent does not need
    for its next decision. Trim early to prevent context bloat.
    """
    if result.get("isError"):
        # Always keep full error context — the agent needs it to recover
        return result

    if tool_name == "get_student_history":
        # TODO: keep only: student_name, learning_plan, topics_below_60, topics_below_50
        # Drop: assessment_scores (the raw scores are verbose; the gap lists are enough)
        pass

    if tool_name == "run_diagnostic_assessment":
        # TODO: keep only: topic, diagnostic_score, assessment_id
        # Drop: questions_asked, assessment_type (not needed for recommendations)
        pass

    if tool_name == "recommend_learning_resource":
        # TODO: keep only: found, title, url, topic, resource_id
        # Drop: duration_minutes (useful for display, not for agent decisions)
        pass

    return result  # TODO: replace pass statements with return statements


def build_student_facts_block(history: dict) -> str:
    """
    Formats the student history as a persistent "case facts" block.
    This block is prepended to EVERY prompt in the session — never summarised.

    Key exam concept (Task 5.1): transactional data (names, IDs, known gaps)
    must be kept verbatim outside the summarised conversation history.
    If this goes into the normal history, progressive summarisation will
    eventually lose the specific topic names and scores.
    """
    if history.get("isError"):
        return "STUDENT FACTS: [unavailable — history retrieval failed]"

    # TODO: format the student facts into a clear block, e.g.:
    # STUDENT FACTS (do not summarise this section):
    # Name: Maya Chen | Learning Plan: standard
    # Topics below 60%: dictionaries, file_handling, classes_and_oop
    # Topics below 50%: file_handling, classes_and_oop
    return "TODO: format student facts block"


# ── ESCALATION LOGIC ──────────────────────────────────────────────────────────

def should_escalate(student_id: str, history: dict, user_message: str) -> tuple[bool, str]:
    """
    Evaluates escalation triggers. Returns (should_escalate, reason).

    Key exam concept (Task 5.2): escalation requires explicit categorical
    criteria, not sentiment analysis or self-reported confidence scores.

    Three triggers:
    1. Student explicitly requests a human tutor
    2. Student is significantly struggling (4+ topics below 50%)
    3. Student has a support learning plan (IEP) — any plan adjustment requires human
    """
    # Trigger 1: explicit student request
    # TODO: check if user_message contains a direct request for a human tutor
    # e.g. "I want to speak to my teacher", "can I talk to a real person", "get my tutor"
    # Key exam concept: honour this immediately — do not attempt to resolve first

    # Trigger 2: significant academic struggle
    # TODO: check if len(history.get("topics_below_50", [])) >= 4
    # Key exam concept: this is a policy-defined threshold, not a model confidence score

    # Trigger 3: IEP policy gap
    # TODO: check if history.get("learning_plan") == "support"
    # Key exam concept: IEP students require human oversight for any plan changes
    # This is a policy gap — the automated system is not authorised to act independently

    return False, ""  # TODO: replace with your logic


def build_escalation_handoff(
    student_id: str,
    history: dict,
    reason: str,
    diagnostics_run: list[dict],
    resources_recommended: list[dict]
) -> str:
    """
    Builds a self-contained handoff summary for the human tutor.

    Key exam concept (Task 5.2): the human tutor will NOT have access to the
    conversation transcript. The handoff must contain everything they need
    to act without starting over. It must also preserve provenance — which
    assessment led to which recommendation.
    """
    # TODO: build a structured summary including:
    # - Student name and ID
    # - Learning plan type
    # - Escalation reason (student_request / significant_struggle / iep_policy_gap)
    # - Topics assessed this session and their diagnostic scores
    # - Resources recommended (with source_assessment_id showing which diagnostic triggered each)
    # - Recommended next action for the human tutor
    return "TODO: build handoff summary"


# ── ERROR PROPAGATION ─────────────────────────────────────────────────────────

def handle_tool_error(tool_name: str, error: dict, attempt: int) -> dict:
    """
    Decides how to handle a tool error and what context to propagate upward.

    Key exam concept (Task 5.3): the agent needs structured error context —
    failure type, what was attempted, whether retry is appropriate — to make
    an intelligent recovery decision. Generic errors prevent recovery.
    """
    # TODO: implement error handling logic:
    #
    # If isRetryable is True and attempt < 2:
    #   Return {"action": "retry", "message": f"Transient failure on {tool_name}, retrying..."}
    #
    # If errorCategory == "permission":
    #   Return {"action": "escalate", "message": error["message"], "redirectTo": error.get("redirectTo")}
    #
    # If isRetryable is False (validation error):
    #   Return {"action": "skip", "message": error["message"]}
    #       — log it, move to the next topic, do not escalate

    return {"action": "skip", "message": error.get("message", "Unknown error")}


# ── MAIN SESSION LOOP ─────────────────────────────────────────────────────────

SYSTEM_PROMPT = """You are a supportive secondary school Python programming tutor.

Your session flow:
1. Review the student's history from the STUDENT FACTS block
2. Run a diagnostic on their weakest topic (lowest score below 60%)
3. Recommend an appropriate learning resource for each identified gap
4. If the student asks questions, answer them clearly and encouragingly
5. Escalate to a human tutor only when explicitly triggered (see tool description)

Always address the student by their first name. Be encouraging but honest about gaps.
Focus on one topic at a time — do not overwhelm the student."""


def run_tutoring_session(student_id: str) -> None:
    """
    Runs a complete adaptive tutoring session for a student.
    """
    print(f"\n{'='*60}")
    print(f"Starting session for student {student_id}")

    # Step 1: Force retrieval of student history as session foundation
    print("  → Retrieving student history (forced first call)...")
    history_raw = start_session(student_id)
    history = trim_tool_result("get_student_history", history_raw)

    if history.get("isError"):
        error_handling = handle_tool_error("get_student_history", history, attempt=1)
        print(f"  [error] Cannot start session: {error_handling['message']}")
        return

    # Step 2: Build the persistent student facts block
    # Key exam concept: this block is prepended to EVERY message, never summarised
    student_facts = build_student_facts_block(history)
    print(f"  → Student facts loaded: {history.get('student_name')}")
    print(f"  → Gaps identified: {history.get('topics_below_60', [])}")

    # Track what happens this session (for provenance in the handoff)
    diagnostics_run = []
    resources_recommended = []

    # Step 3: Run the agentic tutoring loop
    messages = []
    student_input = (
        f"Hi, I'm ready to study. Can you help me improve my Python skills?"
    )

    for turn in range(6):  # limit turns for the exercise
        # Check escalation triggers before processing
        should_esc, esc_reason = should_escalate(student_id, history, student_input)
        if should_esc:
            print(f"  → Escalation triggered: {esc_reason}")
            handoff = build_escalation_handoff(
                student_id, history, esc_reason, diagnostics_run, resources_recommended
            )
            result = tool_escalate_to_tutor(student_id, esc_reason, handoff)
            print(f"  → Escalated: {result}")
            break

        # Prepend student facts block to preserve critical data outside summarised history
        # Key exam concept: this is the "case facts" pattern — always present, never compressed
        full_message = f"{student_facts}\n\n---\n\n{student_input}"
        messages.append({"role": "user", "content": full_message})

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1000,
            system=SYSTEM_PROMPT,
            tools=AGENT_TOOLS,
            tool_choice={"type": "auto"},
            messages=messages
        )

        # Process response: handle tool calls or final text
        assistant_message = {"role": "assistant", "content": response.content}
        messages.append(assistant_message)

        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"  → Tool call: {block.name}({block.input})")

                    # Execute tool with retry logic for transient errors
                    raw_result = None
                    for attempt in range(2):
                        raw_result = execute_tool(block.name, block.input)
                        if not raw_result.get("isError"):
                            break
                        handling = handle_tool_error(block.name, raw_result, attempt)
                        if handling["action"] != "retry":
                            break
                        print(f"    [retry] {handling['message']}")
                        time.sleep(1)

                    # Trim before adding to context
                    trimmed = trim_tool_result(block.name, raw_result)

                    # Track provenance: record what was assessed and recommended
                    if block.name == "run_diagnostic_assessment" and not trimmed.get("isError"):
                        diagnostics_run.append(trimmed)
                    if block.name == "recommend_learning_resource" and trimmed.get("found"):
                        # TODO: attach source_assessment_id from the most recent diagnostic
                        # Key exam concept (Task 5.6): link recommendations to their source
                        trimmed["source_assessment_id"] = (
                            diagnostics_run[-1]["assessment_id"] if diagnostics_run else None
                        )
                        resources_recommended.append(trimmed)

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(trimmed)
                    })

            messages.append({"role": "user", "content": tool_results})

        elif response.stop_reason == "end_turn":
            # Extract agent's text response to show to student
            agent_text = next(
                (b.text for b in response.content if hasattr(b, "text")), ""
            )
            print(f"\n  [Agent → Student]: {agent_text[:300]}...")

            if turn < 5:
                # Simulate student follow-up for the exercise
                follow_ups = [
                    "Which topic should I focus on first?",
                    "That makes sense. What resources do you recommend?",
                    "I find classes really confusing. Can you explain more?",
                    "Ok thanks. What should I do next?"
                ]
                student_input = follow_ups[min(turn, len(follow_ups)-1)]
            else:
                print("\n  [Session complete]")
                break

        time.sleep(0.5)

    # Session summary
    print(f"\n  Session summary for {history.get('student_name')}:")
    print(f"  Diagnostics run: {len(diagnostics_run)}")
    print(f"  Resources recommended: {len(resources_recommended)}")
    if resources_recommended:
        print("  Resource provenance:")
        for r in resources_recommended:
            print(f"    {r.get('title')} ← {r.get('source_assessment_id')}")


if __name__ == "__main__":
    # Run sessions for all four students
    for student_id in ["S001", "S002", "S003", "S004"]:
        run_tutoring_session(student_id)
        time.sleep(2)
```

---

## Success criteria

### Domain 2 — Tool Design & MCP Integration
- [ ] `get_student_history` and `run_diagnostic_assessment` descriptions each include a "when to use this *instead of* the other" clause
- [ ] Transient errors have `isRetryable: True`; validation and permission errors have `isRetryable: False`
- [ ] The empty resource result for `recommend_learning_resource` returns `isError: False` with `found: False` — not an error
- [ ] `start_session` forces `get_student_history` via `tool_choice` — not via a prompt instruction
- [ ] The agent receives only 4 tools, not all 7+ you could define
- [ ] `tool_run_diagnostic` includes valid topic names in its validation error so the agent can self-correct

### Domain 5 — Context Management & Reliability
- [ ] `student_facts` block is prepended to every message — never included in summarised history
- [ ] `trim_tool_result` strips unused fields from each tool's output before it accumulates
- [ ] S002 (Jordan, 5 topics below 50%) triggers escalation via the `significant_struggle` path
- [ ] S004 (Priya, support learning plan) triggers escalation via the `iep_policy_gap` path
- [ ] `build_escalation_handoff` produces a self-contained summary including all diagnostics and recommendations
- [ ] Every resource in `resources_recommended` carries a `source_assessment_id` linking it to its diagnostic

---

## Stretch goals

**Stretch 1 — Parallel diagnostics (Domain 1, Task 1.3)**
When a student has 3+ topics below 60%, run diagnostics on all gap topics in
parallel using `asyncio.gather` instead of sequentially. Measure the latency
improvement. Pass each diagnostic result with its `assessment_id` to the
recommendation step.

**Stretch 2 — Confidence-based routing (Domain 5, Task 5.5)**
Add a `confidence` field to `tool_run_diagnostic` results (0.0–1.0, representing
how reliable the 5-question diagnostic is). Route sessions where the average
diagnostic confidence is below 0.6 to human review — the sample size is too
small to trust. Include a stratified breakdown in the handoff:
high-confidence gaps vs low-confidence gaps.

**Stretch 3 — Scratchpad persistence (Domain 5, Task 5.4)**
Write the session's `diagnostics_run` and `resources_recommended` to a
`sessions/` directory after each session (`S001_session.json`, etc.).
On the next run, load prior session data and inject it into the `student_facts`
block: "In your last session, you were working on dictionaries (score: 47)."
This simulates crash recovery and session continuity.

**Stretch 4 — MCP resources (Domain 2, Task 2.4)**
Expose the `RESOURCES` catalogue as an MCP resource instead of baking it into
the tool implementation. The agent should be able to discover available topics
without making a tool call. Modify `tool_recommend_resource` to use
`client.beta.mcp.resources.get("resource://school/topics")` instead of
directly importing from `school_db`.

---

## Exam concept map

| Exam task statement | Implemented in |
|---|---|
| 2.1 — Tool descriptions: boundaries, examples, differentiating similar tools | Part 1: `get_student_history` vs `run_diagnostic_assessment` descriptions |
| 2.2 — Structured error responses: `errorCategory`, `isRetryable`, typed errors | Part 2: all four mock tool implementations |
| 2.2 — Access failures vs valid empty results | Part 2: `tool_recommend_resource` — `isError: False` with `found: False` |
| 2.3 — Tool distribution: scoped access, prevent cross-role misuse | Part 2: `AGENT_TOOLS` subset; `tool_choice` forced first call |
| 2.3 — `tool_choice` forced selection | Part 2: `start_session()` forces `get_student_history` |
| 5.1 — Persistent case facts block outside summarised history | Part 3: `student_facts` prepended every turn |
| 5.1 — Trimming verbose tool outputs | Part 3: `trim_tool_result()` |
| 5.2 — Escalation triggers: explicit request, policy gap, inability to progress | Part 3: `should_escalate()` three conditions |
| 5.2 — Structured handoff for human without transcript access | Part 3: `build_escalation_handoff()` |
| 5.3 — Error propagation: structured context, retry vs skip vs escalate | Part 3: `handle_tool_error()` |
| 5.5 — Confidence-based human review routing | Stretch 2 |
| 5.4 — Scratchpad persistence across sessions | Stretch 3 |
| 5.6 — Information provenance: source_assessment_id on recommendations | Part 3: `resources_recommended` list |
