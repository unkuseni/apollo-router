## MY THOUGHT PROCESS FOR MERGE REPEATABLE DIRECTIVES

 I first analysed the codebase to identify how it works then I started to think about how to implement the core requirements.

### Implementation Details

 Non-repeatable directives should be overridden by new definitions
Repeatable directives should be accumulated (both old and new preserved)
Need to maintain type definitions with `DirectiveDefinition` struct
Order of directives should be preserved

### My Decision

#### HashMap vs BTreeMap

 Key Characteristics**:
     `BTreeMap`: Ordered by keys (lexicographical order for `Name`)
    `HashMap`: Unordered but faster average lookups (O(1) vs O(log n))

 2. **Why `BTreeMap` could work**:
     `Name` implements `Ord` (required for `BTreeMap`)
    Small number of directives makes performance difference negligible
     Deterministic ordering might be beneficial for debugging


  Decided to go with HashMap, Why??

  1. The current implementation doesn't care about key ordering
  2. `HashMap` is more idiomatic for simple key-value lookups
  3. The dataset (directives) is typically small (< 100 items)
  4. No ordering requirements exist in the specification


  HashMap for Non-Repeatable Tracking:
    stores last index of each non-repeatable directive for efficient replacement
    only tracks non-repeatable directives since repeatables don't need replacement


  Two-Pass Processing:
  process existing directives (record positions of non-repeatables)
   process new directives (replace non-repeatables or add repeatables)

  Preserving Order:
   maintains original order of existing directives
   new repeatable directives are appended to the end
   non-repeatable replacements stay in their original positions


### alternative approaches I considered
```rust
  let mut merged: HashMap<Name, DirectiveDefinition> = HashMap::new();

  // Process existing
  for def in existing {
      if !def.repeatable {
          merged.insert(def.name.clone(), def);
      } else {
          // How to handle multiple repeatables?
      }
  }
  ```

  issue with this idea: loses ordering and doesn't handle multiple repeatable instances well.

  ```rust
  let non_repeatable_names: HashSet<_> = new_defs.iter()
      .filter(|d| !d.repeatable)
      .map(|d| &d.name)
      .collect();

  let mut result: Vec<_> = existing
      .into_iter()
      .filter(|d| d.repeatable || !non_repeatable_names.contains(&d.name))
      .collect();

  result.extend(new_defs);
  ```

  issues with this: loses original positions of replaced directives, new non-repeatables appear at end rather than original position, repeatables from new_defs always appear after existing


  edge cases:
  new non-repeatables appear at end rather than original position
  multiple non-repeatables with the same name: only the lat one is kept
  mixed repeatable/non-repeatable with the same name: treated as different directives

  ### Final insights

   Preserving original directive order
   Efficient replacement of non-repeatables
   Simple accumulation of repeatables
   Clear handling of edge cases
