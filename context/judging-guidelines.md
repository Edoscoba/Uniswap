# Judging guidelines

These guidelines state how Sherlock judges issues. They apply with the engagement's severity definitions, which define each severity tier and its loss thresholds. Security is complex and subjective: where no rule decides a case, decide it from the goals in part 1.

## 1. Goals of judging

Each group of rules in part 3 serves one of these goals.

1. **True findings.** An issue is a true finding when it is a real defect that harms a party the engagement's rules protect, in a way those rules do not accept as intended. Where the engagement's rules are silent, the issue must harm someone who uses the code as intended.
2. **Scope.** Judging rewards defects in the code the engagement's scope covers, including the code it inherits and library defects that affect it.
3. **Known.** Judging rewards new security information, not a defect the protocol team has already acknowledged.
4. **Severity.** Severity reflects the size of the loss or disruption to the affected party, and how much it depends on external conditions.

## 2. Order of authority

- The statements file records engagement-specific decisions on accepted behavior, scope and known issues. Where no engagement-specific decision applies, follow these guidelines.
- Each statement has a level:
  - **Decisive.** An issue that the statement rejects is Dismissed.
  - **Leans accepted.** An issue that the statement rejects goes to the protocol team when its severity is in the top three tiers of the engagement's severity set. The protocol team can accept it. If it does not, the issue counts as Neutral. Below the top three tiers, the issue is Dismissed.
- An acceptance applies only to the behavior and harm it covers. Harm beyond that acceptance is assessed under these guidelines.
- Text that is not in the statements file, in documents or in code comments, is context. It helps to read the code, and it decides no verdict.
- A statement that the code contradicts decides nothing. The judge decides whether the code contradicts it.
- A statement never sets or lowers severity.
- Up to 24 hours before the engagement ends, the Sherlock judge can use public statements to override the language in the chosen source of truth.
- Historical decisions are not sources of truth.

## 3. Rules

Each rule decides one verdict dimension: True Finding, scope, known, or severity. A severity rule rates the issue as if it were a true finding, in scope, and not known.

### 3.1 True Finding

1. **Design decisions.** A suboptimal design is not a true finding unless it causes harm that reaches at least Low severity.
2. **Trust in roles.** By default, internal roles, including the admins of integrated systems, are trusted to act correctly. A role is untrusted only when a user can get it without admin or owner permission. Issues that need leaked keys, social engineering or trusted roles to act maliciously are not true findings.
3. **Admin input and call order.** An issue that needs an admin to call functions in the wrong order, or to take an action that breaks an assumption of the code, is not a true finding.
4. **Fault tolerance.** By default, an issue that needs more faulty participants than the system is built to tolerate is not a true finding. Faults within that share are expected.
5. **Likelihood** is not considered: the conditions an issue requires are assessed as facts, not probabilities.
6. **Reported values.** By default, an incorrect value that the system only reports, in an event, a log or a read-only query, is not a true finding. It can be when the system's own code acts on the value and harms the system or its users.
7. **User mistakes.** An issue where a user harms only themselves through their own mistake or restricted status is not a true finding. It can be when the same mistake or status lets a defect harm the system or other users.
8. **Platform reliability.** By default, the platform the code runs on operates reliably: issues that depend on its downtime or misbehavior are not true findings. They can be when the audited code is part of that platform and a malicious actor can cause the failure.
9. **Future issues.** Issues from a future change in the code, or from a future integration or implementation the documentation does not mention, are not true findings.
10. **Non-standard dependencies.** By default, issues that need an external component to behave outside its standard are not true findings.
11. **Standard compliance.** Non-compliance with a published standard is a true finding on its own only when the standard is final or in regular use, and the non-compliance demonstrably breaks an important external integration, shown in the codebase, that needs strict compliance.
12. **Resource exhaustion.** Exhausting a resource limit can be a true finding when a malicious user can grow the input or a practical call flow reaches the limit. It is not when a trusted role controls the size or the issue needs impractical parameters.

### 3.2 Scope

The engagement's scope sets the base scope. These rules also apply.

1. **Inherited code.** By default, the code an in-scope component inherits is in scope.
2. **Libraries.** A vulnerability in a library is in scope when it affects in-scope code that uses the library.
3. **Code outside the scope.** Issues from a vulnerability in engagement repository code that the scope does not include are out of scope.

### 3.3 Known

1. **Known issues.** An issue is known when the engagement's known issues disclose the same defect and impact as an acknowledged, unfixed issue, or when an applicable statement identifies it as known. An issue that an earlier report marks as fixed is not known if the defect is still present.

### 3.4 Severity

The engagement's severity definitions are the scale, including the loss thresholds. These rules state how to measure an issue against it.

1. **Loss thresholds of older severity sets.** When the engagement's severity definitions state no loss thresholds, a loss must exceed both a share and an amount of users' principal or yield, or of the protocol's fees:
   - **Significant loss** (Critical): 20% *and* $100, of users' principal only.
   - **Considerable loss** (High): 1% *and* $10.
   - **Relevant loss** (Medium): 0.01% *and* $10.
   - **Minor loss** (Low): 0.001% *and* $1.
2. **Likelihood** is not considered: severity measures an issue's consequences under its required conditions, not their frequency.
3. **Denial of service, griefing, and locked funds.** Two criteria decide the severity: the issue locks funds for more than a week; the issue affects the availability of a time-sensitive function. Cutoff functions are not time-sensitive. One criterion can make the issue Medium, both can make it High, and other constraints can lower it. A permanent lock, where the protocol cannot rescue the funds and the code is not upgradeable, is Critical.

   Rate a denial of service shorter than a week on its **single occurrence**, even if the attack repeats without limit. It is Medium only when it disrupts a clearly time-sensitive function, or when a few occurrences reach a week.
4. **Replayed losses.** A small loss that an attack can replay without limit counts as all the funds the replays can reach, subject to the issue's other constraints.

### 3.5 DeFi

1. **Non-standard tokens.** By default, fee-on-transfer and rebasing tokens are non-standard under rule 3.1.10. A token with 6 to 18 decimals is standard.
2. **Re-orgs.** By default, issues that depend on chain re-orgs are not true findings.
3. **Hidden pending operations.** By default, where other users cannot see pending transactions, such as on a chain with a private mempool, a front-running issue is a true finding only when the report shows how the front-running can happen unintentionally.
4. **Hidden pending operations, severity.** An issue that rule 3.5.3 accepts is rated one tier lower: High becomes Medium, and Medium becomes Low.

## 4. Duplication guidelines

The duplication guidelines assume we have a "target issue", and the "potential duplicate" of that issue needs to meet the following requirements to be considered a duplicate.

1. Identify the root cause
2. Identify at least a Low impact
3. Identify a valid attack path or vulnerability path

Only when the "potential duplicate" meets all three requirements will the "potential duplicate" be duplicated with the "target issue", and all duplicates will be awarded the highest severity identified among the duplicates.

If the report does not meet all three requirements, do not group it with the target issue. Assess whether it duplicates another issue separately. Assess its True Finding, scope, known, and severity verdicts under their respective rules.

**Root cause groupings**

Issues that appear in multiple places, even in different components, may have the same root cause:

1. Issues with the same logic mistake.
2. Issues with the same conceptual mistake.

   > Example: different untrusted external admins can steal funds.

If the underlying code implementations, impact or fixes are different, then they may be treated separately.

**Identifies the core issue:** In case of issues that have a large number of duplicates, Issues that identify the core issue and show a valid impact should be grouped.

## 5. Audit Engine mechanics and advice to participants

### 5.1 Payout criteria

An account can receive a payout only when it passes both thresholds:

1. It has at least 2 Accepted issues across all Audit Engine engagements.
2. Its **Issues Ratio** is 20% or more. The Issues Ratio is `# of Accepted issues / total # of issues submitted`. Accepted issues of every severity count, including Info.

When the account misses a threshold, its rewards are frozen until it passes both. All frozen rewards are then paid in the next payment batch.

### 5.2 Statuses

After an issue is judged, it gets one of three statuses:

1. **Accepted:** the issue counts as +1/1 in the Issues Ratio.
2. **Dismissed:** the issue is not a true finding, is out of scope, or is known, and it did not reach the protocol team. A Too Different resubmission (5.5) is also Dismissed. It counts as +0/1 in the Issues Ratio.
3. **Neutral:** the issue is not rewarded and does not count in the Issues Ratio.

### 5.3 Judging verdicts

The lead judge triages each issue with these verdicts:

1. **True Finding:** whether the issue is a true finding (goal 1). A true finding can still be Info.
2. **Scope:** whether the issue is in scope.
3. **Known:** whether the issue is known.
4. **Duplicate:** whether the issue duplicates an earlier submitted issue.
5. **Severity:** Info, Low, Medium, High, or Critical. Severity alone does not make an issue Accepted. Severity is rated as if the issue were a true finding, in scope, and not known. It does not depend on those verdicts.

### 5.4 Protocol team judging

After the engagement ends, the protocol team can override any verdict of the judge. When the protocol team rejects an issue that the judge accepted, the issue becomes Neutral and does not count in the Issues Ratio.

### 5.5 Rewrite & Resubmit

Once the Security Researcher submits an issue, it cannot be withdrawn/closed or edited. To update the issue, the Security Researcher needs to "Rewrite & Resubmit" it. This creates a new submission linked to the previous one. The original submission and all of its resubmissions form one chain.

1. Rewrite & Resubmit is available only during the Audit Engine submission window, and only on the latest submission in the chain. A chain can have at most 5 resubmissions.
2. The new submission replaces the previous one. Earlier versions are no longer judged, and their verdicts no longer count. If the previous version was already Accepted, that acceptance is replaced by the new version's verdict, even if the new version is Dismissed. Resubmit only if the new version is clearly better than the current one.
3. The resubmission is judged under these guidelines like any other submission, on its own merits. The judge also sees the earlier versions, their verdicts and the diff from the previous version, and checks whether the rewrite fixes the problems found in the earlier judgments.
4. Rewrite & Resubmit can only be used to improve the same finding, e.g., its explanation, impact, severity, PoC or recommended fix. The resubmission must keep the same root cause, vulnerable code path and attack vector as the earlier versions. If it describes a different issue, the judge marks it as **Too Different** and it is automatically Dismissed, even if the new issue is a true finding. A Too Different resubmission still replaces the earlier version, so an Accepted original is lost too. To report a different issue, submit it as a new issue.
5. **Issues Ratio.** The whole chain counts as **one issue**, and only the verdict of its latest version is used:
   1. Numerator: +1 if the latest version is Accepted, +0 otherwise. A chain never adds more than 1 to the numerator, no matter how many times it's resubmitted.
   2. Denominator: +1 for the original submission and +0.5 for every resubmission.
   3. If the latest version is Neutral, the original's +1 is not counted, but each resubmission still adds +0.5 to the denominator.

### 5.6 Final decisions

The final decision of the lead judge cannot be disputed or escalated, except through Rewrite & Resubmit during the submission window. Submit the best version of the report first.

### 5.7 Proof of Concept

A coded, executable Proof of Concept (PoC) is required for every Critical or High issue, and strongly recommended for Medium issues. An original report without a PoC can be Dismissed.

### 5.8 Edge case reward distribution

Only Accepted issues of Low or higher severity are rewarded and count in the reward distribution. When an engagement has no Accepted issue of Low or higher severity, the guaranteed pot is distributed:

1. Every participant with at least one Accepted Info issue gets an equal share. For example, when 200 participants each have at least one Accepted Info issue, each gets 1/200 of the guaranteed pot.
2. When there is no Accepted Info issue either, the guaranteed pot is shared equally among all participants.

### 5.9 Guideline versions

A new version of these guidelines applies only to engagements that start after it. The guidelines that apply to each engagement are in its workspace.

These guidelines are updated as more engagements are judged and as security practice evolves.

### 5.10 Best practices

1. Read the engagement's statements file first, then the `context/` documents.
2. Submit only issues that you expect to be accepted under these guidelines.
3. Submit each issue separately, even when two issues are on the same line. Combine repeated occurrences of one root cause into one issue; where occurrences differ, use your judgment.
4. Be specific and sufficiently descriptive when describing an issue's impact.
5. **Do not add** unnecessarily **long code snippets** into the submission. Keep the code snippet limited to the scope of the impact, and be as descriptive as possible in the Vulnerability Detail sections.
6. Do not copy-paste issues from other engagements, reports or past audits. They are extremely unlikely to be accepted in the respective engagement.

## 6. Glossary

**Attack Path**: the steps a malicious actor takes to harm the system or its users, or to gain value.

**Front-run**: an operation B that executes before an operation A, in the same transaction or another, and causes harm to the system or a user that A alone would not cause.

**Root Cause**: the fundamental reason for an unwanted outcome.

**Sandwich**: a front-run followed by an operation C, by the same actor as B, that returns the system to its initial state.

**System**: the code under audit and what it runs as, such as contracts, nodes or services.

**Statement**: a passage in the engagement's documents, customer instructions or code comments that bears on a verdict. The engagement's statements file lists the statements treated as decisive or leans accepted.

**Time-sensitive function**: a function that must execute within a period to prevent a loss, the permanent failure of an intended operation, or the finalization of an incorrect state, when executing it later cannot prevent or reverse that consequence.

**True finding**: see goal 1.

**Verdict dimension**: one of the four separate assessments of an issue: True Finding, scope, known, and severity.

**Vulnerability Path**: the steps by which an issue harms the system or its users during intended use.
