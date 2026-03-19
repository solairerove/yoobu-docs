### Race conditions

For future multi-instance deployment, consider adding explicit optimistic concurrency to the public/admin API contract.
Current backend protection (@Version + server-side 409 on stale write) prevents lost updates, but clients cannot resolve conflicts intelligently because version is not exposed.

Proposed enhancement:
- Include version (or ETag) in booking responses.
- Require version in mutation requests (or If-Match header).
- Reject mismatches with 409 Conflict (or 412 Precondition Failed for ETag flow).
- Update UI/client flows to handle conflict by refetching latest booking state and prompting user to retry.

This will make race handling explicit and user-friendly across multiple service instances and concurrent actors.