---
description: "Hexagonal-Clean Mix Architecture"
trigger: always_on
---

# Architecture Rules

This project follows a Clean Architecture approach. All code under ⁠ src/ ⁠ must respect these rules.

## Layers

The codebase is organized in four layers:

•⁠  ⁠*⁠ domain/ ⁠* — Business logic, use cases, types
•⁠  ⁠*⁠ ports/ ⁠* — Interfaces needed by domain to connect to external systems
•⁠  ⁠*⁠ adapters/ ⁠* — Connects domain to infra (data translation, exception mapping). Implement interfaces defined in ports
•⁠  ⁠*⁠ infrastructure/ ⁠* — External systems: databases, HTTP handlers, agents, cloud services

## Dependency Rules

•⁠  ⁠*Domain must NOT import from adapters or infra.* Domain code is pure Python only. It can import ports.
•⁠  ⁠*Ports must NOT import from adapters or infra.* Domain code is pure Python only. It can import ports.
•⁠  ⁠*Infra must NOT import from domain or adapters.* It defines its own types and exceptions.
•⁠  ⁠*Adapters import from domain, ports and infra.* They are the only layer allowed to bridge domain and infra.

## Coding Conventions

The general principles of DI/SRP/composition live in the ⁠ software-design-principles ⁠ skill; this rule only covers how to express them in this project's Python layout.

•⁠  ⁠Use ⁠ Protocol ⁠ (from ⁠ typing ⁠) for all interfaces — never use ⁠ ABC ⁠.
•⁠  ⁠Use ⁠ @dataclass(frozen=True) ⁠ for all data structures (types, inputs, outputs, entities).
•⁠  ⁠Use private instance variables with underscore prefix: ⁠ self._repository ⁠, not ⁠ self.repository ⁠.
•⁠  ⁠All port, use case, adapter, and repository methods must be ⁠ async ⁠.
•⁠  ⁠Use f-strings format for strings with parameters.
•⁠  ⁠Docstrings are only allowed in interfaces.

## Use Cases

•⁠  ⁠Methods can receive an optional ⁠ ctx:RequestContext ⁠ and single input dataclass and return a single output dataclass. The naming convention is the name of the function in camel case plus Input/Output. Example: GetIndexMappingInput, GetIndexMappingOutput for a function called get_index_mapping.
•⁠  ⁠Types are defined in a separate ⁠ _types.py ⁠ file.
•⁠  ⁠Use cases depend on *ports*, never on infra or adapters directly.
•⁠  ⁠Always raise AIError exceptions (usually already converted by adapters).

## Ports

•⁠  ⁠Ports define their *own input and output models*, separate from use case types. The naming convention is the name of the function in camel case plus PortInput/PortOutput. Example: GetIndexMappingPortInput, GetIndexMappingPortOutput for a function called get_index_mapping.
•⁠  ⁠Expressed as ⁠ Protocol ⁠ classes.
•⁠  ⁠Input and output types as well as the class is defined in a single file.
•⁠  ⁠There must be one file per port.

## Adapters

•⁠  ⁠Implement one or more ports.
•⁠  ⁠Depend on infra interfaces (also ⁠ Protocol ⁠-based unless they are generated like API clients).
•⁠  ⁠Use dedicated *mapper files* (⁠ _database_mappers.py ⁠, ⁠ _handlers_mappers.py ⁠) for all data translation.
•⁠  ⁠Mapper functions are pure, stateless, and follow ⁠ map_<source>_to_<target> ⁠ naming.
•⁠  ⁠Catch infra exceptions and re-raise as domain exceptions or sdk AIError.

## Infrastructure

•⁠  ⁠Defines its own exceptions (⁠ GenericInfrastructureException ⁠), and types.
•⁠  ⁠Interface (⁠ Protocol ⁠) + default implementation in the same file. Generated infra (HTTP clients) doesn't follow this pattern.
•⁠  ⁠Infra code may depend on external libraries (boto3, FastAPI, Pydantic, etc.).

## Type Hints

•⁠  ⁠*All functions, methods, and variables must have explicit type hints.* This includes parameters, return types, and local variables where the type is not obvious from the assignment.
•⁠  ⁠Use types from ⁠ typing ⁠ (e.g., ⁠ Optional ⁠, ⁠ Union ⁠, ⁠ List ⁠, ⁠ Dict ⁠) and domain-specific types wherever applicable.
•⁠  ⁠After writing or modifying code, always verify type correctness by running ⁠ mypy ⁠ with strict mode (⁠ mypy --strict ⁠). Fix all reported errors before considering the task complete.

## Code Style

•⁠  ⁠*NEVER add comments, add docstrings only to interface functions.* No inline comments, no block comments, no function/class/module docstrings. The code must be self-explanatory through clear naming and structure.

## Exceptions

•⁠  ⁠Domain defines its needed extensions in ⁠ domain/exceptions.py ⁠. They must extend from AIError from the sdk and will only define exceptions when they are needed in addition to the sdk AIError ones.
•⁠  ⁠Infrastructure defines: ⁠ GenericInfrastructureException ⁠ (in ⁠ infrastructure/exceptions.py ⁠). These ones never extend from sdk AIError, unless they are in generated code.
•⁠  ⁠Adapters catch infrastructure exceptions and raise domain exceptions.
•⁠  ⁠Domain code only catches and raises domain exceptions (or sdk AIError).
