# CooperativeCoding JSON Schemas

Machine-readable validation schemas for `ccoding` metadata in JSON Canvas files.

## Files

| Schema | Validates |
|--------|-----------|
| `ccoding-canvas.schema.json` | Top-level `ccoding` object on the canvas |
| `ccoding-node.schema.json` | `ccoding` object on individual nodes |
| `ccoding-edge.schema.json` | `ccoding` object on individual edges |

## Usage

These schemas validate the `ccoding` metadata namespace, not the full JSON Canvas file. Apply each schema to the corresponding `ccoding` object extracted from a canvas, node, or edge.

### Example: Validate with Python

```python
import json
from jsonschema import validate

with open("spec/schema/ccoding-node.schema.json") as f:
    schema = json.load(f)

node_metadata = {
    "kind": "class",
    "qualifiedName": "models.User",
    "status": "accepted",
    "source": "src/models/user.py"
}

validate(instance=node_metadata, schema=schema)  # raises on invalid
```

### Example: Validate with ajv (Node.js)

```javascript
import Ajv from "ajv";
const ajv = new Ajv();
const validate = ajv.compile(require("./ccoding-node.schema.json"));

const valid = validate({
  kind: "class",
  qualifiedName: "models.User",
  status: "accepted",
});
```

## Conditional Rules

The schemas encode key spec constraints:

- **proposed status** requires `proposedBy` to be `"agent"` or `"human"`
- **stale status** requires `staleReason` on nodes
- **source paths** must be relative (no leading `/`) and must not contain path traversal (`..`)
- **kind** is required on all code-element nodes
- **relation** is required on all edges with ccoding metadata

## Extensibility

All schemas use `"additionalProperties": true` to support forward compatibility. Implementations MUST preserve unknown fields.
