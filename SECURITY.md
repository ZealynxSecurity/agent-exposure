# Security policy

Report vulnerabilities privately through GitHub's **Security → Report a vulnerability** on this repository. Please don't open a public issue.

The bugs that matter most here:
- a permission reported as **bounded** (or lower) when it allows more to leave;
- a permission left out of a report without a "not checked" note;
- anything that sends a user's addresses or data anywhere other than the RPC they chose.

If agent-exposure helps find a flaw in a third party's contracts, that vendor is told privately first. The case is added to our fixtures only after the fix is public.
