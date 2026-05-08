# Sources as specs — abcd never holds data

Sources are specs (natural-language descriptions of expected data), not data abcd fetches or validates. The executor fulfills them at runtime using whatever tools it has. This means sources are always live — structurally guaranteed, not tracked. This is a deliberate boundary: abcd resolves dependencies and tracks freshness, but never touches external systems. It keeps the core lean (no API clients, no provider protocols, no plugin system) at the cost of abcd being unable to validate source availability before execution.
