# SCIM PATCH Evaluator

Paste a SCIM User or Group and a PATCH request and see the resource after every operation under a literal reading of RFC 7644, beside what Microsoft Entra's documented request shapes intend.

**Live demo:** https://0xelitesystem.github.io/scim-patch-evaluator/

## Use

1. Paste a SCIM resource into the left box: a User, a Group, or a User carrying the enterprise extension (`urn:ietf:params:scim:schemas:extension:enterprise:2.0:User`).
2. Paste the PATCH request body into the right box: `schemas` with the PatchOp URN and an `Operations` array.
3. Read the result. Each operation is applied in order, the result of one feeding the next, and every step shows the op as received, the normalised op, the resolved target, the RFC 7644 section and bullet applied, and the before and after difference.
4. The verdict is either success (RFC 7644 3.5.2: 200 with the resource, or 204) or HTTP 400 with its `scimType`. A failed request is atomic, so the page shows the ORIGINAL resource, not a half-applied one.
5. Where the request matches a shape Microsoft documents, a second column, **Sender's documented intent (Microsoft Learn)**, sits beside **Literal RFC 7644**. It never appears for any other shape.

Or pick one of the eight presets. Each uses original identifiers and values in the same structure as the cited sample:

| Preset | What it shows |
| --- | --- |
| 1 | Entra member removal without the feature flag: `"op":"Remove","path":"members"` plus a `value` array. Literal result: members unassigned. Intent result: two members left. |
| 2 | The same removal with the flag on: `members[value eq "..."]`. |
| 3 | Legacy disable user: `"op":"Replace","path":"active","value":"False"`. Literal: 400 invalidValue. Intent: active becomes false. |
| 4 | Flag-on multi-attribute replace on a user with a work email: a filtered sub-attribute replace plus a no-path replace with `name.givenName` and an enterprise `employeeNumber` URN key. |
| 5 | The same on a user with a home email only: 400 noTarget and a rollback. |
| 6 | `remove emails[type eq "work" and value ew "example.com"]`. |
| 7 | `replace addresses[type eq "work"]` with `primary` true, so the other primary is reset. |
| 8 | Atomic rollback: operation 1 succeeds, operation 2 is noTarget, and the whole request rolls back. |

### Use cases

- **You are building a SCIM server that Entra will call.** Paste a group from your database and the request body from your provisioning logs, and see what a literal reading of the RFC does with it next to what Microsoft says the request means. Then decide, on purpose, which one your handler implements.
- **You are reviewing a SCIM library's PATCH handler.** Run the same resource and request through the library and through this page. Every step names the bullet it applied, so a difference points at one sentence of the RFC instead of a vague "it behaves differently".
- **A group has fewer members than expected after a provisioning cycle.** Paste the logged request and the group as it was before. The headline row tells you whether the request was the unfiltered `members` removal that the literal text reads as "remove every member".
- **You are writing test fixtures.** Load a preset, change the values, and copy the final resource out of the "final resource" panel as the expected output for your own tests, with the rule that produced it written beside it.
- **You are debugging a filter.** `emails[type eq "work" and value ew "example.com" or primary eq true]` parses with attribute expressions binding tightest, then `not`, `and`, `or`. The resolved target line re-prints the filter from the parse tree with the parentheses that precedence implies: `emails[(type eq "work" and value ew "example.com") or primary eq true]`.

### What the literal column does that is easy to get wrong

- `remove` with path `members` and no filter removes the attribute and all values (3.5.2.2, bullet 3). No remove rule reads a `value` member, and the page says so on that step.
- An `add` whose path carries a filter has no rule in 3.5.2.1. It is shown as UNDEFINED and not applied, and the verdict says it depends on an undefined step. Erratum 8097 (Held for Document Update) raises exactly this case.
- `replace` with a filter that matches nothing is 400 noTarget (3.5.2.3, bullet 8). When the attribute is entirely absent, bullet 4 (treat as add) and bullet 8 conflict; both are shown.
- `members.value` has `caseExact` false in RFC 7643 8.7.1, so `members[value eq "ABC"]` matches `"abc"`. `manager.value` is `caseExact` true per erratum 8472 (Verified).
- Setting `primary` true on one value sets it false on the others (3.5.2, SHALL). Writing an attribute by its fully qualified extension name adds that URN to `schemas` (3.5.2). Nothing removes a URN from `schemas` implicitly.

### Where the text gives two readings, both are shown

- Dotted or URN-qualified keys inside a no-path value (`"name.givenName": "..."`). Reading A: value keys are attribute names, and the RFC 7643 2.1 ATTRNAME rule has no `.`. Reading B: RFC 7644 3.10 says "All operations share a common scheme for referencing simple and complex attributes", `{urn}:{Attribute name}.{Sub-Attribute name}`. The page carries reading B forward and never labels the shape invalid. This shape appears in the sample Microsoft labels SCIM Compliant.
- A boolean sent as the string `"False"`. Literal: 400 invalidValue, because RFC 7643 2.3 Table 1 maps Boolean to the JSON literal. RFC 7643 2.3.2 also says "A boolean has no case sensitivity or uniqueness.", and that sentence is sometimes read as permitting it.
- op case. RFC 7644 3.5.2 says op "MAY be one of "add", "remove", or "replace"" and states no case rule. By default the page evaluates op case-insensitively and flags anything else; a **Strict literal op** checkbox rejects instead, without naming a `scimType`, because RFC 7644 names none for this case.

### Named refusals

A value path nested inside a value filter (400 invalidFilter; erratum 4690, Held for Document Update), a bare attribute expression used as a PATCH path (400 invalidPath; erratum 7122, Held for Document Update, proposes allowing it), and an unknown operator such as `regex` (400 invalidFilter, 3.4.2.2). Filter precedence follows erratum 4670 (Held for Document Update): the printed text lists grouping, logical, attribute, and every example in the RFC needs attribute expressions to bind tightest.

### The self test

The page runs a self test on load and on demand, against the same code that evaluated your request: one fixture per bullet of 3.5.2.1, 3.5.2.2 and 3.5.2.3, every preset, the URN, quoting, case and `caseExact` fixtures, the named refusals, a precedence fuzz that generates seeded random boolean trees over and, or and not, prints each with minimal parentheses, parses it back and compares the result with an independent oracle on random records, and a positive control: an assertion written to fail. The page prints its own live counts; if the control ever reports PASS, the suite is not running.

## Why this exists

The request Microsoft documents for removing a group member and the text of RFC 7644 point at different results, and each side follows its own published document. Microsoft's known-issues page lists "Update PATCH behavior to ensure compliance (such as active as boolean and proper group membership removals)" as Fixed: No, Fix date: TBD, Backwards compatibility: use feature flag, and calls the behaviour without the flag "non SCIM compliant". Its SCIM tutorial still prints the unfiltered shape as the "Update Group [Remove Members]" example. Read literally, RFC 7644 3.5.2.2 removes the attribute and all its values, and no remove rule reads a `value` member.

This page puts the literal result and the sender's documented intent side by side, with the clause behind every change, so the choice is visible and deliberate.

Single file, no build step, no network call, MIT licensed. Nothing to install, and nothing watching you paste a production group.

## Privacy

Everything runs in your browser. There is no network call of any kind: no request for code, fonts, analytics or telemetry, and nothing you paste is transmitted anywhere. A Content-Security-Policy in the page blocks every connection. The only thing stored is your light or dark theme choice, in your own browser's localStorage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/scim-patch-evaluator.git
cd scim-patch-evaluator
```

Open `index.html` in a browser. That is the whole thing. If you would rather serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and inline JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Sources

Re-read on 2026-10-01.

- RFC 7644, SCIM Protocol (September 2015): https://www.rfc-editor.org/rfc/rfc7644.txt
- RFC 7643, SCIM Core Schema (September 2015): https://www.rfc-editor.org/rfc/rfc7643.txt
- Errata: https://www.rfc-editor.org/errata_search.php?rfc=7644 and https://www.rfc-editor.org/errata_search.php?rfc=7643. Statuses used: 4670, 4690, 7122 and 8097 Held for Document Update and 7319 Reported (RFC 7644); 5368, 8471 and 8472 Verified and 6011 Reported (RFC 7643).
- RFC 9865 (October 2025) and RFC 9967 (May 2026) both update RFC 7643 and 7644. RFC 9865 adds cursor-based pagination; RFC 9967 adds SCIM security events and an optional asynchronous request mode. Neither changes how PATCH operations are applied under 3.5.2.
- Microsoft Learn, SCIM 2.0 compliance known issues (visible "Last updated on" 2025-08-25): https://learn.microsoft.com/en-us/entra/identity/app-provisioning/application-provisioning-config-problem-scim-compatibility. Flags `aadOptscim062020` (SCIM Compliant) and `AzureAdScimPatch2017` (roll back). The page notes: "Note this feature flag currently doesn't work with on-demand provisioning."
- Microsoft Learn, SCIM endpoint tutorial (visible "Last updated on" 2026-09-15): https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups. "Microsoft Entra ID emits the values of op as Add, Replace, and Remove."

## Independence

This is an independent tool. It is not affiliated with or endorsed by Microsoft or the IETF. Microsoft Entra is named only to describe request shapes that Microsoft publishes, and nothing here implies endorsement by either party.

## Third-party notices

Behaviour was reimplemented from RFC 7643 and RFC 7644; no code is reproduced here. The schemas are a derived table written for this page (name, type, multi-valued, required, caseExact, mutability, sub-attributes), not the RFC 7643 8.7.1 figure. Short sentences of the RFCs are quoted on the page as the rule behind each step. Microsoft Learn publishes request samples, not rule sentences, so for shapes (i) to (iii) the intent line is marked "Intent inferred from the sample" and is the page's reading of that sample, not Microsoft's wording. Preset request structures follow the Microsoft Learn samples cited above, with original identifiers and values.

RFC 7643 and RFC 7644 carry this notice: "Copyright (c) 2015 IETF Trust and the persons identified as the document authors. All rights reserved."

## Related

- https://github.com/0xelitesystem/json-path-tester
- https://github.com/0xelitesystem/oauth-and-sso-basics-reference
- https://github.com/0xelitesystem/http-message-signature-base-builder

## License

MIT. Copyright (c) 2026 0xelitesystem.
