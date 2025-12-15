# A2A Protocol Extension: Input/output schemas (v1)

- **URI**: https://raw.githubusercontent.com/facultyai/a2a-extension-object-schemas/refs/heads/main/v1
- **Type**: Profile Extension / Data-only Extension
- **Version**: 1.0.0

# Abstract

This extension enables an Agent2Agent (a2a) agent to declare JSONSchemas for structured data, which can be used as inputs or outputs for the agent and/or specific skills.

Schemas used for an input mode will, if received, mandate the triggering of a specific task. There is no prerequisite as to _which_ task is triggered, and there is no formal correlation between agents/skills and tasks created by this specification.

These schemas enable agents to:

- Ensure that they receive all the data that might be required to properly execute a task upfront
- Behave in a consistent manner around how that task is executed
- Better understand the format of output JSON from a task when parsing the response
- Drive the implementation of a form-based UX for submitting requests to an agent

This last point is the main differentiator of the spec, which would otherwise look like it's reimplementing MCP over a2a. It also does not preclude messages being sent in a format other than data parts that conform to the JSONSchema.

# Implementation

## 1. Agent Card Extension

An A2A Agent that is capable of receiving and utilizing structured data from schemas MUST declare its support in the `AgentCard` under the `extensions` part of the `AgentCapabilities` object.

It can then declare a new root key under the `AgentCard` object, `schemas`. This is a key->value store of `"schemaName": JSonSchema`, which declares a named schema.

These named schemas can then be included as an `inputMode` or `outputMode` either for the agent itself or for a specific skill under the `AgentSkill[]` array. The modes should be referenced as `application/json;schema=<name-of-schema>`. It is **highly recommended** that the `inputMode` also includes `text/plain` and that core agentic language communication is still possible, this extension should be treated as a graceful upgrade to capabilities rather than an override.

### Example AgentCard declaration

This partial implementation of an `AgentCard` declares a skill based around comparing two contestants to see which one would win in a fight, by:

1. Declaring the extension
2. Declaring the `fightComparison` schema.
3. Declaring that the `fight-comparison` skill can accept both plaintext input, as well as JSON using the `fightComparison` schema.

```json
{
  "capabilities": {
    "extensions": [
      {
        "uri": "https://raw.githubusercontent.com/facultyai/a2a-extension-object-schemas/refs/heads/main/v1",
        "description": "Allow more deterministic invoking of tasks by providing JSON input corresponding to a provided schema"
      }
    ]
  },
  "schemas": {
    "fightComparison": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "a": {
          "description": "The name of the first contestant",
          "type": "string"
        },
        "b": {
          "description": "The name of the second contestant",
          "type": "string"
        }
      },
      "required": ["a", "b"],
      "additionalProperties": false
    },
    "fightResponse": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "winner": {
          "title": "Winner",
          "description": "The name of the winner of the fight",
          "type": "string"
        },
        "probability": {
          "title": "Probability",
          "description": "The probability of the winner winning the fight",
          "type": "number",
          "minimum": 0,
          "maximum": 1
        },
        "explanation": {
          "title": "Explanation",
          "description": "A freeform field justifying the reason for choosing the winner",
          "type": "string"
        }
      },
      "required": ["winner", "probability", "explanation"],
      "additionalProperties": false
    }
  },
  "skills": [
    {
      "id": "fight-comparison",
      "name": "Fight Comparison",
      "description": "Determines who would win in a hypothetical fight between two contestants. Requires a data payload with two fields: \"a\" (first contestant) and \"b\" (second contestant). Returns the winner, probability of victory, and an explanation.",
      "inputModes": ["text/plain", "application/json;schema=fightComparison"],
      "outputModes": ["text/plain", "application/json;schema=fightResponse"],
      "examples": [
        "{\"a\": \"Superman\", \"b\": \"Batman\"}",
        "{\"a\": \"Lion\", \"b\": \"Tiger\"}",
        "{\"a\": \"Godzilla\", \"b\": \"King Kong\"}"
      ],
      "tags": ["comparison", "fight", "ai-analysis", "prediction"]
    }
  ]
}
```

## 2. User message extension

In order to trigger this structured message flow, a message from the user to this a2a agent MUST:

- Contain a part of `kind: "data"`, which has the data set to a JSON object which corresponds to one of the declared schemas.
- Have a `mimeType` property in the `metadata` object which is set to the declared `inputMode` for the at schema.

### Example user message

This user message declares that the data being sent corresponds to the `fightComparison` schema and should be treated as such:

```json
{
  "kind": "message",
  "messageId": 1,
  "role": "user",
  "parts": [
    {
      "kind": "data",
      "data": {
        "a": "100 duck sized horses",
        "b": "1 horse sized duck"
      },
      "metadata": {
        "mimeType": "application/json;schema=fightComparison"
      }
    }
  ]
}
```

## 3. User request flow

An a2a Agent that declares this extension MUST follow this flow when handling a message from the user:

- If the user message contains a part which can be flagged for special processing (ie: The part is `"kind": "data"` and has a `mimeType` metadata property set to `application/json;schema=<name>`)
  - If the name of the schema has not been declared in the agent card, set a structured input error for this user message to indicate this.
  - If the name of the schema has been declared in the agent card, validate the data from this part against the schema provided.
    - If the data passes validation (ie: The data corresponds to the schema), set this as the structured input for this user message.
    - If the data does not pass validation (ie: The data does not correspond to the schema), set this as a structured input error for this user message.
- If the user message contains multiple parts which can be flagged for special processing, take only the first found part, ignore the rest.
- If the user message does not contain any parts which can be flagged for special mprocessing,

After this special processing step:

- If the user message has no existing task associated with it:

  - If a structured input has been set for this user message, the response from this user message **MUST** be a task which uses this input as the basis to run it.
  - If a structured input error has been set for this user message, the response from this user message **SHOULD** indicate that the data supplied wasn't valid.
  - If neither a structured input or structured input error has been set for this user message, the response from this user message is down to individual implementation.

- If the user message has an existing task associated with it:
  - If either a structured input or structured input error has been set for this user message, the response from this user message **MUST** indicate that the data has been rejected because a task is already running.
  - If neither a structured input or structured input error has been set for this user message, the response from this user message is down to individual implementations.

## 4. Task response

If a task returned by an a2a agent wants to make use of a declared schema for detailing the structure of a data part included as part of the task's artifacts, it should do so by setting the `mimeType` metadata property to `application/json;schema=<name>`

### Example task response

```json
{
  "id": "a20a3b55-799a-4eeb-b9dc-f68014b7f031",
  "kind": "task",
  "contextId": "0d94b771-6b14-41a0-8425-89533e3edbf5",
  "status": {
    "state": "completed",
    "timestamp": "2025-12-15T08:36:24.728Z"
  },
  "history": [
    {
      "kind": "message",
      "messageId": "79948407-2588-4add-b823-5a890285ca6f",
      "role": "user",
      "contextId": "0d94b771-6b14-41a0-8425-89533e3edbf5",
      "parts": [
        {
          "kind": "data",
          "data": {
            "a": "Lion",
            "b": "Tiger"
          },
          "metadata": {
            "mimeType": "application/json;schema=fightComparison"
          }
        }
      ],
      "taskId": "a20a3b55-799a-4eeb-b9dc-f68014b7f031"
    }
  ],
  "metadata": {
    "title": "Who would win in a fight between Lion and Tiger?"
  },
  "artifacts": [
    {
      "artifactId": "fight-result",
      "parts": [
        {
          "kind": "data",
          "data": {
            "winner": "Tiger",
            "probability": 0.65,
            "explanation": "Tigers are generally larger and heavier than lions, with stronger muscle mass and more powerful forelimbs. In a one-on-one encounter, a tiger's solitary hunting habits also give it an advantage in terms of stealth and strength."
          },
          "metadata": {
            "mimeType": "application/json;schema=fightResponse"
          }
        }
      ]
    }
  ]
}
```
