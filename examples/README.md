# OpenExchange worked examples

Synthetic documents an exchange adapter emits. Nothing here is real venue data;
`example-exchange` does not exist and `example.invalid` is a reserved domain.

| File | Document | Shows |
| --- | --- | --- |
| `fee_profile.json` | `exchange:fee_profile` | 5% commission on net market winnings |
| `venue.json` | `exchange:venue` | A betting exchange with a distributor brand and a `feeProfile` reference |
| `market_book.json` | `exchange:market_book` | An open two-runner market with native `selectionId`s and matched volume |
| `order_book_back.json` | `exchange:order_book` | Back ladder for runner `58805`, best price first |
| `order_book_lay.json` | `exchange:order_book` | Lay ladder with one venue-injected `virtualPrice` level |

Walk the documents in this order:

1. `fee_profile.json` → `venue.json` — the venue points at its fee schedule by id.
2. `market_book.json` — exchange-native state (`open`, `inplay`, `betDelay`,
   `totalMatched`) plus the runner list. `status` distinguishes open, suspended,
   closed, settled and void — information an OpenBook market alone cannot carry.
3. `order_book_*.json` — one order book per runner per side. `back` is the price
   at which the venue lets you buy the outcome; `lay` the price to sell it.
   Prices and sizes are decimal strings, never floats.

## Validate an example

Schemas are draft 2020-12 and resolve `common.schema.json` relatively, so
validate from the repository root:

```bash
pip install 'jsonschema>=4.18' referencing
python3 - <<'PY'
import json, pathlib
from jsonschema import Draft202012Validator
from referencing import Registry, Resource

root = pathlib.Path(__file__).parent if '__file__' in dir() else pathlib.Path('.')

def load(path):
    return json.loads((root / path).read_text())

registry = Registry().with_resources(
    [(name, Resource.from_contents(load(f'schema/{name}')))
     for name in ('common.schema.json', 'runner.schema.json')]
)

pairs = [
    ('examples/venue.json', 'schema/venue.schema.json'),
    ('examples/fee_profile.json', 'schema/fee_profile.schema.json'),
    ('examples/market_book.json', 'schema/market_book.schema.json'),
    ('examples/order_book_back.json', 'schema/order_book.schema.json'),
    ('examples/order_book_lay.json', 'schema/order_book.schema.json'),
]
for doc_path, schema_path in pairs:
    schema = load(schema_path)
    resolver = registry.resolver(base_uri=schema['$id'])
    Draft202012Validator(schema, registry=registry).validate(load(doc_path))
    print(f'{doc_path}: OK')
PY
```

Expected: five `OK` lines. An adapter's own output should pass the same check
before it ships.

## What an adapter must add

- Its real venue id, native `selectionId`s and jurisdiction/currency coverage.
- `asOf` (venue timestamp) distinct from `observedAt` (when you saw it).
- `derived: true` on any level you synthesized rather than the venue reported.
- Its own fee model — do not assume `net_winnings`; prediction exchanges often
  charge `per_contract`.
