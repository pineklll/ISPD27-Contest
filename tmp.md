"Swap logically-equivalent pins" = **re-wiring which net connects to which input pin of the same cell instance**, where the cell's Boolean function doesn't notice the difference. The cell isn't resized, moved, added, or removed — only the input pin assignment changes.

## Concrete example from your ASAP7 library

`AOI21x1_ASAP7_75t_R` in [platform/asap7/lib/NLDM/](platform/asap7/lib/NLDM/) has:

```
Y = (!A1 * !B) + (!A2 * !B)      # i.e. Y = !((A1 & A2) | B)
```

| Swap | Legal? | Why |
|---|---|---|
| `A1` ↔ `A2` | ✅ yes | `A1&A2` is commutative — the function is identical |
| `A1` ↔ `B` | ❌ no | `B` sits in a different role in the expression; logic changes |
| any input ↔ `Y` | ❌ no | output pin; also inverts/changes connectivity |

So if a net arrives late at `A2`, and `A1` has more slack, you can exchange them. Same gate, same logic, different timing.

## Why it helps if the logic is identical

Because pins on a real cell are **not physically identical**, even when they're logically equivalent. The same AOI21 in your library:

```
pin (A1) capacitance : 1.11252 fF
pin (A2) capacitance : 0.963289 fF
pin (B)  capacitance : 1.13437 fF
```

Different input capacitance, and separate timing arcs per `related_pin` with different internal delays. Putting the critical net on the lower-capacitance / faster pin buys slack **for free**: no extra area, no extra cells, no power increase from added logic, and no placement displacement (the cell doesn't move).

## How the contest enforces it

[legality.py:358-363](evaluator/internal/legality.py#L358-L363) doesn't take your word for it. It:

1. Rejects any change to the set of pins or the output connectivity.
2. Requires the **multiset of input nets to be unchanged** — otherwise it's a logic change, not a swap.
3. Brute-forces the Liberty `function` truth table over every input combination under the permutation ([`symmetric()`](evaluator/internal/legality.py#L222-L232)) and raises `Non-equivalent input-pin swap` if any case differs.

So the rule is exactly "permutation of inputs that is a symmetry of the cell's Boolean function."

## Practical notes

- OpenROAD's `repair_timing` **already does this by default** during setup fixing — `-skip_pin_swap` disables it. Check `/OpenROAD/src/rsz/README.md:262` in the container.
- Your tool must update **both** the `.def` and `.v` consistently. A swap is a netlist connectivity change; if the two disagree, the canonical-export check in [audit.py](evaluator/internal/audit.py) fails.
- It's usually the cheapest win of the three allowed transformations, but the effect is small (a few ps per gate, and only where symmetric pins have asymmetric arcs/caps).

For a beginner, treat it as: *free timing recovery on critical paths, no area/power cost, but only legal inside one cell and only for symmetric inputs.*