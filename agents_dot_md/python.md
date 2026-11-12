## Code style

### Type hints

- **Fully typed Python — critical.** Type-hint everything Python's syntax allows: every parameter
  and return type (including `-> None`), and an annotation on **every assignment line** — first
  assignment and every re-assignment of that variable later. Skip a hint only where the syntax is
  impossible (`for`-loop targets, `with ... as` targets, lambda arguments, augmented `+=`); "it's
  obvious", "it's verbose", or "the checker infers it" are not exemptions. Untyped code is a defect.
- **Signatures & syntax.** Abstract parameter types (`Mapping`, `Sequence`, `Iterable`), concrete
  return types. Modern syntax only (`list[int]`, `X | None`), never `List`/`Dict`/`Optional`.
  `NamedTuple` only for tiny immutable returns; dataclass is the default.
- **Most specific type.** Annotate every value with the *narrowest* type it can have: when two
  branches produce different models, write the union of the exact types (`Invoice | Receipt`),
  never a shared base class, `object` or `Any` widened just to let one annotation cover both.
  A base class is correct only when its contract is all we use — and then it is the interface
  we mean, not a convenience. Same for the reverse: `dict[str, JsonValue]` is not "the same as"
  the row type it actually holds.

### Structured data

- **Model all structured data — critical.** Any variable, parameter, return value or attribute
  holding data with more than one named field uses exactly one of: Pydantic v2 `BaseModel`,
  `@dataclass`, or `TypedDict`. Never `dict[str, Any]`, bare `dict`, plain tuples or `Any` for
  structured data; about to write `data["key"]` on an untyped dict → define a type first.
  One concept = one type (never both a TypedDict and a dataclass for the same shape).
- **Decision procedure** (first match wins):
  1. Crosses a trust boundary or needs validation/parsing/coercion (HTTP bodies, config files,
     env vars, user input, LLM output, files, third-party responses, DB rows we don't control)
     → Pydantic v2 `BaseModel`.
  2. Must remain a plain dict at runtime (JSON passed through unchanged, library dict contracts,
     `**kwargs` typed via `Unpack[...]`) → `TypedDict`.
  3. Otherwise (trusted internal, already validated) → `@dataclass`.
  If unsure: "could it be malformed on arrival?" → Pydantic. "must it stay a dict?" → TypedDict.
- **Pydantic v2 only.** `model_validate`, `model_dump`, `field_validator`, `model_validator`,
  `ConfigDict`, `Field`, `Annotated`; never v1 APIs (`parse_obj`, `.dict()`, `.json()`,
  `@validator`, `class Config`). Default `model_config = ConfigDict(extra="forbid", frozen=True)`
  unless extras/mutation are explicitly needed (comment why). `TypeAdapter` for non-model types
  (incl. validating a TypedDict at the boundary). Convert to internal dataclasses after
  validation when data flows deep and needs no further validation.
- **Dataclasses.** Default `@dataclass(slots=True, frozen=True, kw_only=True)`; drop `frozen`
  only if mutation is genuinely required (comment why). `field(default_factory=...)` for
  mutable defaults — never a mutable default directly. No runtime validation here; needing it
  makes it a Pydantic case.
- **TypedDicts.** `NotRequired`/`Required` for optional keys, `ReadOnly` where applicable.
  No runtime validation — a promise to the checker only; validate untrusted data with
  `TypeAdapter(MyTypedDict).validate_python(...)` at the boundary.
- **Typed collections are not structured data.** `list[int]`, `set[str]` and scalar-keyed
  lookup maps (`dict[str, str]`, e.g. status-code → label) carry no named fields — plain
  typed dicts are fine. But a map's *values* must be modelled when they have named fields:
  `dict[str, InvoiceLine]`, never `dict[str, dict[str, Any]]`.
- **Opaque JSON** that we never interpret (forwarded verbatim, or validated later by a Pydantic
  model at the call site) is typed inline as `dict[str, JsonValue]` (Pydantic's recursive
  `JsonValue`) — never `Any`, never a project-defined alias. If you read
  a field out of it, it is not opaque: model it.
- **Pre-task checklist.** No untyped dict/`Any` carrying structured data; every system boundary
  validates with Pydantic v2; internal structured objects are frozen dataclasses; must-stay-dict
  shapes are TypedDicts; no Pydantic v1 APIs.