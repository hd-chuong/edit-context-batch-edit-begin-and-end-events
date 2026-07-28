# Explainer for EditContext Batch Edit Events (`batcheditbegin` / `batcheditend`)

This proposal is an early design sketch to describe the generic problem of web applications processing interim model state during Input Method Editor (IME) batch edits, and to solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

---

## Proponents

- Blink Editing & Web Input Team

## Participate
- Issue / Discussion: [https://github.com/w3c/edit-context/issues/141](https://github.com/w3c/edit-context/issues/141)
- ChromeStatus: [https://chromestatus.com/feature/5130535027474432](https://chromestatus.com/feature/5130535027474432)

## Table of Contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1: IME Batch Edit Operations](#use-case-1-ime-batch-edit-operations)
  - [Use case 2: Preventing Interim Model State Corruption](#use-case-2-preventing-interim-model-state-corruption)
- [Potential Solution](#potential-solution)
  - [Sample Usage Code](#sample-usage-code)
  - [Blink's Architecture](#blinks-architecture)
  - [How this solution solves the use cases](#how-this-solution-solves-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Tricky design choice #1: Dedicated Events vs. Reusing `beforeinput` / `input`](#tricky-design-choice-1-dedicated-events-vs-reusing-beforeinput--input)
- [Considered alternatives](#considered-alternatives)
  - [Alternative 1: Time-Based Debouncing in Web Applications](#alternative-1-time-based-debouncing-in-web-applications)
  - [Alternative 2: Synchronous Full-Document Model Re-Evaluation](#alternative-2-synchronous-full-document-model-re-evaluation)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

---

## Introduction

Input Method Editors (IMEs) frequently perform multi-edit operations wrapped inside native OS batch signals. In Android, multi-edit operations are wrapped inside `InputConnection#beginBatchEdit` and `InputConnection#endBatchEdit`. During a batch transaction, the IME assumes that the surrounding document state and selection coordinate frame remain stable until the entire sequence of atomic text mutations finishes.

Currently, the web-exposed `EditContext` API lacks standard events to notify web applications when a batch edit transaction starts and ends. Consequently, web text editors receive incoming text mutations as separate, unbatched events and immediately mutate their internal document model state.

Exposing interim model state during an active batch transaction causes state drift between the IME's expected document snapshot and the web editor's intermediate state. This misalignment leads to invalid range calculations, selection errors, and document text corruption.

This explainer proposes adding two new events to `EditContext`: `batcheditbegin` and `batcheditend`. These events notify web applications when an IME batch edit transaction starts and ends, enabling web editors to defer interim model state updates and apply mutations atomically when the transaction completes.

---

## Goals

* **End-User Need**: Ensure reliable, corruption-free text editing in web applications when using IMEs that execute multi-edit batch transactions.
* **Web Developer Need**: Provide explicit, deterministic event boundaries (`batcheditbegin` and `batcheditend`) on `EditContext` so web applications do not process or expose interim model state during batch edit transactions.

---

## Non-goals

* **Do not replace `textupdate` or `beforeinput` events**: Individual text mutations still fire normal payload events during a batch. The new events only mark when the batch starts and ends.
* **Do not create a generic DOM Undo or Transaction API**: This proposal is strictly for IME text input boundaries and does not create a general Web or DOM transaction framework.
* **Do not alter native OS or keyboard IME behavior**: This proposal does not change how native operating systems or keyboards generate edits, but only passes existing OS signals (`beginBatchEdit`/`endBatchEdit`) down to the browser.
* **Do not expose private keyboard or IME internal data**: The events carry no keyboard layout, telemetry, or user typing metrics, ensuring zero security or privacy risks.
* **Do not dictate how web applications store or render text**: Web editors retain full control over how they buffer, commit, or render internal model updates during a batch.

---

## User research

TODO
---

## Use cases

### Use case 1: IME Batch Edit Operations
An end-user using an IME executes a multi-edit operation that is enclosed within a native OS batch transaction (`beginBatchEdit` ... `replaceText` ... `replaceText` ... `endBatchEdit`).

### Use case 2: Preventing Interim Model State Corruption
A web text editor maintains an internal document model and selection coordinate system. Without batch edit lifecycle events, receiving the first text mutation in a batch prompts the editor to update its internal model state mid-transaction. Subsequent text mutations within the same batch operate against this mutated interim model state, causing index offset drift and text corruption.

---

## Potential Solution

We propose extending `EditContext` with two events: `batcheditbegin` and `batcheditend`.

```webidl
// Extension to the EditContext Web API
partial interface EditContext {
  attribute EventHandler onbatcheditbegin;
  attribute EventHandler onbatcheditend;
};
```

### Sample Usage Code

```js
const editContext = new EditContext();
let isBatchEditing = false;

// Registering batch edit lifecycle listeners on EditContext
editContext.addEventListener('batcheditbegin', (event) => {
  // Suspend interim model updates and selection recalculations
  isBatchEditing = true;
});

editContext.addEventListener('textupdate', (event) => {
  // Queue or apply text updates against a stable initial transaction snapshot
  handleTextUpdatePayload(event);
});

editContext.addEventListener('batcheditend', (event) => {
  // IME batch transaction complete; resume normal model state updates
  isBatchEditing = false;
  finalizeModelTransaction();
});
```

### Blink's Architecture

```
[Android OS Input Method]
    │  InputConnection#beginBatchEdit() / InputConnection#endBatchEdit()
    ▼
[Android Clank (Java)]
  ThreadedInputConnection.java ──► ImeAdapterImpl.java
    │  JNI Call
    ▼
[Chromium Browser Process (C++)]
  ImeAdapterImpl.cpp
    │  Mojo IPC (mojom::FrameWidgetInputHandler)
    ▼
[Blink Renderer Engine (C++)]
  FrameWidgetInputHandlerImpl ──► WebInputMethodController ──► InputMethodController.cpp
    │  Calls EditContext::BatchEditBegin() / BatchEditEnd()
    ▼
[Blink EditContext (C++)]
  EditContext.cc
    ├── 1. Dispatches 'batcheditbegin' (event_type_names::kBatcheditbegin)
    ├── 2. Receives intermediate 'textupdate' (event_type_names::kTextupdate) payloads
    └── 3. Dispatches 'batcheditend' (event_type_names::kBatcheditend)
```

### How this solution solves the use cases

When the native OS input method calls `beginBatchEdit()`, the browser engine dispatches `batcheditbegin` to `EditContext`. The web editor receives the signal and freezes interim model state updates. As individual `textupdate` events arrive during the batch, the editor processes them against a stable initial transaction snapshot. When `endBatchEdit()` is called, the browser dispatches `batcheditend`, prompting the web editor to finalize and commit the model state in a single atomic update.

---

## Detailed design discussion

### Tricky design choice #1: Dedicated Events vs. Reusing `beforeinput` / `input`

* **Option A: Extending `beforeinput` and `input` events**
  * *Design*: Use `beforeinput` with new `inputType` values (e.g. `inputType = "formatBatchStart"` / `inputType = "formatBatchEnd"`) or fire a `beforeinput` event before the batch transaction and an `input` event after the batch settles.
  * *Why reconsidered*: 
    1. **EditContext Architecture Alignment**: `EditContext` was specifically designed to decouple text input handling from standard `contenteditable` DOM elements and the traditional `beforeinput`/`input` event pipeline. Forcing `EditContext` users back onto `beforeinput` breaks the architectural separation of `EditContext`.
    2. **Semantics of `beforeinput` Cancelability**: `beforeinput` events are cancelable to allow web applications to prevent DOM mutations. In native IME batch edits, the browser is delivering input payloads driven directly by the OS IME; canceling `beforeinput` does not cleanly map to OS-level `InputConnection#beginBatchEdit` semantics.
    3. **Multiple Text Mutations per Batch**: A single batch transaction can contain multiple distinct `textupdate` payload events. If `beforeinput`/`input` wrap individual text updates, they fail to signal outer batch boundaries. If they wrap the entire batch, developers lose the granular text update payloads required by `EditContext`.

* **Option B (Chosen): Dedicated `batcheditbegin` and `batcheditend` Events on `EditContext`**
  * *Design*: Add explicit lifecycle events `batcheditbegin` and `batcheditend` directly to the `EditContext` interface.
  * *Why chosen*: 
    1. **Clear Lifecycle Sequence**: Provides explicit transaction boundary signals (`batcheditbegin` $\rightarrow$ `textupdate` ... `textupdate` $\rightarrow$ `batcheditend`).
    2. **Pre-Payload Setup**: Allows web text editors to freeze interim model updates *before* the first `textupdate` payload arrives, and finalize the model transaction *after* all text updates settle.
    3. **Native EditContext Compatibility**: Integrates seamlessly alongside existing `EditContext` events (`textupdate`, `textformatupdate`, `characterboundsupdate`).

---

## Considered alternatives

### Alternative 1: Time-Based Debouncing in Web Applications
* **Description**: Web applications delay model state finalization using a time-based timer (e.g. 100ms debouncer).
* **Why Rejected**: Time-based heuristics are non-deterministic. If CPU load, garbage collection, or model processing takes longer than the timer threshold, interim model state is exposed mid-batch, causing corruption.

---

## Security and Privacy Considerations

The `batcheditbegin` and `batcheditend` events do not expose user data or cross-origin information. They only communicate the start and end boundaries of user-initiated IME editing transactions on an element where `EditContext` is already explicitly attached.

---

## Stakeholder Feedback / Opposition

- **Chromium / Blink**: In Development ([ChromeStatus Feature Entry #5130535027474432](https://chromestatus.com/feature/5130535027474432))
- **Web Developers**: In Development (Requested by web document editor developers)
- **Gecko (Mozilla)**: Pending standards position request (`mozilla/standards-positions`)
- **WebKit (Apple)**: Pending standards position request (`WebKit/standards-positions`)

---

## References & acknowledgements

TODO
