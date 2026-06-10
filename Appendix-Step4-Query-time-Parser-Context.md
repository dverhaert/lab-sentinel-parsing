# Appendix — Step 4 Query-time Parser Context

This appendix contains the deep-dive context moved out of Step 4.6.5 so the main lab flow stays concise.

## Why the parser is shaped this way

| Block | Why |
|-------|-----|
| Parameters defined in Save as function | These define the callable signature. Defaults like `datetime(null)`, `dynamic([])`, and `"*"` mean "no filter". |
| `where not(disabled)` | Optional kill switch used by ASIM patterns. |
| Early time filter | `TimeGenerated` is indexed; filtering here first is the biggest performance win. |
| Early extraction (`_upn`, `_srcip`, `_rawType`, `_outcome`) | Enables cheap filter pushdown before full normalization. |
| Map to ASIM values first | Rules filter on ASIM vocabulary (`Logon`, `Failure`), not vendor-native values. |
| Pushdown filters (`targetusername_has_any`, `srcipaddr_has_any_prefix`, `eventtype_in`, `eventresult`) | Limits rows before expensive shaping. |
| Full normalization late | Avoids parsing/shaping rows that will be filtered out anyway. |
| Aliases (`User`, `IpAddr`, `Dvc`) | Common ASIM aliases used by some content. |

## Buffet vs menu (detailed)

- `ASimAuthenticationContosoAuth` is the buffet: parse first, then filter.
- `vimAuthenticationContosoAuth(...)` is the menu: filter first, parse only survivors.
- Same logical result, usually different cost at scale.

### Two terms

- `parse_json`: opening nested dynamic payloads such as `RawEvent.user.upn`.
- Optimizer: Kusto planner that can push cheap filters down when the function body allows it.

Key rule: filter on indexed columns like `TimeGenerated` as early as possible, then do heavier parsing and projection.

### Practical walkthrough

If a rule looks for failed sign-ins in the last hour from one source prefix:

```kusto
vimAuthenticationContosoAuth(
    starttime = ago(1h),
    eventresult = "Failure",
    srcipaddr_has_any_prefix = dynamic(["185.220."]))
| summarize FailCount = count() by TargetUserName, SrcIpAddr
| where FailCount >= 5
```

The engine can narrow by time and simple filters first, then fully normalize only remaining rows.

### Parameter quick reference

| Parameter | What it does |
|-----------|---------------|
| `starttime`, `endtime` | Time window bounds |
| `targetusername_has_any` | Match one or more users |
| `srcipaddr_has_any_prefix` | Match one or more source IP prefixes |
| `eventtype_in` | Match ASIM event types |
| `eventresult` | Match result (`Success`/`Failure`/`*`) |

Defaults used in ASIM filtering conventions:

- `datetime(null)` means no time bound
- `dynamic([])` means no list filter
- `"*"` means no result filter

### Why this matters for unifiers

`_Im_Authentication` forwards named parameters to registered parsers. A filtering parser should keep expected parameter names and defaults so unifier calls behave consistently and efficiently.

## Follow-up documentation

- [Develop a custom ASIM parser](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers)
- [Authentication ASIM schema](https://learn.microsoft.com/en-us/azure/sentinel/normalization-schema-authentication)
- [Manage ASIM parsers](https://learn.microsoft.com/en-us/azure/sentinel/normalization-manage-parsers)
