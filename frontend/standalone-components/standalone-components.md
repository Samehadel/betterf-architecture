# Angular Standalone Components — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Component boundaries

Each Angular standalone component imports the template dependencies it uses. Keep page-level orchestration distinct from reusable presentational components.

A neutral presentational example:

```typescript
import { Component, ChangeDetectionStrategy, input, output } from '@angular/core';

@Component({
  selector: 'app-item-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: './item-card.component.html',
})
export class ItemCardComponent {
  readonly label = input.required<string>();
  readonly selected = output<void>();
}
```

```html
<!-- item-card.component.html -->
<button type="button" (click)="selected.emit()">{{ label() }}</button>
```

This example follows the selected standalone, external-template, and OnPush conventions; it does not define a BetterF feature.

## Imports and dependency injection

Import directives, pipes, components, and forms support used by the template. Standalone components are imported in `TestBed` rather than placed in `declarations`.

Angular's `inject()` can resolve dependencies inside an injection context. Use `inject()` and external templates as specified in the frontend architecture.

## Composition

Page components coordinate services and state, then pass the data a child needs through inputs. Children emit user intent through outputs. Keep shared controls free of feature-specific service and store dependencies.

Standalone directives can encapsulate reusable interaction behavior. Pure pipes can format presentation values without owning business state.

## Testing

Set signal inputs through `fixture.componentRef.setInput()`. Assert rendered behavior and output events through user interactions. Use semantic controls or stable test identifiers and test disabled, empty, and error states when applicable.
