`core/schemas/envelope.json`

```JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GOSMessageEnvelope",
  "type": "object",
  "required": ["gos", "from", "to", "timestamp", "payload"],
  "properties": {
    "gos": {
      "type": "string",
      "description": "The GOS specification version enforcing this message structure.",
      "pattern": "^[0-9]+\\.[0-9]+$"
    },
    "from": {
      "type": "string",
      "description": "The unique logical identity of the sending entity (GUID or Ghost MAC).",
      "pattern": "^[a-zA-Z0-9_\\-:\\.]+$"
    },
    "to": {
      "type": "string",
      "description": "The target identity, broadcast group, or logical service address.",
      "pattern": "^[a-zA-Z0-9_\\-:\\.\\*]+$"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 UTC timestamp reflecting message generation time."
    },
    "meta": {
      "type": "object",
      "description": "Optional operational metadata. Reserved for trace IDs, routing telemetry, or security nonces.",
      "additionalProperties": true
    },
    "payload": {
      "type": "object",
      "description": "The isolated transactional container housing operational capabilities, commands, or telemetry.",
      "additionalProperties": true
    }
  },
  "additionalProperties": true
}
```