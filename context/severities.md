# Severities

## Critical (severity-def:critical-v5)

Abbreviation: C · Weight: 0.6%

At any moment, and with no external conditions outside its control, a malicious actor steals funds directly, takes control of the system, or makes it finalize a state of its choosing that its rules forbid, with harm comparable to stealing funds or taking control. Stolen funds must be a significant loss: more than 20% *and* more than $100 of the affected users' principal. Unclaimed yield and rewards do not count.

## High (severity-def:high-v5)

Abbreviation: H · Weight: 0.2%

Under external conditions outside a malicious actor's control that are at most specific, not extreme, an issue causes a direct considerable loss of funds, or a malicious actor takes control of the system or makes it finalize a state its rules forbid, with harm comparable to a considerable loss. A considerable loss is more than 1% *and* more than $10 of users' principal or yield, or of the protocol's fees.

## Medium (severity-def:medium-v7)

Abbreviation: M · Weight: 0.05%

An issue causes a relevant loss of funds that can require specific external conditions or system states outside a malicious actor's control, or that is highly constrained; or core functionality fails during intended use, making the system unusable or causing a relevant loss; or, under such conditions, a malicious actor makes the system finalize a state its rules forbid, with harm comparable to a relevant loss. A relevant loss is more than 0.01% *and* more than $10 of users' principal or yield, or of the protocol's fees.

## Low (severity-def:low-v7)

Abbreviation: L · Weight: 0.02%

An issue causes a minor loss of funds; or a loss that requires conditions or system states outside a malicious actor's control and far outside normal operation, or that admins can recover; or it breaks functionality without a direct loss of funds, causing minor disruption or needing admin intervention. A minor loss is more than 0.001% *and* more than $1 of users' principal or yield, or of the protocol's fees.

## Info (severity-def:info-v2)

Abbreviation: I · Weight: 0%

An issue that does not reach Low.
