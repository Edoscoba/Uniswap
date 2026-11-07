# How to write each finding field

Every finding is submitted as separate fields, and this applies to humans, AI auditors and base models alike. Write each field so a developer who knows the protocol, but has not seen this bug, can understand it, confirm it and fix it.

## Rules for all fields

- Do not add headings inside a field, and do not repeat the field's name. The platform adds a heading to each field.
- Put each piece of content in its own field. Do not put the attack path or the fix in `description`.
- Make each finding stand on its own, and do not refer to other findings.
- Keep one idea in each sentence, and aim for under 20 words.
- Use the code's own names in backticks. Explain any term or abbreviation you introduce.
- Use real numbers in place of words like "significant" or "large".
- Bold labels such as **Limits:** are fine where this guide asks for them. Code blocks are fine.
- A base model's submission is rejected when it lacks a labelled part this guide asks for, or is far off the guide. The rejection says what to fix.

## Field guidance

The examples below all describe one fictional vault, `SampleVault`. It is not part of any codebase, so do not submit it as a finding.

### `title`

Write one line in plain words, saying what breaks and what it causes. Lead with the effect, not the function name.

Example: First depositor can steal the next deposit by inflating the share price

### `summary`

Write three sentences at most: what is wrong, who is affected and what they lose. A reader should understand the issue from this field alone. Leave code snippets and line numbers for `description`.

Example: The vault prices shares from its token balance, which anyone can raise with a direct transfer. The first depositor can use this to make the next deposit round down to zero shares. The next depositor loses their full deposit to the attacker.

### `description`

This is the root cause. Start with the location: file, function and lines. Show the short code snippet that matters in a code block. Then explain what the code does and what it should do. Stop there, because the steps belong in `attack_path` and the fix belongs in `recommendation`.

Example:

`SampleVault.sol`, `deposit()`, lines 42-47.

```solidity
shares = supply == 0 ? assets : assets * supply / totalAssets();
```

`totalAssets()` counts every token the vault holds, including tokens sent to it directly. The share calculation also rounds down, and `deposit()` does not reject a result of zero. The vault should track deposits in a way a direct transfer cannot exploit.

### `impact`

Say who loses what and how much, with a worked number where funds are involved. If no funds are at risk, say what stops working, for whom and for how long. End with **Limits:** and anything that caps the damage.

Example:

The next depositor loses their full deposit. With a 10,000 USDC deposit, the victim loses 10,000 USDC and the attacker gains the same amount.

**Limits:** The attack only works while the vault is empty or nearly empty.

### `attack_path`

Start with **Preconditions:** and state what must be true first. Then write numbered steps: who acts, what they call and what changes. Use real values. If nobody needs to attack, say so in the preconditions and list the steps that trigger the failure.

Example:

**Preconditions:** The vault has no shares, and the attacker deposits before anyone else.

1. The attacker deposits 0.000001 USDC and receives 1 share.
2. The attacker transfers 10,000 USDC directly to the vault.
3. A victim deposits 10,000 USDC, which rounds down to 0 shares.
4. The attacker redeems their 1 share and receives all 20,000.000001 USDC.

### `recommendation`

Give the specific change to make, with a short diff when it helps. Name every other place the same fix is needed. End with **Trade-off:** and anything the team should know about this fix, or "None".

Example:

Add a virtual share offset and reject deposits that mint zero shares. Use the same offset in `redeem()`.

```diff
- shares = supply == 0 ? assets : assets * supply / totalAssets();
+ shares = assets * (supply + 1e3) / (totalAssets() + 1);
+ require(shares > 0, "ZERO_SHARES");
```

**Trade-off:** A tiny part of the first deposit's value stays with the virtual shares.

### `proof_of_concept`

Give a runnable test that proves the bug. Then add **Run:** with the exact command, and **Expected result:** with what the test shows. Only describe results you actually got. If you have no PoC, leave this field empty.

Example, after the test code:

**Run:** `forge test --match-test test_firstDepositorSteal -vv`

**Expected result:** The test passes. The victim holds 0 shares and the attacker ends with the victim's 10,000 USDC.

### `assumptions_and_uncertainties`

This field is optional. List what you assumed, what you could not verify and what depends on the deployment. Arguments that the finding is in scope or distinct from a known issue also go here. Keep it to a few lines, and leave it empty if there is nothing to add.

Example: This assumes the vault launches with no seed deposit. I could not confirm the deployment order from the repo.
