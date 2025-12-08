---
title: "TOMOYO Domain transitions"
date: "2025-03-03"
tags: ["linux"]
ShowToc: true
---

How TOMOYO decides if a process can run a program (via `execve()`) and switch domains
**Example:** Process in `<kernel> /sbin/init` wants to run `/usr/bin/calc`

## Step 1: Obtain program pathname

- Get the program’s full path `[candidate]`
- can be a symlink
- **Example:**

  ```plaintext
  /usr/bin/calc → [candidate] = "/usr/bin/calc"
  ```

## Step 2: Check for aggregator directive

- Check exception policy for `aggregator [candidate] [pathname]`
- If found, replace `[candidate]` with `[pathname]`
- **Example:**

  ```plaintext
  aggregator /usr/bin/calc /bin/calculator → [candidate] = "/bin/calculator"
  ```

- Aggregator groups similar programs; if none, keep original `[candidate]`

## Step 3: Check for `file execute` directive

- Look in current domain’s policy (e.g., `<kernel> /sbin/init`) for `file execute [candidate] ...`
- **Options (sets `[destination]`, jumps to Step 5):**
- `file execute [candidate] keep` → Stay in `<kernel> /sbin/init`
- `file execute [candidate] child` → `<kernel> /sbin/init /usr/bin/calc`
- `file execute [candidate] reset` → `<kernel> /usr/bin/calc`
- `file execute [candidate] initialize` → `<kernel> /usr/bin/calc` (namespace root)
- `file execute [candidate] parent` → `<kernel>` (parent domain)
- `file execute [candidate] [domainname]` → Custom domain (e.g., `<kernel> /usr/bin/game`)
- `file execute [candidate] [pathname]` → `<kernel> /sbin/init /some/path`
- `file execute [candidate]` → No modifier, go to Step 4
- **Fail:** No rule + enforcing mode → Deny
- **Example:**  

  ```plaintext
  file execute /usr/bin/calc reset → [destination] = "<kernel> /usr/bin/calc"
  ```

## Step 4: Determine default domain transition

- If Step 3 didn’t set `[destination]`, check exception policy in order:
  
1. `reset_domain [candidate] from [source] (or "any") + no no_reset_domain` → `<kernel> /usr/bin/calc`, jump to Step 5
2. `initialize_domain [candidate] from [source] (or "any") + no no_initialize_domain` → `<kernel> /usr/bin/calc`, jump to Step 5
3. `keep_domain [candidate] from [source] (or "any") + no no_keep_domain` → `<kernel> /sbin/init`, jump to Step 5
4. **Default** → `<kernel> /sbin/init /usr/bin/calc`

- **Example:**  
  
  ```plaintext
  initialize_domain /usr/bin/calc → [destination] = "<kernel> /usr/bin/calc"
  ```

- Fallback plan if Step 3 was vague

## Step 5: Create destination domain

- Set up `[destination]` from Step 3 or 4
- **Failures:**
  - Step 3 rule, cannot be created → Deny
  - `reset_domain` → Deny
  - Step 4 default + enforcing mode → Deny
- Step 4 default + other mode → execution is continued but stays in current domain, mark `transition_failed` directive

## Step 6: Check environment variables

- Verify env vars (e.g., `PATH`) in `[destination]`
- if >1 env var is rejected + enforcing mode  → Deny
- **Example:**

  ```plaintext
  "LD_PRELOAD" rejected in "<kernel> /usr/bin/calc" → Deny if enforcing
  ```

## Step 7: Check interpreters

- If program needs an interpreter (e.g., `/bin/bash` for a script), check read permission in `[destination]`
- if read for interpreter denied + enforcing mode → Deny
- **Example:**

  ```plaintext
  "/usr/bin/calc" needs "/bin/bash", read in "/bin/bash" denied → Fail if enforcing
  ```

## Step 8: Execute the program

- Run the program, if successful, switch to `[destination]`
- **Example:** `/usr/bin/calc` runs → Process moves to `<kernel> /usr/bin/calc`
- Success = transition complete

## Domain transition flowchart

```plaintext
Start: "<kernel> /sbin/init"
1. Get "/usr/bin/calc"
2. Aggregator? → No
3. file execute?
    - Yes → Set [destination] → Jump to 5
    - No → 4. Defaults?
        - reset_domain → "<kernel> /usr/bin/calc"
        - initialize_domain → "<kernel> /usr/bin/calc"
        - keep_domain → "<kernel> /sbin/init"
        - Default → "<kernel> /sbin/init /usr/bin/calc"
5. Create [destination]
    - Yes → 6. Env vars okay?
        - Yes → 7. Interpreter okay?
            - Yes → 8. Run & transition SUCCESSFUL
            - No → DENY (enforcing)
        - No → DENY (enforcing)
    - No → DENY or stay in current domain
```

## Example scenario

- `<kernel> /sbin/init`, wants to transition to `/usr/bin/calc`
- Policy: `file execute /usr/bin/calc initialize`, enforcing mode
- **Steps:**
  1. `[candidate] = "/usr/bin/calc"`
  2. No aggregator
  3. `initialize` → `[destination] = "<kernel> /usr/bin/calc"`, jump to 5
  4. Create `<kernel> /usr/bin/calc`
  5. Env vars okay → Yes
  6. No interpreter issues → Yes
  7. Run → Transition to `<kernel> /usr/bin/calc`

## Attaching notes

- **Key Terms:**
  - `[candidate]`: Program path
  - `[destination]`: New domain
  - `[current_domain]`: Starting domain (e.g., `<kernel> /sbin/init`)
  - `[current_namespace]`: Root like `<kernel>`
- **Priority:** Step 3 > Step 4 (specific rules beat defaults)
- **Enforcing Mode:** Strict—denies on failures
