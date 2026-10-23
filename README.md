# Agentype

A meta-architecture for agent-first business apps.

## What this is

One document: [META-ARCHITECTURE.md](META-ARCHITECTURE.md).

It sets architecture rules for business apps where users work through a runtime agent. Each rule states what is decided, why, and how it is enforced: by an automated check, a verification agent, or an eval.

Topics: structure, data and enforcement, agent interface, security, external systems, user experience, observability and cost, and build and change.

It is not the architecture of any one app, and not a framework or a library. It is the base that each app's architecture is written from.

Review the rules against your app's needs before you use them, especially the security rules. This document may change at any time.

## How to use it

1. Write an app design from this document. Carry forward every rule, adapted to the app. Change any rule at design time, and record why. Section 9 lists what the app design must define.
2. Remove this document from the app repo. The app design is then the only source of rules.
3. Build and verify the core first, before any domain code.

## Related

[PhaseCraft](https://github.com/anoop-pb/phasecraft): a framework for building maintainable software with Claude Code in verified phases. The two work well together but do not depend on each other.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You may copy, adapt, and use this document, including in commercial and client work, with credit.

Suggested credit: *Based on Agentype by Anoop P B, licensed under CC BY 4.0.*

Copyright © 2026 Anoop P B.
