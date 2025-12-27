# Hammerspoon chooser optimization analysis

## Executive summary

The chooser extension has significant optimization opportunities across four areas: UI rendering, data flow, keyboard handling, and table view implementation. The most impactful changes involve consolidating event monitors, fixing row height configuration, and adding optional UI simplification modes.

## Completed optimizations (PR #1)

- Pre-lowercase query string once per filter (was O(n) allocations)
- Track last hovered row to avoid redundant selection updates
- Use `enumerateAvailableRowViewsUsingBlock:` for color updates
- Cache default placeholder image at init

---

## Critical issues

### 1. Keyboard event monitor explosion

**Problem:** 17 independent event monitors are created when the chooser opens. Each monitor receives every keystroke and performs modifier checking + string comparison.

**Location:** `HSChooser.m:93-114` (`windowDidBecomeKey:`)

**Current code:**
```objc
[self addShortcut:@"1" keyCode:-1 mods:NSEventModifierFlagCommand handler:^{...}];
[self addShortcut:@"2" keyCode:-1 mods:NSEventModifierFlagCommand handler:^{...}];
// ... 15 more times
```

**Impact:**
- 17 monitors × every keystroke = 17 callback invocations per character typed
- Each callback: modifier check + `charactersIgnoringModifiers` + string comparison
- ~51 comparisons per keystroke minimum

**Solution:** Single unified handler with hash table dispatch:
```objc
- (void)setupKeyboardHandling {
    NSDictionary *handlers = @{
        @"cmd:1": ^{ [self tableView:self.choicesTableView didClickedRow:0]; },
        @"cmd:2": ^{ [self tableView:self.choicesTableView didClickedRow:1]; },
        // ...
    };

    id monitor = [NSEvent addLocalMonitorForEventsMatchingMask:NSEventMaskKeyDown
                  handler:^NSEvent*(NSEvent* event) {
        NSString *key = [NSString stringWithFormat:@"%@:%@",
                        [self modifierString:event.modifierFlags],
                        event.charactersIgnoringModifiers];
        dispatch_block_t handler = handlers[key];
        if (handler) { handler(); return nil; }
        return event;
    }];
}
```

**Estimated improvement:** 70-85% reduction in keystroke handling overhead

---

### 2. Conflicting row height configuration

**Problem:** XIB specifies both fixed row height AND automatic row heights:
```xml
<tableView rowHeight="40" usesAutomaticRowHeights="YES" ...>
```

**Location:** `HSChooserWindow.xib:50`

**Impact:** Forces NSTableView to calculate row heights for every row despite fixed setting.

**Solution:** Remove `usesAutomaticRowHeights="YES"` from XIB.

---

### 3. No query callback debouncing

**Problem:** `queryChangedCallback` fires on every single keystroke with no delay.

**Location:** `HSChooser.m:455-462`

**Impact:** If Lua callback performs heavy operations (network, file I/O), typing becomes laggy.

**Solution:** Add optional debounce parameter to `hs.chooser:queryChangedCallback()`:
```lua
chooser:queryChangedCallback(fn, 150)  -- 150ms debounce
```

---

## High-impact optimizations

### 4. Make images optional

**Current:** Every cell renders a 36×36 image view, defaulting to `NSImageNameFollowLinkFreestandingTemplate`.

**Location:** `HSChooser.m:366`, `HSChooserWindow.xib:69-130`

**Proposal:** Add `hs.chooser:showImages(bool)` API:
```lua
chooser:showImages(false)  -- Disable image column entirely
```

**Benefits:**
- ~10% cell width recovered
- Eliminates NSImageView scaling overhead
- Faster cell configuration

**Implementation:**
- Add `showImages` property to HSChooser
- In `tableView:viewForTableColumn:row:`, hide image view when disabled
- Consider third cell variant without image view for maximum performance

---

### 5. Make keyboard shortcuts optional

**Current:** ⌘1-9 shortcuts always displayed and registered.

**Location:** `HSChooser.m:339-346` (display), `HSChooser.m:93-102` (registration)

**Proposal:** Add `hs.chooser:showShortcuts(bool)` API:
```lua
chooser:showShortcuts(false)  -- Hide ⌘1-9 column, don't register handlers
```

**Benefits:**
- ~8-10% cell width recovered (40px column)
- Eliminates 10 event monitor registrations
- Cleaner UI for mouse-centric workflows

---

### 6. Pre-index choice text for filtering

**Current:** Every keystroke performs:
- O(n) full array scan
- `lowercaseString` on every text field
- `containsString:` comparison

**Location:** `HSChooser.m:465-489`

**Proposal:** Build lowercase index when choices are set:
```objc
@property(nonatomic, retain) NSArray *lowercaseTextIndex;  // Parallel array

- (void)buildSearchIndex {
    NSMutableArray *index = [NSMutableArray arrayWithCapacity:self.currentStaticChoices.count];
    for (NSDictionary *choice in self.currentStaticChoices) {
        NSString *text = [choice objectForKey:@"text"] ?: @"";
        NSString *subText = [choice objectForKey:@"subText"] ?: @"";
        [index addObject:@{
            @"text": [text lowercaseString],
            @"subText": [subText lowercaseString]
        }];
    }
    self.lowercaseTextIndex = index;
}
```

**Benefits:**
- Eliminates repeated `lowercaseString` calls
- ~15-20% filtering CPU reduction

---

## Medium-impact optimizations

### 7. Fix scroll view settings

**Current:** XIB disables scroll optimization:
```xml
<scrollView copiesOnScroll="NO" usesPredominantAxisScrolling="NO" ...>
```

**Location:** `HSChooserWindow.xib:44-46`

**Solution:** Enable optimizations:
```xml
<scrollView copiesOnScroll="YES" usesPredominantAxisScrolling="YES" ...>
```

---

### 8. Cache getChoices() during reload

**Current:** `getChoices()` called for every row in `tableView:viewForTableColumn:row:`.

**Location:** `HSChooser.m:323`

**Solution:** Cache at start of data source cycle:
```objc
@property(nonatomic, retain) NSArray *cachedChoicesForCurrentReload;

- (void)reloadData {
    self.cachedChoicesForCurrentReload = [self getChoices];
    [self.choicesTableView reloadData];
    self.cachedChoicesForCurrentReload = nil;
}
```

---

### 9. Unified cell with toggle instead of two cell types

**Current:** Two separate XIB cell definitions (`HSChooserCell`, `HSChooserCellSubtext`).

**Location:** `HSChooserWindow.xib:69-180`

**Issue:** Cell identifier changes based on content, causing reuse pool fragmentation.

**Solution:** Single cell type with subText view hidden when nil:
```objc
cellView.subText.hidden = (subText == nil);
```

---

### 10. Remove custom vertical centering cell

**Current:** `HSChooserVerticallyCenteringTextFieldCell` calculates text bounds on every draw.

**Location:** `HSChooserVerticallyCenteringTextFieldCell.m:13-45`

**Issue:** `boundingRectWithSize:` called in `titleRectForBounds:` → `drawInteriorWithFrame:` → every render.

**Solution:** Use Auto Layout constraints for vertical centering instead of custom cell class.

**Risk:** May affect precise centering behavior.

---

## Low-impact optimizations

### 11. Pre-generate shortcut strings

**Current:**
```objc
shortcutText = [NSString stringWithFormat:@"⌘%ld", (long)row + 1];
```

**Solution:** Static array:
```objc
static NSArray *shortcutStrings;
+ (void)initialize {
    shortcutStrings = @[@"⌘1", @"⌘2", @"⌘3", @"⌘4", @"⌘5",
                        @"⌘6", @"⌘7", @"⌘8", @"⌘9", @"⌘0"];
}
```

---

### 12. Cache view references instead of tag lookups

**Current:**
```objc
NSTextField *text = [cellView viewWithTag:1];
NSTextField *shortcutText = [cellView viewWithTag:2];
```

**Issue:** `viewWithTag:` is O(n) for view hierarchy depth.

**Solution:** Already using IBOutlet properties in HSChooserCell - ensure these are used consistently.

---

## Simplification opportunities

### Minimal mode

Add a "minimal" mode that disables all optional UI elements for maximum performance:

```lua
chooser:minimal(true)
-- Equivalent to:
--   chooser:showImages(false)
--   chooser:showShortcuts(false)
--   chooser:searchSubText(false)
```

### Text-only mode

For users who only need basic text filtering:

```lua
chooser:textOnly(true)
-- Single-line cells, no images, no shortcuts, no subtext
-- Maximum performance for large lists
```

---

## Implementation priority

| Priority | Optimization | Effort | Impact |
|----------|--------------|--------|--------|
| P0 | Consolidate event monitors | Medium | High |
| P0 | Fix row height XIB setting | Low | Medium |
| P1 | Optional images API | Medium | Medium |
| P1 | Optional shortcuts API | Low | Medium |
| P1 | Query callback debouncing | Medium | Medium |
| P2 | Pre-index choice text | Medium | Medium |
| P2 | Fix scroll view settings | Low | Low |
| P2 | Cache getChoices() | Low | Low |
| P3 | Unified cell type | Medium | Low |
| P3 | Remove custom centering cell | High | Low |

---

## Files involved

| File | Lines | Changes needed |
|------|-------|----------------|
| `HSChooser.h` | 17-68 | Add new properties |
| `HSChooser.m` | 93-114, 325-378, 455-489, 622-643 | Event consolidation, optional UI, indexing |
| `HSChooserTableView.h` | 20-26 | Already optimized |
| `HSChooserTableView.m` | 13-53 | Already optimized |
| `HSChooserWindow.xib` | 44-50, 69-180 | Row height, scroll settings |
| `HSChooserVerticallyCenteringTextFieldCell.m` | 13-45 | Potential removal |
| `libchooser.m` | 148-384 | New Lua API bindings |

---

## Testing recommendations

1. **Baseline measurement:** Profile keystroke latency with 1000+ items
2. **Event monitor test:** Count callback invocations per keystroke before/after
3. **Filtering benchmark:** Time filtering with 10k items, rapid typing
4. **Memory profiling:** Track allocations during active filtering
5. **Visual regression:** Ensure UI appearance unchanged with optimizations
