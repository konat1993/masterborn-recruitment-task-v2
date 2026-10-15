# Submission: [Łukasz Konatowski]

## Time Spent

Total time: **4.5** hours (approximate)

## Ticket Triage

### Tickets I Addressed

1. **CFG-142**: Stopped stale price responses from being dropped or overwriting a newer selection. The total no longer stays at `$0.00`.
2. **CFG-148**: Turning off Include Packaging now removes Gift Wrap instead of crashing the summary.
3. **CFG-152**: Keyboard access for color swatches, a visible focus style on the quantity buttons, and returning focus to the control that opened a modal.

### Tickets I Deprioritized

I left these because a wrong total, a crash, and missing keyboard access were blocking the demo. From the ticket text, none of the rows below stop that demo.


| Ticket  | Reason                                                                                                                                                                                                                        |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-143 | Lag shows up only after 15–20 minutes and is hard to reproduce. QA is not sure whether it is preview images or resize handling. A short demo will not hit it.                                                                 |
| CFG-146 | The ticket itself calls the wrong "Last saved" time cosmetic and low priority. It does not block a correct total or a crash-free demo.                                                                                        |
| CFG-147 | Share links are reported broken, and the example name has Polish characters and an emoji. The steps are not reproducible, and engineering says it may be a local flake. I would check it after the demo blockers, not before. |
| CFG-149 | UX calls a loading state polish. The note says calculation is usually fast enough that users may not notice the old price.                                                                                                    |
| CFG-150 | Swatches wrap badly on mobile, and the screenshot is missing from the export. The reporter says it may be a column-count calculation, not CSS. It does not block the demo.                                                    |
| CFG-153 | Side-by-side compare is a new feature, estimated at 2–3 weeks. The note asks why it is in this sprint.                                                                                                                        |
| CFG-155 | Dark mode would match TechStyle's site. The ticket says that is nice to have and not critical.                                                                                                                                |
| CFG-156 | Missing `key` warnings in the console. The ticket says they cause no visible issues.                                                                                                                                          |
| CFG-157 | There is no confirmation when unsaved changes are discarded. That is a UX gap in the description, and it does not stop showing a configured product.                                                                          |
| CFG-158 | The suggestion is to memoize `product` and `currentConfig` so the tree re-renders less. I did not profile it. A stuck price, a crash, and keyboard access are clearer demo blockers than an unmeasured render count.          |




### Tickets That Need Clarification


| Ticket  | Question                                                                                                                                                                    |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 | Product wants Quick Add removed (2% usage). Customer Success wants a shortcut because TechStyle asked to show it. There is no clear decision on which ticket still applies. |
| CFG-154 | Public docs say 50+ items get 15% off. `CONTEXT.md` says 10% off from 50. The code uses `quantity > 50`, so 50 gets nothing.                                                |




### Tickets With a Misleading Priority Label


| Ticket  | Label on board | Your read   | Why                                                                                                                                                                                |
| ------- | -------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-152 | ⚪ Low          | 🔴 Critical | The Low label predates Legal. TechStyle's MSA requires WCAG 2.1 AA, so a demo without keyboard access is a contract problem.                                                       |
| CFG-158 | 🟠 High        | ⚪ Low       | High assumes TechStyle will notice lag on a weak demo machine. The description is an unmeasured optimization, not a broken flow, so I would not put it ahead of the demo blockers. |




### Tickets Where You Stopped to Ask Instead of Implementing


| Ticket            | Who would you ask          | What you'd ask                                                                                                                                                      |
| ----------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 / CFG-145 | Product and TechStyle's PM | Do we remove Quick Add, or keep it and add the shortcut for the demo? Shipping both, or either one blindly, fights the other ticket.                                |
| CFG-154           | PM / Finance               | Which rule is billable: 15% from 50 (marketing), 10% from 50 (`CONTEXT.md`), or the current `>` so 51 is the first discounted quantity? Either change moves margin. |


---



## Technical Write-Up



### Critical Issues Found



#### Issue 1: Price stays at $0.00, and an older response can win

**Ticket(s):** CFG-142

**What was the bug?**

`calculatePrice` stored an incrementing request counter on `timestamp`. The hook compared that number with `Date.now()` before applying the total, so a real price was thrown away and the placeholder `$0.00` stayed. The mock also waits 150–800ms, so responses arrive out of order.

**How did you find it?**

The network call returned a price, and `setFormattedTotal` never ran. The success check compared a counter with a clock time.

**How did you fix it?**

The hook keeps a `useRef` counter. Each call stores its own id and writes the price, the error, and `isLoading` only while that id is still the latest. I removed the counter from the price API because it was redundant.

**Why this approach?**

Always applying whichever response arrives last shows the previous configuration when a slow request finishes after a fast one. Putting `Date.now()` in `timestamp` has the same hole. A local id does not depend on that field. `useRef` survives re-renders of `ProductConfigurator`, so an in-flight call still sees a later increment.

---



#### Issue 2: Deselecting packaging crashed on a missing add-on

**Ticket(s):** CFG-148

**What was the bug?**

Gift Wrap depends on the Include Packaging switch. Cleanup of dependent add-ons ran only for `select` options, and it `splice`d the existing state array. Unchecking packaging left Gift Wrap selected.

**How did you find it?**

The steps in the ticket end in `Cannot read properties of undefined (reading 'price')`. Include Packaging is `type: 'toggle'`, so `handleOptionChange` skipped it. `handleAddOnToggle` only runs when the user clicks an add-on checkbox, not when they flip that switch.

**How did you fix it?**

Select options and switches both drop add-ons whose dependency this change turned off. The main fix was adding the `"toggle"` type to the condition, so that check also takes the switch into account.

**Why this approach?**

Dependent add-ons were already removed for `select` options: when the value no longer matches `requiredValue`, the add-on leaves the list. Include Packaging is a switch (`type: 'toggle'`), and Gift Wrap depends on it the same way Premium Engraving depends on the material select. Adding `"toggle"` to that same condition sends the switch down the existing path. A separate handler for the switch was unnecessary.

---



#### Issue 3: Keyboard users cannot finish a configuration

**Ticket(s):** CFG-152

**What was the bug?**

Color swatches were `div`s outside the tab order. Quantity buttons had no focus style. Closing a modal restored `document.activeElement`, which is often `body`, not the button that opened the dialog. The native `<select>` already moves with arrow keys. The custom control is the color group.

**How did you find it?**

The audit lists four gaps. The swatches have `role="radio"` and no `tabIndex`. The focus rule covers `.btn`, `.option-select`, `.quantity-input`, and `.toggle-switch`, not `.quantity-btn`. The modal stores `document.activeElement` inside `onClick`.

**How did you fix it?**

The selected swatch is the only tab stop. Arrow keys move the selection and call `focus()` on the next swatch. Space does not scroll the page. Quantity buttons and swatches share the existing focus outline. Drafts and Share store `e.currentTarget`, move focus to the close button, and return it on Escape.

**Why this approach?**

`tabIndex={0}` on every swatch would make Tab stop on each color. Replacing `<select>` would rebuild something the browser already does. A focus trap inside the dialog is the remaining WCAG gap. It is not included in Acceptance Criteria in the ticket, and the draft list changes which buttons exist, so I did not add one in this pass.

---



### Other Changes Made

- Removed `requestCounter` from `calculatePrice`. It only existed to fill `timestamp`.

---



## Code Quality Notes



### Things I Noticed But Didn't Fix


| Issue                                               | Why I Left It                                                                                                                                                                                      |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dialog focus can Tab onto the page behind the modal | Not in the CFG-152 list. Escape returning to the opener is what that item asks for.                                                                                                                |
| No unit or integration tests                        | There is no test runner, and adding one now would have crowded out the demo blockers. For the demo I would still want QA, or another frontend, to run the three flows by hand and review the diff. |
| `ProductConfigurator` is still one large component  | `CONTEXT.md` already calls the split a later refactor. The fixes work in place. Breaking the file up under this time limit is tech debt work, not a demo fix.                                      |
| Linter errors                                       | I noticed them and left them. The deadline was too tight to clean them up. That is tech debt, not a demo blocker.                                                                                  |




### Potential Improvements for the Future

1. Split `ProductConfigurator` into smaller components once the demo is stable. Not before: the current fixes work, and a wide move risks them.
2. Add a unit and integration test setup for the whole app, not only the three bugs. Overlapping price requests, packaging dependencies, and keyboard focus would be the first cases. That setup would have stood a strong chance of catching the bugs already described in the tickets before they reached the demo.
3. Trap focus inside dialogs, and prune dependent add-ons with one immutable `filter` so two add-ons on the same option cannot overwrite each other.

---



## Questions for the Team

1. For the TechStyle demo, do we keep Quick Add and add the shortcut, or remove the feature as Product asked?
2. Which quantity discount is the one we are allowed to charge: 15% from 50, 10% from 50, or the current behavior where 51 is the first discounted quantity?
3. After the demo, do you want a test runner in this package, or will QA stay manual until the widget is split up?
4. If I could reach the PO before coding, I would ask whether the packaging crash or the WCAG gaps should go first. The board order was not set by the person who talks to TechStyle. I did not have that channel, so I shipped the stuck price, then the crash, then keyboard access.

---



## Assumptions Made

1. The demo bar is a correct total, no white screen, and enough keyboard support to avoid shipping an obvious WCAG miss. Three fixes done properly beat a shallow pass over the rest of the board.
2. Quick Add stays as it is until Product and TechStyle agree. I did not remove it and I did not add a shortcut.
3. I did not change discount thresholds. `CONTEXT.md`, the marketing note, and the code disagree, and the wrong edit costs money.

---



## Self-Assessment



### What went well?

I read the tickets, `CONTEXT.md`, and the README first, and picked three tasks from that. That kept me off the rest of the board. Each of those three got finished and checked in the running app, instead of a light pass over more tickets.

### What was challenging?

The price guard looks correct until you see that `timestamp` is not a time. `useRef` also looks like it resets on every render, and it does not. The packaging path looked fixed once the checkbox disappeared, while `setSelectedAddOns` was still handing React the same array.

### What would you do differently with more time?

First I would take CFG-147, the next Critical ticket: share links that break for some configurations. Then I would set up unit and integration tests for the whole app, starting with those three paths, and trap focus in the dialogs. I would not start the compare feature, dark mode, or a `useMemo` pass.

---



## AI Assistance Notes (optional)

I used Cursor to read the repo and trace the price hook against `calculatePrice`, and later to draft this write-up, which I then reviewed.

The first triage it offered followed the three Jira Critical labels. I rejected that order. CFG-152 is marked Low, and the TechStyle MSA makes a demo without keyboard access a contract problem, so it replaced a board Critical in the set I actually fixed. CFG-147 is marked Critical, and from the ticket I would look at it after those three, not instead of the WCAG work. I did not verify it.

I found and fixed the price path and the packaging crash myself, then checked both in the running app.

Some suggestions went past the ticket. They would have improved the app, but they were not required to close it. That slowed me down: I had to see why a change was proposed, talk it through, and revert it. I wanted the required fix only, and any extra improvement left for later.

Reading the repo and the markdown, plus the first ticket, took a little over two hours, so the next two had less time. For the WCAG ticket I asked Cursor to write the solution, and then I reviewed it and applied my own changes. I usually find and fix the bug myself first. There was not enough time left to do that on this one.

---



## Additional Notes

None