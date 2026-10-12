# A save error should leave a recoverable editing state

An editor can display an accurate error message and still leave its data in the wrong state. The message explains that a value was rejected, while an export quietly includes that value. The user now has to work out which version the application believes.

For a small offline editor, I would make recovery part of the save contract. After validation fails, the user should retain the text they typed, the accepted document should retain its previous value, and the next action should have a clear result. These are separate requirements. Showing an alert satisfies none of them by itself.

## Give the draft its own place

Consider a localization editor with an imported array of rows and a text area for the selected translation. The text area holds a draft. The array holds the document accepted by the application. The two can differ while someone is typing.

This distinction becomes useful when a translation exceeds the file format's field limit. Replacing the text area with the old value would discard work that the user could shorten. Copying the oversized value into the accepted array would weaken the document's validity. Keeping both versions gives the editor a way to reject the change without losing the attempt.

A third state describes persistence. An accepted value in a JavaScript object is still only in page memory. It is not evidence that an exported file exists on disk. I would describe the states explicitly:

| State | What the application knows | Useful message |
| --- | --- | --- |
| Pending draft | The control differs from the last accepted value | Review and apply this edit |
| Accepted in memory | The candidate passed the chosen document checks | Applied in this page; export to keep a copy |
| Download requested | The application initiated a browser download | Check that the file was saved |

The last row deliberately makes a narrow claim. MDN notes that the anchor element's [download property cannot establish whether a download will occur](https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement/download). A generated filename and a click are insufficient evidence for a stronger success message.

## Validate before changing the accepted document

The order of operations matters more than the presence of a catch handler. Assigning a new value, marking the edit as applied, and then validating allows a failed check to leave earlier assignments intact. JavaScript's [try/catch construct transfers control when an exception is thrown](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch); it does not supply an application transaction.

In a local localization prototype, the save path was changed to build a candidate row array and validate it before assigning the edited target. It also checks glossary syntax before that assignment. The draft remains pending when either of those checks rejects it. Export calls the same commit operation first, so these failures stop that export path before it reaches the download step.

That ordering has a limited scope. The prototype still updates its issue display after assigning the accepted value. A later rendering error would happen after the assignment. Describing the whole handler as atomic would overstate what this change provides. The improvement is protection against the specified document and glossary validation failures, not rollback for every possible exception.

For a more demanding editor, I would compute all required validation results before the state transition, then keep display failures separate from document rejection. The interface could report that a value was applied but the findings could not be refreshed. That is a proposed design direction, not a claim that the prototype already handles every failure that way.

## Make navigation obey the same rule

Save is rarely the only route out of an editing state. A row selection, export button, or import operation may also cause the editor to leave the draft behind.

I prefer an explicit policy for each route. Row selection can attempt to apply the draft and stay on the current row if validation fails. Export can block while an invalid draft remains unresolved. Import can ask before discarding pending work. Another product might allow exporting the last accepted document, but it should label that choice clearly: the exported file would exclude the visible draft.

The particular policy is a product decision. Consistency is the engineering requirement. A strict Save button combined with a permissive Export button leaves a second path around the same rule. Sharing the validation boundary reduces that risk, provided callers stop when it fails.

## Test what the user can do next

A useful regression checks the state after rejection and the recovery path. For the oversized-field case, I would look for four observations: the accepted row is unchanged, the draft text remains available, export has not been initiated, and a corrected draft can subsequently be accepted and exported.

The prototype's existing local regression record covers rejection, blocked export with a pending edit, and successful correction into reimportable rows. That is narrower than a complete browser workflow test. It does not establish successful disk persistence, every navigation route, or crash recovery. Those require their own evidence before the interface promises them.

This is why I would prioritize recoverable state over elaborate error wording. A plain message is useful when the user's text is still present and the next action is predictable. Even a polished message cannot repair a document that changed before the application decided to reject it.

For an offline editor, the practical contract is small: keep the draft, preserve the last accepted value when validation fails, and distinguish applying an edit from saving a file. Those boundaries make failure easier to understand and correction easier to finish.
