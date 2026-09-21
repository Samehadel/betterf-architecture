# Legacy OpenAPI Code Generation Notes

> This project does not currently use OpenAPI to generate controllers or transport models. This file is retained only to explain the retired pattern and to keep old references from being misleading.

---

## Status

The active architecture is documented in:

- [backend-architecture.md](../../backend-architecture.md)
- [module-structure/module-structure.md](../module-structure/module-structure.md)
- [api-specification.md](../../api-specification.md)

Current rule:

- Controllers are handwritten Spring MVC classes
- Request objects and response Views are handwritten in `api/dto/`
- `ResponseAdvice` applies the `ApiResponse<T>` envelope at runtime

Do not use this file as implementation guidance for new work.
