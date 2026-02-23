# LangCore Hybrid

> Provider plugin for [LangCore](https://github.com/ignatg/langcore) — combine deterministic rule-based extraction with LLM fallback to save 50–80% of LLM costs.

[![PyPI version](https://img.shields.io/pypi/v/langcore-hybrid-llm-regex)](https://pypi.org/project/langcore-hybrid-llm-regex/)
[![Python](https://img.shields.io/pypi/pyversions/langcore-hybrid-llm-regex)](https://pypi.org/project/langcore-hybrid-llm-regex/)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

---

## Overview

**langcore-hybrid** is a provider plugin for [LangCore](https://github.com/ignatg/langcore) that evaluates deterministic rules (regex patterns, Python functions) before falling back to an LLM. When a rule matches with sufficient confidence, the extraction is instant and free. Only prompts that miss all rules are sent to the LLM — dramatically reducing API costs for documents with predictable patterns.

---

## Features

- **Rules-first architecture** — deterministic extraction for patterns that don't need an LLM (dates, amounts, reference numbers, emails, etc.)
- **Regex rules** — extract structured data via named capture groups with optional output templates
- **Callable rules** — plug in any Python function for arbitrary extraction logic
- **Confidence thresholds** — optionally fall back to LLM when rule confidence is below a minimum threshold
- **Batch-aware async** — in async mode, only prompts that miss all rules are batched for LLM inference
- **Observability counters** — built-in thread-safe counters for `rule_hits` vs `llm_fallbacks` with atomic snapshots and reset
- **Text extractor hook** — isolate document text from prompt instructions before rule evaluation
- **Output formatter hook** — normalise rule outputs for downstream consumers
- **Custom rules via ABC** — extend `ExtractionRule` to plug in spaCy NER, custom parsers, or any logic
- **Zero overhead on hits** — rule evaluation is pure Python with no network calls
- **Zero-config plugin** — auto-registered via Python entry points

---

## Installation

```bash
pip install langcore-hybrid-llm-regex
```

For optional spaCy NER support:

```bash
pip install "langcore-hybrid-llm-regex[spacy]"
```

---

## Quick Start

### Integration with LangCore

langcore-hybrid integrates with LangCore through the **decorator provider pattern**. Wrap any LangCore model to add rule-based extraction with LLM fallback:

```python
import langcore as lx
from langcore_hybrid import HybridLanguageModel, RegexRule, RuleConfig

# Create the fallback LLM provider
inner_model = lx.factory.create_model(
    lx.factory.ModelConfig(model_id="litellm/gpt-4o", provider="LiteLLMLanguageModel")
)

# Define rules for common, predictable patterns
rules = RuleConfig(rules=[
    RegexRule(
        r"Date:\s*(?P<date>\d{4}-\d{2}-\d{2})",
        description="ISO date extraction",
    ),
    RegexRule(
        r"Amount:\s*\$(?P<amount>[\d,.]+)",
        description="USD amount extraction",
    ),
    RegexRule(
        r"Ref(?:erence)?[:\s]+(?P<ref>[A-Z]+-\d+)",
        description="Reference number extraction",
    ),
])

# Create hybrid provider
hybrid_model = HybridLanguageModel(
    model_id="hybrid/gpt-4o",
    inner=inner_model,
    rule_config=rules,
)

# Use as a drop-in replacement
result = lx.extract(
    text_or_documents="Contract Ref: ABC-123, Date: 2026-01-15, Amount: $75,000",
    model=hybrid_model,
    prompt_description="Extract contract metadata.",
    examples=[
        lx.data.ExampleData(
            text="Contract Ref: XYZ-456, Date: 2025-06-01, Amount: $10,000",
            extractions=[
                lx.data.Extraction("reference", "XYZ-456"),
                lx.data.Extraction("date", "2025-06-01"),
                lx.data.Extraction("monetary_amount", "$10,000"),
            ],
        )
    ],
)

# Check cost savings
print(f"Rule hits: {hybrid_model.rule_hits}")       # Patterns caught by rules (free)
print(f"LLM fallbacks: {hybrid_model.llm_fallbacks}")  # Prompts sent to LLM
```

---

## Usage

### Regex Rules

Use named capture groups to extract structured data:

```python
from langcore_hybrid import RegexRule

# Basic extraction
rule = RegexRule(r"Date:\s*(?P<date>\d{4}-\d{2}-\d{2})", description="ISO date")

# With custom output template
import json
rule = RegexRule(
    r"Amount:\s*\$(?P<amount>[\d,.]+)",
    output_template=lambda groups: json.dumps({
        "amount": float(groups["amount"].replace(",", "")),
        "currency": "USD",
    }),
)
```

### Callable Rules

Plug in any Python function for complex extraction logic:

```python
import json
from langcore_hybrid import CallableRule, RuleConfig

def extract_email(prompt: str) -> str | None:
    """Extract email addresses deterministically."""
    import re
    emails = re.findall(r'\b[\w.+-]+@[\w-]+\.[\w.]+\b', prompt)
    if emails:
        return json.dumps({"emails": emails})
    return None  # Return None to signal a miss — fall back to LLM

rules = RuleConfig(rules=[
    CallableRule(func=extract_email, description="email extraction"),
])
```

### Confidence Thresholds

Route low-confidence rule matches to the LLM instead:

```python
from langcore_hybrid import RegexRule, RuleConfig

rules = RuleConfig(
    rules=[
        RegexRule(r"(?P<date>\d{4}-\d{2}-\d{2})", confidence=1.0),     # ISO dates: high confidence
        RegexRule(r"(?P<date>\d{1,2}/\d{1,2}/\d{2,4})", confidence=0.6),  # Ambiguous: low confidence
    ],
    fallback_on_low_confidence=True,
    min_confidence=0.8,
)
# "2026-01-15" → rule hit (1.0 ≥ 0.8) → instant result
# "1/15/26"   → rule hit but low confidence (0.6 < 0.8) → LLM fallback
```

### Rule Evaluation Order

Rules are evaluated **in list order**. The first rule that matches and meets the confidence threshold wins:

1. Place **high-confidence, specific** rules first (e.g., ISO dates)
2. Follow with **lower-confidence, broader** rules (e.g., ambiguous date formats)
3. The LLM acts as the final catch-all

### Async Batch Optimisation

In async mode, only prompts that miss all rules are sent to the LLM in a single batch:

```python
results = await hybrid_model.async_infer([
    "Date: 2026-01-15",       # Rule hit — instant, free
    "Complex clause text...",  # Rule miss — sent to LLM
    "Ref: ABC-123",           # Rule hit — instant, free
])
# Only 1 LLM call for the 1 miss, not 3
```

### Text Extractor Hook

Isolate document text from prompt instructions to prevent rules from matching instruction fragments:

```python
def extract_after_marker(prompt: str) -> str:
    marker = "---DOCUMENT---"
    idx = prompt.find(marker)
    return prompt[idx + len(marker):].strip() if idx >= 0 else prompt

rules = RuleConfig(
    rules=[RegexRule(r"Date:\s*(?P<date>\d{4}-\d{2}-\d{2})")],
    text_extractor=extract_after_marker,
)
```

### Output Formatter Hook

Normalise rule outputs into a consistent format for downstream consumers:

```python
import json

rules = RuleConfig(
    rules=[RegexRule(r"Ref:\s*(?P<ref>[A-Z]+-\d+)")],
    output_formatter=lambda raw: json.dumps({"result": json.loads(raw)}),
)
```

---

## Observability

Track cost savings with thread-safe counters:

```python
print(f"Rule hits: {hybrid_model.rule_hits}")
print(f"LLM fallbacks: {hybrid_model.llm_fallbacks}")

# Atomic snapshot (thread-safe)
snapshot = hybrid_model.get_counters()
# {"rule_hits": 42, "llm_fallbacks": 7}

# Reset for the next job
hybrid_model.reset_counters()
```

---

## Custom Rules

Implement the `ExtractionRule` ABC for any extraction logic:

```python
from langcore_hybrid.rules import ExtractionRule, RuleResult

class SpacyNERRule(ExtractionRule):
    def __init__(self, nlp_model: str = "en_core_web_sm") -> None:
        import spacy
        self._nlp = spacy.load(nlp_model)

    def evaluate(self, prompt: str) -> RuleResult:
        doc = self._nlp(prompt)
        entities = [{"text": ent.text, "label": ent.label_} for ent in doc.ents]
        if entities:
            import json
            return RuleResult(hit=True, output=json.dumps(entities), confidence=0.85)
        return RuleResult(hit=False)
```

---

## Composing with Other Plugins

langcore-hybrid composes with other decorator providers — use guardrails to validate rule and LLM outputs, and audit to log everything:

```python
import langcore as lx
from langcore_audit import AuditLanguageModel, LoggingSink
from langcore_guardrails import GuardrailLanguageModel, SchemaValidator, OnFailAction
from langcore_hybrid import HybridLanguageModel, RegexRule, RuleConfig

# Base LLM
llm = lx.factory.create_model(
    lx.factory.ModelConfig(model_id="litellm/gpt-4o", provider="LiteLLMLanguageModel")
)

# Layer 1: Rules + LLM fallback
hybrid = HybridLanguageModel(
    model_id="hybrid/gpt-4o", inner=llm,
    rule_config=RuleConfig(rules=[
        RegexRule(r"Date:\s*(?P<date>\d{4}-\d{2}-\d{2})"),
        RegexRule(r"Amount:\s*\$(?P<amount>[\d,.]+)"),
    ]),
)

# Layer 2: Output validation
guarded = GuardrailLanguageModel(
    model_id="guardrails/gpt-4o", inner=hybrid,
    validators=[SchemaValidator(MySchema, on_fail=OnFailAction.REASK)],
)

# Layer 3: Audit trail
audited = AuditLanguageModel(
    model_id="audit/gpt-4o", inner=guarded,
    sinks=[LoggingSink()],
)

result = lx.extract(text_or_documents="...", model=audited, ...)
```

---

## Development

```bash
pip install -e ".[dev]"
pytest
```

## Requirements

- Python ≥ 3.12
- `langcore`
- Optional: `spacy` ≥ 3.8.11 (for `[spacy]` extra)

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
