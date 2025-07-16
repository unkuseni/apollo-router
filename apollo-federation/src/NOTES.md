## Thought Process for Merging Repeatable Directives

When approaching this problem, I started by understanding how GraphQL handles directives. The key requirements were:

    Non-repeatable directives should be overridden by new definitions

    Repeatable directives should accumulate both old and new versions

    Original directive order must be preserved

Merging Repeatable Directives

### Requirements
1. **Non-repeatable directives**: New definitions override existing ones.
2. **Repeatable directives**: Accumulate old and new versions.
3. **Order preservation**: Original directive order must be maintained.

### Approaches Considered

#### 1. Naive Accumulation (Flawed)
```rust
result.extend(existing.iter().cloned());
result.extend(new_defs.iter().cloned());
result.retain(|def| def.repeatable || seen.insert(def.name.clone()).is_none());
```
❌ Fails: Overwrites last occurrence, not necessarily `new_defs`.

#### 2. HashMap for Lookups
❌ Overcomplicates logic; risks incorrect overwrites.

#### 3. BTreeMap for Ordering
❌ Lexicographical order ≠ original position.

#### 4. **Two-Pass Processing (Chosen Solution)**
✅ Efficient (O(n)), preserves order, ensures `new_defs` precedence.

### Final Implementation
```rust
fn merge_repeatable_directives(
    existing: &[DirectiveDefinition],
    new_defs: &[DirectiveDefinition],
) -> Vec<DirectiveDefinition> {
    let mut result = Vec::new();
    let mut non_repeatable_indices = HashMap::new();

    // First pass: Track non-repeatable directives
    for (i, def) in existing.iter().enumerate() {
        if !def.repeatable { non_repeatable_indices.insert(def.name.clone(), i); }
        result.push(def.clone());
    }

    // Second pass: Overwrite or append
    for new_def in new_defs {
        if new_def.repeatable {
            result.push(new_def.clone());
        } else if let Some(&index) = non_repeatable_indices.get(&new_def.name) {
            result[index] = new_def.clone();
        } else {
            result.push(new_def.clone());
        }
    }
    result
}
```

### Key Advantages
- **Correct**: `new_defs` takes precedence for non-repeatable directives.
- **Efficient**: O(n) time complexity.
- **Order-safe**: Original positions preserved.

### Edge Cases
- Non-repeatable duplicates → first occurrence replaced.
- Repeatable directives → all versions retained.
- New directives → appended if not in `existing`.
