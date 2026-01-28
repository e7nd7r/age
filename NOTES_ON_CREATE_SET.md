# ON CREATE SET / ON MATCH SET Feature Analysis

This document analyzes the implementation effort required to add `ON CREATE SET` and `ON MATCH SET` support to Apache AGE's MERGE clause, matching Neo4j/Memgraph behavior.

**Related Issue**: https://github.com/apache/age/issues/1619

## Current Behavior

```cypher
-- This works:
MERGE (n:Person {name: 'Alice'})
SET n.updated_at = timestamp()

-- This does NOT work (Neo4j syntax):
MERGE (n:Person {name: 'Alice'})
ON CREATE SET n.created_at = timestamp()
ON MATCH SET n.updated_at = timestamp()
```

## Files That Need Changes

### 1. Parser Grammar (`src/backend/parser/cypher_gram.y`)

**Current grammar** (lines 1133-1142):
```c
merge:
    MERGE path
        {
            cypher_merge *n;
            n = make_ag_node(cypher_merge);
            n->path = $2;
            $$ = (Node *)n;
        }
    ;
```

**Proposed grammar**:
```c
merge:
    MERGE path merge_actions_opt
        {
            cypher_merge *n;
            n = make_ag_node(cypher_merge);
            n->path = $2;
            n->on_create = $3.on_create;
            n->on_match = $3.on_match;
            $$ = (Node *)n;
        }
    ;

merge_actions_opt:
    /* empty */
        {
            $$.on_create = NIL;
            $$.on_match = NIL;
        }
    | merge_actions
    ;

merge_actions:
    on_create_clause
        {
            $$.on_create = $1;
            $$.on_match = NIL;
        }
    | on_match_clause
        {
            $$.on_create = NIL;
            $$.on_match = $1;
        }
    | on_create_clause on_match_clause
        {
            $$.on_create = $1;
            $$.on_match = $2;
        }
    | on_match_clause on_create_clause
        {
            $$.on_create = $2;
            $$.on_match = $1;
        }
    ;

on_create_clause:
    ON CREATE SET set_item_list
        {
            $$ = $4;
        }
    ;

on_match_clause:
    ON MATCH SET set_item_list
        {
            $$ = $4;
        }
    ;
```

**Note**: Need to add `ON` keyword token. `MATCH`, `CREATE`, `SET` already exist.

### 2. Node Structure (`src/include/nodes/cypher_nodes.h`)

**Current** (lines 123-127):
```c
typedef struct cypher_merge
{
    ExtensibleNode extensible;
    Node *path;
} cypher_merge;
```

**Proposed**:
```c
typedef struct cypher_merge
{
    ExtensibleNode extensible;
    Node *path;
    List *on_create;  /* List of cypher_set_item for ON CREATE SET */
    List *on_match;   /* List of cypher_set_item for ON MATCH SET */
} cypher_merge;
```

### 3. Serialization Functions

#### `src/backend/nodes/cypher_copyfuncs.c`
Add to `copy_cypher_merge()`:
```c
COPY_NODE_FIELD(on_create);
COPY_NODE_FIELD(on_match);
```

#### `src/backend/nodes/cypher_outfuncs.c`
Add to `out_cypher_merge()`:
```c
WRITE_NODE_FIELD(on_create);
WRITE_NODE_FIELD(on_match);
```

#### `src/backend/nodes/cypher_readfuncs.c`
Add to `read_cypher_merge()`:
```c
READ_NODE_FIELD(on_create);
READ_NODE_FIELD(on_match);
```

### 4. Analyzer (`src/backend/parser/cypher_clause.c`)

The `transform_cypher_merge()` function needs to:
1. Transform `on_create` set items into executable form
2. Transform `on_match` set items into executable form
3. Store them in `cypher_merge_information` for the executor

Need to update `cypher_merge_information` struct in `cypher_nodes.h` (lines 473-480):
```c
typedef struct cypher_merge_information
{
    ExtensibleNode extensible;
    uint32 flags;
    int merge_function_attr;
    cypher_create_path *path;
    Oid graph_oid;
    List *on_create_set;  /* NEW: transformed SET items for ON CREATE */
    List *on_match_set;   /* NEW: transformed SET items for ON MATCH */
} cypher_merge_information;
```

### 5. Executor (`src/backend/executor/cypher_merge.c`)

The key logic is in `exec_cypher_merge()` around line 627:

```c
// Current logic:
if (check_path(css, econtext->ecxt_scantuple))
{
    // Path NOT found -> create it
    process_path(css, prebuilt_path_array, true);
    // NEW: Apply ON CREATE SET here
    apply_merge_set_items(css, css->merge_information->on_create_set, econtext);
}
else
{
    // Path found
    // NEW: Apply ON MATCH SET here
    apply_merge_set_items(css, css->merge_information->on_match_set, econtext);
}
```

Need to implement `apply_merge_set_items()` - can likely reuse logic from `cypher_set.c`.

### 6. Custom Scan State (`src/include/executor/cypher_utils.h`)

The `cypher_merge_custom_scan_state` may need additional fields if SET execution requires state.

## Estimated Effort

| Component | Files | Lines | Complexity |
|-----------|-------|-------|------------|
| Parser grammar | 1 | ~50 | Medium |
| Node structure | 1 | ~10 | Simple |
| Serialization | 3 | ~30 | Simple |
| Analyzer | 1 | ~100 | Medium |
| Executor | 1 | ~150 | Medium-Hard |
| Regression tests | 1-2 | ~200 | Simple |

**Total**: ~500-600 lines across 7-8 files

## Test Cases to Add

Location: `regress/sql/` and `regress/expected/`

```cypher
-- Basic ON CREATE SET
MERGE (n:Person {name: 'Alice'})
ON CREATE SET n.created = true
RETURN n;

-- Basic ON MATCH SET
MERGE (n:Person {name: 'Alice'})
ON MATCH SET n.visits = coalesce(n.visits, 0) + 1
RETURN n;

-- Both clauses
MERGE (n:Person {name: 'Alice'})
ON CREATE SET n.created_at = timestamp()
ON MATCH SET n.updated_at = timestamp()
RETURN n;

-- With relationships
MERGE (a:Person {name: 'Alice'})-[r:KNOWS]->(b:Person {name: 'Bob'})
ON CREATE SET r.since = date()
ON MATCH SET r.strength = coalesce(r.strength, 0) + 1
RETURN r;

-- Multiple SET items
MERGE (n:Person {name: 'Alice'})
ON CREATE SET n.a = 1, n.b = 2
ON MATCH SET n.c = 3, n.d = 4
RETURN n;
```

## References

- Neo4j MERGE documentation: https://neo4j.com/docs/cypher-manual/current/clauses/merge/
- Existing SET implementation: `src/backend/executor/cypher_set.c`
- Existing MERGE implementation: `src/backend/executor/cypher_merge.c`
