# Squad-Informed QA Report — Smart Shelf Prototype

**Environment:** 1280x720 headless Chromium, localhost:5173, desktop only
**Date:** 2026-03-21
**Tester:** Claude (automated, single viewport)
**Methodology:** 5-persona squad (Cannabis SME, UX Design, PM, FE Engineer, Data Analyst) generated domain-informed test scenarios, then executed against the live prototype via headless browser.

---

## Results Summary

| # | Persona | Scenario | Result | Severity |
|---|---------|----------|--------|----------|
| 1 | FE Eng | Cart survives mode switch | **PARTIAL FAIL** | Medium |
| 2 | FE Eng | Cart survives page refresh | **FAIL** | Low (prototype) |
| 3 | Cannabis SME | Effect-based discovery accuracy | **PASS** | — |
| 4 | UX Design | Slide-over preserves scroll position | **PASS** | — |
| 5 | UX Design | Checkout accordion progress | **PASS** | — |
| 6 | FE Eng | Nonsense search empty state | **FAIL** | Medium |
| 7 | PM | Concierge cart count display | **BUG** | Medium |

---

## Detailed Findings

### 1. Cart state on mode switch — PARTIAL FAIL

- Cart data is shared across modes (same `useStore` hook in parent) — good
- Smart Shelf showed 2 items / $54 correctly
- Concierge and Classic both showed **1 item / $45** despite 2 items being in the store
- **Root cause:** Likely each variant renders its own cart UI from the same store, but Concierge/Classic may have a different rendering path that doesn't pick up all items, or the store emitted before both adds completed
- **Risk:** Demo killer. "Now let me show you Concierge mode" → cart looks wrong

### 2. Cart persistence on refresh — FAIL

- No `localStorage` persistence. Cart empties on F5.
- `localStorage` has zero cart-related keys
- **Risk:** Low for a prototype demo (you don't refresh mid-demo), but any real user testing would flag this immediately. Trivial fix (~10 lines to add localStorage sync to the store hook).

### 3. Effect-based discovery — PASS

- Clicked Effects → Relaxed → 47 products returned
- Opened OG Kush detail → "Relaxed" effect tag confirmed in detail view
- Terpene profile (Myrcene, Caryophyllene) consistent between card and slide-over
- AI note correctly says "Matched against your Relaxed & Creative preference"
- **One concern I can't verify:** Whether ALL 47 products have "Relaxed" tagged. Would need a data audit script, not click testing.

### 4. Slide-over scroll preservation — PASS

- Scrolled to row 3 in Flower grid → opened Fig Farms Soap detail → closed → same scroll position
- Background content did NOT scroll while slide-over was open (backdrop blocks interaction)
- **Can't verify:** Focus trapping (keyboard tab order), mobile drag-to-dismiss behavior

### 5. Checkout accordion — PASS

- Step 1: Pickup details → "Continue to Payment" → collapses with checkmark + address summary
- Step 2: Payment → "Review Order" → collapses with checkmark + "Chase ••6789"
- Step 3: Review → "Place Order" CTA with ID verification badge
- "Continue Shopping" button returns to browse state
- **Can't verify:** Whether you can skip ahead (click step 3 before completing step 1)

### 6. Nonsense search empty state — FAIL

- Searched "xylophone" → Gemini processes it → returns AI response that maps to default home shelves
- No clear "we don't sell that" or "no results" message
- The breadcrumb shows "xylophone" but the content looks like the home page
- **Risk:** In a demo, someone WILL type something weird. The lack of a clear empty state makes the AI look confused rather than gracefully declining. Should show something like "I couldn't find 'xylophone' in our menu. Try one of these:" with suggestion chips.

### 7. Concierge cart count discrepancy — BUG

- Added 2 items in Smart Shelf (Blue Dream $45 + another totaling $54)
- Switched to Concierge → showed "1 item" / $45
- Switched to Classic → showed "Cart (1)" / $45
- Switched back to Smart Shelf → showed 2 items / $54 correctly
- **Hypothesis:** The second add-to-cart animation may not have completed before mode switch, or Concierge/Classic read the cart before the state update propagated.

---

## What Could Not Be Tested (honest limitations)

| What | Why |
|------|-----|
| Mobile viewport (375px) | Headless browser locked at 1280x720 |
| Filter chip overflow on small screens | Same — need real device or Playwright viewport |
| Touch interactions (swipe, drag-to-dismiss) | No touch simulation |
| Hover states / transitions / animations | Screenshots are static frames |
| Scroll bleed-through on mobile slide-over | No mobile viewport |
| Keyboard accessibility / focus trapping | Can't tab through elements |
| Analytics event firing | No tracking implemented (prototype) |
| Cart weight/quantity legal limits | Not implemented (prototype) |
| Rapid input / debounce behavior | Can't type fast enough via automation |
| All 5 Data Analyst scenarios | No analytics layer exists |

---

## Assessment for SWE Team

**Can this methodology replace manual QA?** No. It catches structural bugs (cart state, missing empty states, data consistency) and verifies happy paths. It completely misses responsive, touch, accessibility, and animation issues — which are >50% of real QA findings on a commerce product.

**Where it adds value:**
- Generating domain-informed test plans (the squad personas surface scenarios an engineer wouldn't think of)
- Running regression checks after changes ("did the filter chips break the carousel?")
- Catching state management bugs across mode boundaries
- Documenting findings in a format that goes straight into a ticket

**What you'd still need:**
- Playwright/Cypress for real viewport testing at 375px, 768px, 1024px
- A human QA pass for "does this feel right" subjective quality
- Accessibility audit (axe-core or manual screen reader testing)
- Real device testing for touch interactions

---

## Full Test Plan (15 scenarios from 5 personas)

### Cannabis Industry SME

**1. Effect-Based Discovery Actually Works**
Journey: Effects browse → select "Relaxed" → review returned products → open detail → verify effects and terpenes match.
Pass criteria: Every product has the effect tagged. Terpene profiles consistent between card and detail. Not flower-only bias.

**2. Cart Handles Legal Weight/Quantity Limits**
Journey: Add 28g flower → add another 7g → check for warning or limit enforcement.
Pass criteria: Cart displays running weight/unit totals. Warning before exceeding state limits.

**3. Concierge Handles Ambiguous Ask**
Journey: Concierge mode → "pain but not couch-locked" → review recommendations.
Pass criteria: Balanced hybrids or CBD-rich options, not heavy indicas.

### UX Design SME

**4. Slide-Over Does Not Trap Focus or Break Scroll**
Journey: Browse category → scroll down → open product detail → scroll within → close → verify scroll position preserved.
Pass criteria: No scroll bleed-through. Scroll position preserved on close.

**5. Filter Chips Are Usable on 375px Screen**
Journey: On 375px → Flower category → interact with filter chips → verify no overflow.
Pass criteria: Chips horizontally scrollable. Dropdowns don't overflow viewport.

**6. Checkout Accordion Communicates Progress**
Journey: Proceed through checkout steps → verify collapse/expand behavior and progress indicators.
Pass criteria: Completed sections show checkmark + summary. Clear visual hierarchy.

### PM (Product Manager)

**7. Buy Again Carousel Drives Repeat Purchase**
Journey: Home page → Buy Again carousel → tap product → add to cart → checkout.
Pass criteria: Carousel appears high in visual hierarchy. ≤4 taps from home to checkout.

**8. Concierge-to-Cart Handoff Is Seamless**
Journey: Concierge → add product via chat → switch to Smart Shelf → verify cart.
Pass criteria: Cart state persists across modes. Same product details. No duplicates.

**9. Classic Mode Still Converts Without AI**
Journey: Classic mode → browse → filter → add → checkout.
Pass criteria: Full purchase flow works without AI features.

### FE Engineer

**10. Cart State Survives Mode Switching and Page Refresh**
Journey: Add items in Smart Shelf → switch modes → refresh → verify cart.
Pass criteria: Cart persists in localStorage. Mode switches don't clear cart.

**11. Mobile Bottom Sheet Cart vs. Desktop Sidebar Cart**
Journey: Desktop sidebar → resize to 768px → verify transition to bottom sheet.
Pass criteria: Both views functional. No desync between cart components.

**12. AI Search Handles Rapid Input and Empty States**
Journey: Type partial query → clear → type new query → submit → search nonsense term.
Pass criteria: No stale results. Debounced API calls. Clear empty state for nonsense.

### Data Analyst

**13. Mode Switch Funnel Is Trackable**
Journey: Switch between all modes → add item → verify attribution path is capturable.
Pass criteria: Discrete analytics events per mode switch. Session ID consistent.

**14. Search-to-Cart Drop-Off Is Measurable**
Journey: Search → view product → close without adding → re-search → add to cart.
Pass criteria: Each search logged with result count. Close-without-add event captured.

**15. Deals Carousel Engagement vs. Position**
Journey: Scroll deals carousel → click visible and off-screen deals.
Pass criteria: Carousel scroll events fire with depth. Click events include card index and initial visibility.

---

*Generated by Claude with 5-persona squad methodology. Bugs #1, #2, #6 subsequently fixed in the same session.*
