## Thought Process for Merging Repeatable Directives

When approaching this problem, I started by understanding how GraphQL handles directives. The key requirements were:

    Non-repeatable directives should be overridden by new definitions

    Repeatable directives should accumulate both old and new versions

    Original directive order must be preserved

I considered a few approaches:

#### Normal accumulations: Push all directives from `existing` and `new_defs` into the result, then deduplicate non-repeatable ones.
```rust
    let mut result = Vec::new();
    result.extend(existing.iter().cloned());
    result.extend(new_defs.iter().cloned());

    // Deduplicate non-repeatable directives
    let mut seen = HashMap::new();
    result.retain(|def| {
        if def.repeatable {
            true
        } else {
            seen.insert(def.name.clone(), ()).is_none()
        }
    });
```
it fails because:
Overwrites the **last occurrence** of a non-repeatable directive, not necessarily the one from `new_defs`.
 Does not guarantee that `new_defs` takes precedence.



 ####   Using a HashMap for quick lookups:

        Pros: Fast O(1) lookups for replacements

        Cons:Overcomplicates the logic.
         May incorrectly overwrite directives if `result` is modified during iteration.

        Verdict: Not suitable since order matters

  ```rust
  let mut result = Vec::new();
  let mut latest_non_repeatable = HashMap::new();

  // Process `existing`
  for def in existing {
      if !def.repeatable {
          latest_non_repeatable.insert(def.name.clone(), def.clone());
      }
      result.push(def.clone());
  }

  // Process `new_defs`
  for new_def in new_defs {
      if new_def.repeatable {
          result.push(new_def.clone());
      } else {
          latest_non_repeatable.insert(new_def.name.clone(), new_def.clone());
      }
  }

  // Overwrite non-repeatable directives in `result`
  for def in &mut result {
      if !def.repeatable {
          if let Some(latest) = latest_non_repeatable.get(&def.name) {
              *def = latest.clone();
          }
      }
  }
```

  ####  Using a BTreeMap for ordering:

        Pros: Maintains key ordering

        Cons: Lexicographical order isn't the same as original position

        Verdict: Doesn't match our position preservation needs

   #### Two-pass processing:

        First pass: Record positions of non-repeatable directives

        Second pass: Replace existing non-repeatables and append new ones

        Pros: Maintains order efficiently

        Correctly overwrites non-repeatable directives with `new_defs`.
         Accumulates repeatable directives as expected.

        Cons: Slightly more complex.

         If `new_defs` contains a non-repeatable directive not in `existing`, it is added (correct).
         If `existing` has a repeatable directive and `new_defs` has the same one, both are retained (correct).

```rust
        let mut result = Vec::new();
        let mut non_repeatable_indices = HashMap::new();

        // First pass: Track non-repeatable directives from `existing`
        for (i, def) in existing.iter().enumerate() {
            if !def.repeatable {
                non_repeatable_indices.insert(def.name.clone(), i);
            }
            result.push(def.clone());
        }

        // Second pass: Overwrite non-repeatable directives from `new_defs`
        for new_def in new_defs {
            if new_def.repeatable {
                result.push(new_def.clone());
            } else if let Some(&index) = non_repeatable_indices.get(&new_def.name) {
                result[index] = new_def.clone();
            } else {
                result.push(new_def.clone());
            }
        }
```

The Two-pass approach is the most reliable:
1. First pass: Track non-repeatable directives from `existing`.
2. Second pass:
   - Overwrite non-repeatable directives from `new_defs`.
   - Accumulate repeatable directives.

```rust
fn merge_repeatable_directives(
    existing: &[DirectiveDefinition],
    new_defs: &[DirectiveDefinition],
) -> Vec<DirectiveDefinition> {
    let mut result = Vec::new();
    let mut non_repeatable_indices = HashMap::new();

    // First pass: Track non-repeatable directives from `existing`
    for (i, def) in existing.iter().enumerate() {
        if !def.repeatable {
            non_repeatable_indices.insert(def.name.clone(), i);
        }
        result.push(def.clone());
    }

    // Second pass: Overwrite non-repeatable and accumulate repeatable
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


The breakthrough came when I realized we should:

    Maintain the original list of directives

    Only replace non-repeatable directives in-place when found

    Always append repeatable directives to preserve their multiple instances

    Add new directives not found in the original list

    Final Answer**
    The **two-pass approach** is the correct solution because:
    1. It guarantees `new_defs` takes precedence for non-repeatable directives.
    2. It retains all repeatable directives.
    3. It is **efficient** (O(n) time complexity).

    Here’s the final implementation and test:

This handles edge cases like:

    Multiple non-repeatable directives with the same name (replace first occurrence)

    New repeatable directives added after existing ones

    Mixed repeatable/non-repeatable with same name (treated separately)


The final implementation processes directives in their original order while efficiently handling replacements and accumulations. It preserves the GraphQL specification requirements while being computationally efficient (O(n) complexity).
