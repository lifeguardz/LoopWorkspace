# Feature: HealthKit Carb Import with Confirmation UI

## Overview

Add ability to import carbohydrate entries from Apple Health (HealthKit) into Loop. Users can select which carb entries from other apps (e.g., MyFitnessPal, Apple Health manual entries) to import into Loop's carb log.

## Business Requirements

### User Stories

1. **As a Loop user**, I want to import carb entries from HealthKit so I don't have to manually enter carbs that I've already logged in other diabetes/food tracking apps.

2. **As a Loop user**, I want to review and select which HealthKit carb entries to import so I can avoid duplicates or incorrect entries.

3. **As a Loop user**, I want imported carbs to be treated the same as manually entered carbs so Loop's algorithm works correctly.

### Functional Requirements

1. **Import Trigger**
   - Manual trigger via button in CarbEntryView
   - Button labeled "Import from HealthKit" (or similar)
   - Button only appears for new carb entries (not editing existing)

2. **Data Source**
   - Query HealthKit for dietary carbohydrate samples
   - Filter for entries from OTHER apps only (exclude Loop's own entries)
   - Default time range: last 24 hours
   - Maximum entries to display: 100

3. **Display**
   - List view showing:
     - Grams of carbs (in large text)
     - Timestamp (date and time)
     - Source app name (e.g., "MyFitnessPal", "Apple Health")
     - Food type/description (if available in HealthKit metadata)
   - Multi-select with toggle switches
   - Selected count displayed at bottom
   - Loading spinner while fetching
   - Empty state when no carbs found

4. **User Actions**
   - Cancel: close without importing
   - Import: add selected entries to Loop's carb log
   - Each entry selected gets added to CarbStore

5. **Data Handling**
   - Use HealthKit sample timestamp as carb entry date
   - Extract food type from `HKMetadataKeyFoodType` (if available)
   - Use default absorption time (medium: ~3 hours)
   - Store as `NewCarbEntry` in CarbStore
   - Avoid duplicates (check against existing entries by UUID)

6. **Error Handling**
   - No HealthKit permissions: show error message
   - No external carb entries: show empty state message
   - Fetch error: show error with retry option
   - Import failure: show error message

### Non-Functional Requirements

1. **Performance**
   - HealthKit query should complete within 2 seconds
   - UI should remain responsive during fetch
   - Import should process entries asynchronously

2. **Privacy**
   - Only read from HealthKit (no write permissions needed beyond what Loop already has)
   - Respect user's HealthKit privacy settings
   - No data sent to external servers

3. **Compatibility**
   - iOS 15.0+ (Loop's minimum deployment target)
   - SwiftUI-based UI
   - Works on iPhone and Apple Watch (if applicable)

## Technical Requirements

### Affected Files

**Modified Files:**
- `Loop/Loop/View Models/CarbEntryViewModel.swift`
  - Add `healthStore` computed property (optional, or access via delegate)
  - No new protocol requirements needed (DeviceDataManager already has these properties)

**New Files:**
- `Loop/Loop/View Models/HealthKitCarbImportViewModel.swift`
  - ObservableObject class
  - Fetch carb samples from HealthKit
  - Manage selection state
  - Import selected samples via CarbStore
  
- `Loop/Loop/Views/HealthKitCarbImportView.swift`
  - SwiftUI View
  - List of carb entries with multi-select
  - Import/Cancel buttons
  - Loading/error/empty states

### Implementation Notes

#### CarbEntryViewModel Changes (Optional)

```swift
// Option 1: Access via delegate (DeviceDataManager already conforms)
var healthStore: HKHealthStore? {
    return delegate?.healthStore
}

// Option 2: No changes needed - DeviceDataManager already has:
// let healthStore: HKHealthStore
// let carbStore: CarbStore
// These are automatically available via CarbEntryViewModelDelegate
```

**CRITICAL: DeviceDataManager already has `healthStore` and `carbStore` properties. No protocol changes needed.**

#### HealthKit Query

```swift
func fetchHealthKitCarbs() {
    guard let carbType = HKQuantityType.quantityType(forIdentifier: .dietaryCarbohydrates) else {
        return
    }
    
    let now = Date()
    let startDate = now.addingTimeInterval(-24 * 60 * 60) // Last 24 hours
    let predicate = HKQuery.predicateForSamples(withStart: startDate, end: now, options: [.strictStartDate])
    
    // Filter for entries NOT from this app (other apps only)
    let notFromCurrentApp = NSCompoundPredicate(notPredicateWithSubpredicate: 
        HKQuery.predicateForObjects(from: HKSource.default()))
    let finalPredicate = NSCompoundPredicate(andPredicateWithSubpredicates: [predicate, notFromCurrentApp])
    
    let query = HKSampleQuery(sampleType: carbType, predicate: finalPredicate, limit: 100, sortDescriptors: [NSSortDescriptor(key: HKSampleSortIdentifierStartDate, ascending: false)]) { _, samples, _ in
        // Process samples
    }
    
    healthStore.execute(query)
}
```

#### CarbStore Integration

```swift
func importSelectedSamples(completion: @escaping (Bool) -> Void) {
    for sample in selectedSamples {
        let entry = NewCarbEntry(
            date: sample.startDate,
            quantity: HKQuantity(unit: .gram(), doubleValue: sample.quantity.doubleValue(for: .gram())),
            startDate: sample.startDate,
            foodType: sample.metadata?[HKMetadataKeyFoodType] as? String,
            absorptionTime: defaultAbsorptionTimes.medium // Use default
        )
        
        carbStore.addCarbEntry(entry) { result in
            // Handle result
        }
    }
}
```

### Patch Generation CRITICAL REQUIREMENTS

#### Directory Structure Context

```
LoopWorkspace/                    # Root workspace
├── patches/                      # Patches go here
├── Loop/                         # Git submodule
│   ├── Common/                   # Note: ONE level deep
│   │   └── Models/
│   │       └── WatchContext.swift
│   └── Loop/                     # Note: TWO levels deep (Loop/Loop/)
│       ├── View Models/
│       │   ├── CarbEntryViewModel.swift
│       │   └── [NEW] HealthKitCarbImportViewModel.swift
│       └── Views/
│           ├── CarbEntryView.swift
│           └── [NEW] HealthKitCarbImportView.swift
```

#### Patch File Format (CRITICAL)

**For files in `Loop/Loop/` (TWO levels deep):**

```diff
diff --git a/Loop/Loop/View Models/CarbEntryViewModel.swift b/Loop/Loop/View Models/CarbEntryViewModel.swift
--- a/Loop/Loop/View Models/CarbEntryViewModel.swift
+++ b/Loop/Loop/View Models/CarbEntryViewModel.swift
```

**Both paths MUST be identical and include `Loop/Loop/`**

**For NEW files:**

```diff
diff --git a/Loop/View Models/HealthKitCarbImportViewModel.swift b/Loop/View Models/HealthKitCarbImportViewModel.swift
new file mode 100644
--- /dev/null
+++ b/Loop/View Models/HealthKitCarbImportViewModel.swift
```

**Both paths should be `Loop/View Models/` (ONE level deep for new files inside submodule)**

#### GitHub Actions Context

Patches are applied via:
```bash
git apply ./patches/* --allow-empty -v --whitespace=fix
```

This runs from **workspace root**, NOT from inside submodule.

#### Testing Patch Locally

```bash
# From workspace root:
cd LoopWorkspace
git submodule update --init --recursive
git apply ./patches/healthkit-carb-import.patch --allow-empty -v --whitespace=fix

# Verify files are in correct location:
ls -la Loop/Loop/View\ Models/HealthKit*.swift
ls -la Loop/Loop/Views/HealthKit*.swift
```

Files MUST appear in `Loop/Loop/`, NOT at workspace root.

### Common Pitfalls to Avoid

1. **Wrong Path Depth**
   - ❌ `Loop/View Models/` (one level - creates file at wrong location)
   - ✅ `Loop/Loop/View Models/` (two levels - correct location)

2. **Protocol Changes**
   - ❌ Adding new protocol requirements to `CarbEntryViewModelDelegate`
   - ✅ Using existing properties from `DeviceDataManager` (already has `healthStore`, `carbStore`)

3. **Missing Imports**
   - Add `import HealthKit` to files using HKHealthKit
   - Add `import Combine` for ObservableObject

4. **Force Unwrapping**
   - ❌ `delegate!.healthStore`
   - ✅ `delegate?.healthStore` (optional chaining)

5. **Patch Paths**
   - Always verify with `head -10 patches/your-feature.patch`
   - Both `---` and `+++` lines should have SAME path for modified files
   - New files should have `/dev/null` as source

## Acceptance Criteria

### Must Have
- [ ] Button in CarbEntryView to trigger import
- [ ] List shows carb entries from HealthKit (last 24h, external apps only)
- [ ] Multi-select with toggle switches
- [ ] Import adds selected entries to CarbStore
- [ ] Basic error handling (no permissions, no data)

### Should Have
- [ ] Loading spinner during fetch
- [ ] Empty state message
- [ ] Source app name displayed
- [ ] Food type from metadata (if available)

### Nice to Have
- [ ] Date range picker (custom time ranges)
- [ ] Select all/clear all buttons
- [ ] Search/filter functionality
- [ ] Undo import feature

## Testing Checklist

### Unit Tests (if applicable)
- [ ] HealthKit query filters correctly (external apps only)
- [ ] CarbStore receives correct NewCarbEntry objects
- [ ] Deduplication logic works

### Integration Tests
- [ ] Import flow works end-to-end
- [ ] Cancel doesn't add entries
- [ ] Import adds multiple entries correctly

### Manual Testing
- [ ] Add carb entry in Apple Health app
- [ ] Sync from MyFitnessPal (if available)
- [ ] Open Loop → Add Carbs → Import from HealthKit
- [ ] Verify all entries appear
- [ ] Select and import
- [ ] Check Loop's carb log shows imported entries
- [ ] Verify no duplicates
- [ ] Test error scenarios (no permissions, no data, network error)

### Edge Cases
- [ ] Import 0 entries
- [ ] Import 100+ entries (max limit)
- [ ] Entry with missing food type
- [ ] Entry with very large carb value
- [ ] Duplicate import (same UUID)
- [ ] Import while offline
- [ ] Background app refresh

## Implementation Phases

### Phase 1: Core Functionality
- CarbEntryViewModel extension (optional)
- HealthKitCarbImportViewModel
- HealthKitCarbImportView
- Basic import flow

### Phase 2: Polish
- Error states
- Empty states
- Loading states
- Better UI/UX

### Phase 3: Enhancements (Future)
- Custom date ranges
- Batch operations
- Advanced filtering
- Undo/review

## Related Files

- `LoopKit/LoopKit/CarbKit/CarbStore.swift` - Manages carb entries
- `Loop/Loop/View Models/CarbEntryViewModel.swift` - Existing carb entry view model
- `Loop/Loop/Views/CarbEntryView.swift` - Existing carb entry UI
- `DeviceDataManager.swift` - Has `healthStore` and `carbStore` properties

## Dependencies

- HealthKit framework (already used by Loop)
- LoopKit module (provides CarbStore, NewCarbEntry)
- SwiftUI (already used by Loop)
- Combine (already used by Loop)

## Security & Privacy

- No new permissions required (Loop already has HealthKit read/write)
- Only reads from HealthKit (no new write operations)
- Data stays on device
- No external API calls

## Performance Considerations

- HealthKit queries are async
- Limit to 100 entries max
- Use background thread for processing
- Cache recent queries (optional optimization)

## Accessibility

- VoiceOver support for all text
- Dynamic Type support
- High contrast colors
- Haptic feedback for actions

## Localization

- All user-facing strings should use `NSLocalizedString`
- Key strings:
  - "Import from HealthKit"
  - "Select carbs to import"
  - "Import"
  - "Cancel"
  - "No carb entries found"
  - "Loading..."
  - Food type labels
  - Error messages

## Documentation

- Code should include inline comments for complex logic
- README update with new feature description
- CHANGELOG entry for user-facing changes

## Questions for Future Consideration

1. Should we show carb entries from Loop itself (for review purposes)?
2. Should we allow editing imported carbs before confirming?
3. Should we cache HealthKit results locally?
4. Should we show nutritional info beyond carbs (protein, fat)?
5. Should we integrate with specific food tracking apps' APIs?
6. Should we support background sync of carbs?
7. Should we show which entries were imported vs manual?

## Success Metrics

- Users import carbs from HealthKit successfully
- No increase in app crashes
- No performance degradation
- User feedback positive
- Feature used regularly by target users

## Timeline Estimate

- Phase 1: 2-3 days
- Phase 2: 1-2 days  
- Phase 3: Future iterations
- Testing: 1-2 days

## Resources

- [Apple HealthKit Documentation](https://developer.apple.com/documentation/healthkit)
- [LoopKit CarbStore Reference](LoopKitSources)
- Existing CarbEntryView implementation
- DeviceDataManager for CarbStore access patterns