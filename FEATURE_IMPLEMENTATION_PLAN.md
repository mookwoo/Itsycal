# Itsycal Feature Enhancement Plan

This document outlines the implementation strategy for six major feature enhancements to Itsycal. Each section includes technical considerations, architecture recommendations, and implementation steps.

---

## 1. Natural Language Event Creation

**Complexity: High** | **Priority: High** | **Estimated Effort: 2-3 weeks**

### Overview
Add quick event entry like "Lunch with Bob tomorrow at noon" or "Team meeting next Tuesday 2pm" that parses and creates events automatically.

### Technical Approach

#### Dependencies
- **NSDataDetector** (built-in): For detecting dates, times, and addresses
- **NSLinguisticTagger** (built-in): For natural language processing
- **Alternative**: Consider integrating a third-party NLP library like [Chronic](https://github.com/mojombo/chronic) (Ruby) or similar Objective-C/Swift parser

#### Implementation Strategy

1. **Create NaturalLanguageParser Class**
   - New class: `MoNaturalLanguageParser`
   - Parse input string for:
     - Date/time references (today, tomorrow, next Tuesday, etc.)
     - Duration (1 hour, 30 minutes, etc.)
     - Event title (remaining text after extraction)
     - Location (if "at" keyword is detected)

2. **Enhance EventViewController**
   - Add a new quick-entry text field above existing form
   - Real-time parsing feedback (show interpreted date/time)
   - "Use Quick Entry" vs "Use Form" toggle

3. **Parser Components**
   ```objc
   @interface MoNaturalLanguageParser : NSObject
   
   + (EventParseResult *)parseEventString:(NSString *)input 
                          relativeToDate:(NSDate *)referenceDate 
                                calendar:(NSCalendar *)calendar;
   
   @end
   
   @interface EventParseResult : NSObject
   @property (nonatomic, strong) NSString *title;
   @property (nonatomic, strong) NSDate *startDate;
   @property (nonatomic, strong) NSDate *endDate;
   @property (nonatomic, strong) NSString *location;
   @property (nonatomic) BOOL isAllDay;
   @property (nonatomic) NSInteger confidence; // 0-100
   @end
   ```

4. **Parsing Rules**
   - Detect relative dates: today, tomorrow, tonight, this/next/last [day of week]
   - Detect absolute dates: January 15, 12/25, etc.
   - Detect times: noon, midnight, 2pm, 14:00, 2:30, etc.
   - Extract prepositions: "at" (location), "with" (participants - for title)
   - Handle duration: "for 2 hours", "1.5h", "30min"
   - Default duration: 1 hour for events with time, all-day for date-only

5. **UI Flow**
   - Quick entry field at top of EventViewController
   - As user types, show preview of parsed event below
   - "Create Event" button or press Enter to accept
   - "Edit Details" button to transfer to full form
   - Error state if parsing fails or confidence is low

### Files to Modify/Create
- **New**: `Itsycal/MoNaturalLanguageParser.h`
- **New**: `Itsycal/MoNaturalLanguageParser.m`
- **Modify**: `Itsycal/EventViewController.h`
- **Modify**: `Itsycal/EventViewController.m`
- **Modify**: `Itsycal/Base.lproj/Localizable.strings`

### Testing Considerations
- Test various date formats and languages
- Handle ambiguous inputs gracefully
- Support all localized date/time formats
- Edge cases: past dates, very distant future dates

### Future Enhancements
- Learn from user corrections
- Support recurring event phrases ("every Monday")
- Integration with contacts for meeting participants

---

## 2. Time Zone Support

**Complexity: Medium-High** | **Priority: Medium** | **Estimated Effort: 2-3 weeks**

### Overview
Show multiple time zones in the calendar, add events in different time zones, and display colleague's time zones for remote teams.

### Technical Approach

#### Core Components

1. **Time Zone Manager**
   ```objc
   @interface MoTimeZoneManager : NSObject
   
   @property (nonatomic, strong) NSArray<NSTimeZone *> *trackedTimeZones;
   @property (nonatomic, strong) NSTimeZone *primaryTimeZone;
   
   + (instancetype)sharedManager;
   - (void)addTimeZone:(NSTimeZone *)timeZone;
   - (void)removeTimeZone:(NSTimeZone *)timeZone;
   - (NSString *)formattedTimeForDate:(NSDate *)date inTimeZone:(NSTimeZone *)timeZone;
   
   @end
   ```

2. **UI Components**

   **A. Time Zone Clock View (Menu Bar Dropdown)**
   - Show current time in each tracked time zone
   - Add above or below calendar in main window
   - Compact view: City name + time
   - Click to expand for more details

   **B. Event Creation Enhancement**
   - Add time zone picker in EventViewController
   - Store time zone with event (use EKEvent's timeZone property)
   - Show warning when creating events in different time zones
   
   **C. Event Display Enhancement**
   - Show time zone abbreviation for events in non-local time zones
   - In agenda view: "2:00 PM PST" vs "2:00 PM" for local
   - Tooltip shows full time zone name and local equivalent

3. **Data Persistence**
   - Store tracked time zones in NSUserDefaults
   - Key: `kTrackedTimeZones` (array of time zone identifiers)

4. **Calendar Integration**
   - EventKit already supports time zones on EKEvent
   - Use `event.timeZone` property
   - For all-day events, timeZone should be nil (as currently implemented)

### Implementation Steps

1. **Create MoTimeZoneManager**
   - Singleton pattern for managing time zones
   - Load/save from NSUserDefaults
   - Notification when time zones change

2. **Add Time Zone Clock View**
   - New class: `TimeZoneClockViewController`
   - Show in ViewController above calendar
   - Collapsible/expandable section

3. **Enhance EventViewController**
   - Add time zone popup button
   - Show "Time Zone" label and picker
   - Default to local time zone
   - Save selected time zone with event

4. **Update AgendaViewController**
   - Detect when event time zone differs from local
   - Add time zone abbreviation to time display
   - Show both local and event time in tooltip

5. **Preferences Integration**
   - Add "Time Zones" tab in PrefsVC
   - List of tracked time zones with add/remove
   - Time zone search/picker
   - Reorder time zones

### Files to Modify/Create
- **New**: `Itsycal/MoTimeZoneManager.h`
- **New**: `Itsycal/MoTimeZoneManager.m`
- **New**: `Itsycal/TimeZoneClockViewController.h`
- **New**: `Itsycal/TimeZoneClockViewController.m`
- **New**: `Itsycal/PrefsTimeZonesVC.h`
- **New**: `Itsycal/PrefsTimeZonesVC.m`
- **Modify**: `Itsycal/EventViewController.m` (add time zone picker)
- **Modify**: `Itsycal/AgendaViewController.m` (display time zones)
- **Modify**: `Itsycal/ViewController.m` (integrate clock view)
- **Modify**: `Itsycal/Itsycal.h` (add constants)

### UI Considerations
- Keep compact to avoid cluttering the interface
- Make time zone display optional (preference to hide)
- Use system time zone picker where possible
- Consider showing world clock as separate popover to avoid bloat

### NSUserDefaults Keys
```objc
NSString * const kTrackedTimeZones = @"TrackedTimeZones";
NSString * const kShowTimeZoneClock = @"ShowTimeZoneClock";
```

---

## 3. Event Templates

**Complexity: Medium** | **Priority: Medium** | **Estimated Effort: 1-2 weeks**

### Overview
Create reusable event templates for common meetings with pre-filled details, duration, and calendar selection.

### Technical Approach

#### Data Model

```objc
@interface EventTemplate : NSObject <NSCoding, NSSecureCoding>

@property (nonatomic, strong) NSString *templateId; // UUID
@property (nonatomic, strong) NSString *name; // e.g., "Weekly Standup"
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *location;
@property (nonatomic, strong) NSString *notes;
@property (nonatomic, strong) NSString *urlString;
@property (nonatomic) NSTimeInterval duration; // in seconds
@property (nonatomic) BOOL isAllDay;
@property (nonatomic, strong) NSString *calendarIdentifier;
@property (nonatomic, strong) NSArray<NSNumber *> *alertOffsets;
@property (nonatomic, strong) NSDate *dateCreated;
@property (nonatomic, strong) NSDate *dateModified;

- (EKEvent *)createEventForDate:(NSDate *)date 
                   eventCenter:(EventCenter *)eventCenter;

@end
```

#### Template Manager

```objc
@interface EventTemplateManager : NSObject

@property (nonatomic, strong) NSArray<EventTemplate *> *templates;

+ (instancetype)sharedManager;

- (void)addTemplate:(EventTemplate *)template;
- (void)updateTemplate:(EventTemplate *)template;
- (void)deleteTemplate:(EventTemplate *)template;
- (EventTemplate *)templateWithId:(NSString *)templateId;

@end
```

#### Storage
- Store templates as archived NSData in NSUserDefaults or separate plist file
- Key: `kEventTemplates` or file: `~/Library/Application Support/Itsycal/templates.plist`

### Implementation Steps

1. **Create EventTemplate and EventTemplateManager Classes**
   - Implement NSCoding for serialization
   - CRUD operations
   - Load/save persistence

2. **Create Template Management UI**
   - **New**: `TemplateManagementViewController`
   - Show in preferences or as separate window
   - List of templates with edit/delete/duplicate
   - "New Template" button

3. **Create Template Editor**
   - **New**: `TemplateEditorViewController`
   - Similar to EventViewController but:
     - No date/time pickers (duration only)
     - "Template Name" field
     - Save as template, not event

4. **Integrate into Event Creation Flow**
   - Add "Templates" button in EventViewController
   - Show popover with template list
   - Click template → pre-fills form
   - User can adjust before saving

5. **Quick Access**
   - Add templates menu in ViewController
   - Separate "Templates" button next to "New Event"
   - Or submenu on "New Event" button
   - Click template → opens EventViewController with pre-filled data

### Files to Modify/Create
- **New**: `Itsycal/EventTemplate.h`
- **New**: `Itsycal/EventTemplate.m`
- **New**: `Itsycal/EventTemplateManager.h`
- **New**: `Itsycal/EventTemplateManager.m`
- **New**: `Itsycal/TemplateManagementViewController.h`
- **New**: `Itsycal/TemplateManagementViewController.m`
- **New**: `Itsycal/TemplateEditorViewController.h`
- **New**: `Itsycal/TemplateEditorViewController.m`
- **Modify**: `Itsycal/EventViewController.m` (add template picker)
- **Modify**: `Itsycal/ViewController.m` (add templates button/menu)
- **Modify**: `Itsycal/PrefsVC.m` (add templates preference tab)

### User Experience Flow

1. **Creating a Template**
   - Preferences → Templates → New Template
   - Fill in template editor (like event form but no date)
   - Save with template name

2. **Using a Template**
   - Click "New Event" → Shows template picker
   - Select template → EventViewController opens pre-filled
   - Adjust date/time as needed
   - Save event

3. **Managing Templates**
   - Preferences → Templates
   - View all templates in list
   - Edit, delete, or duplicate
   - Export/import templates (future enhancement)

### NSUserDefaults Keys
```objc
NSString * const kEventTemplates = @"EventTemplates";
```

---

## 4. Smart Event Editing

**Complexity: High** | **Priority: High** | **Estimated Effort: 2-3 weeks**

### Overview
Currently can only delete events. Add the ability to edit event details directly from Itsycal, drag-and-drop to reschedule, and quick duration adjustments.

### Technical Approach

This is a significant enhancement that transforms Itsycal from a viewer to a full editor.

#### Core Capabilities

1. **Event Editing UI**
   - Reuse EventViewController for editing
   - Load existing event data into form
   - Save changes back to EventKit

2. **Drag-and-Drop Rescheduling**
   - Make events draggable in AgendaViewController
   - Drag to different date in MoCalendar
   - Show visual feedback during drag
   - Update event's start/end dates on drop

3. **Quick Duration Adjustments**
   - Resize event by dragging end time
   - +/- buttons for common increments (15, 30, 60 min)
   - Keyboard shortcuts for adjustments

4. **Context Menu Enhancements**
   - Right-click event → Edit, Duplicate, Delete
   - Quick actions: Move to tomorrow, Add 30 minutes, etc.

### Implementation Steps

#### A. Event Editing

1. **Modify EventViewController**
   - Add mode property: `.create` or `.edit`
   - Add property: `eventToEdit` (EKEvent)
   - Load event data when in edit mode:
     ```objc
     - (void)loadEventData:(EKEvent *)event;
     ```
   - Update save logic to use `updateEvent:` for existing events

2. **EventCenter Enhancements**
   ```objc
   // Add to EventCenter.h
   - (BOOL)updateEvent:(EKEvent *)event error:(NSError **)error;
   ```
   
   ```objc
   // In EventCenter.m
   - (BOOL)updateEvent:(EKEvent *)event error:(NSError **)error {
       return [_store saveEvent:event span:EKSpanThisEvent commit:YES error:error];
   }
   ```

3. **Trigger Edit from Agenda**
   - Double-click event in AgendaViewController → Opens edit
   - Or add "Edit" button in event row
   - Or context menu "Edit Event"

#### B. Drag-and-Drop Rescheduling

1. **Make AgendaViewController Support Dragging**
   - Implement `NSTableViewDelegate` drag methods:
     ```objc
     - (BOOL)tableView:writeRowsWithIndexes:toPasteboard:
     - (NSDragOperation)tableView:validateDrop:proposedRow:proposedDropOperation:
     - (BOOL)tableView:acceptDrop:row:dropOperation:
     ```
   - Create custom pasteboard type for EventInfo

2. **Make MoCalendar Accept Drops**
   - Register for drag types
   - Implement drop delegate methods
   - Highlight target date during drag
   - Calculate new start/end dates based on drop date

3. **Update Event on Drop**
   - Preserve time of day (or make all-day if dragging to date cell)
   - Preserve duration
   - Call EventCenter to save changes
   - Refresh UI

#### C. Quick Duration Adjustments

1. **Add Duration Controls to Event Detail View**
   - When showing event in popover/tooltip
   - Add +15m, +30m, +1h buttons
   - Add -15m, -30m, -1h buttons
   - Directly update event duration

2. **Visual Resize in Agenda**
   - More complex: show events with visual length
   - Allow dragging end time to resize
   - Similar to Calendar.app's list view
   - May require significant AgendaViewController refactor

#### D. Context Menu

1. **Enhance Right-Click Menu in AgendaViewController**
   ```objc
   - (NSMenu *)menuForEvent:(EKEvent *)event {
       NSMenu *menu = [[NSMenu alloc] initWithTitle:@""];
       [menu addItemWithTitle:@"Edit Event" 
                       action:@selector(editEvent:) 
                keyEquivalent:@""];
       [menu addItemWithTitle:@"Duplicate Event" 
                       action:@selector(duplicateEvent:) 
                keyEquivalent:@""];
       [menu addItem:[NSMenuItem separatorItem]];
       [menu addItemWithTitle:@"Move to Tomorrow" 
                       action:@selector(moveEventToTomorrow:) 
                keyEquivalent:@""];
       [menu addItemWithTitle:@"Add 30 Minutes" 
                       action:@selector(addThirtyMinutes:) 
                keyEquivalent:@""];
       [menu addItem:[NSMenuItem separatorItem]];
       [menu addItemWithTitle:@"Delete Event" 
                       action:@selector(deleteEvent:) 
                keyEquivalent:@""];
       return menu;
   }
   ```

### Files to Modify/Create
- **Modify**: `Itsycal/EventViewController.h` (add edit mode)
- **Modify**: `Itsycal/EventViewController.m` (implement editing)
- **Modify**: `Itsycal/EventCenter.h` (add updateEvent method)
- **Modify**: `Itsycal/EventCenter.m` (implement updateEvent)
- **Modify**: `Itsycal/AgendaViewController.h` (drag support)
- **Modify**: `Itsycal/AgendaViewController.m` (implement drag/drop)
- **Modify**: `Itsycal/MoCalendar.h` (drop support)
- **Modify**: `Itsycal/MoCalendar.m` (implement drop handling)
- **New**: `Itsycal/EventQuickActionsViewController.h`
- **New**: `Itsycal/EventQuickActionsViewController.m`

### Implementation Priority Order

1. **Phase 1**: Basic edit functionality (1 week)
   - Edit event in EventViewController
   - Double-click to edit from agenda
   - Save changes via EventCenter

2. **Phase 2**: Context menu (3-4 days)
   - Right-click menu with quick actions
   - Move to tomorrow
   - Quick duration adjustments via menu

3. **Phase 3**: Drag-and-drop (1 week)
   - Drag from agenda to calendar
   - Visual feedback
   - Date recalculation

4. **Phase 4**: Advanced features (optional)
   - Visual event length in agenda
   - Resize by dragging
   - Batch operations

### Considerations
- **Recurring Events**: Handle carefully - show alert asking "This event only" or "All future events"
- **Permissions**: Check event's calendar is writable before allowing edits
- **Conflict Detection**: Warn about overlapping events
- **Undo/Redo**: Consider implementing undo for edit operations

---

## 5. Calendar Analytics

**Complexity: Medium** | **Priority: Low-Medium** | **Estimated Effort: 2-3 weeks**

### Overview
Weekly/monthly meeting time summaries, busiest days visualization, focus time tracker, and color-coded heatmap of event density.

### Technical Approach

This feature requires significant data analysis and visualization components.

#### Analytics Engine

```objc
@interface CalendarAnalytics : NSObject

+ (instancetype)sharedAnalytics;

// Data aggregation
- (NSTimeInterval)totalMeetingTimeForDateRange:(NSDateRange *)range;
- (NSArray<DayMetrics *> *)metricsForDateRange:(NSDateRange *)range;
- (NSArray<NSDate *> *)busiestDaysInRange:(NSDateRange *)range count:(NSInteger)count;
- (NSTimeInterval)averageDailyMeetingTimeForRange:(NSDateRange *)range;
- (NSTimeInterval)focusTimeForDate:(NSDate *)date;

// Analytics queries
- (NSDictionary *)weeklyBreakdown; // Total hours per day of week
- (NSDictionary *)monthlyBreakdown; // Total hours per month
- (NSArray *)meetingsByCalendar; // Group by calendar
- (NSArray *)meetingsByType; // Recurring vs one-off

@end

@interface DayMetrics : NSObject
@property (nonatomic, strong) NSDate *date;
@property (nonatomic) NSTimeInterval totalMeetingTime;
@property (nonatomic) NSTimeInterval focusTime; // Time without meetings
@property (nonatomic) NSInteger eventCount;
@property (nonatomic) NSInteger focusBlockCount; // Continuous blocks >= 2 hours
@property (nonatomic, strong) NSArray<EKEvent *> *events;
@end

@interface NSDateRange : NSObject
@property (nonatomic, strong) NSDate *startDate;
@property (nonatomic, strong) NSDate *endDate;
@end
```

#### Visualization Components

1. **Analytics Dashboard View**
   - New window or preference pane
   - Multiple visualization tabs
   - Date range selector (week, month, quarter, year)

2. **Visualizations**

   **A. Meeting Time Summary**
   - Pie chart: Meeting time vs Focus time
   - Bar chart: Hours per day of week
   - Line chart: Trend over time
   
   **B. Busiest Days**
   - Ranked list with hours
   - Visual bars showing relative busyness
   
   **C. Focus Time Tracker**
   - Show continuous blocks of >= 2 hours
   - Highlight best focus time slots
   - Suggestions for scheduling focus time
   
   **D. Heatmap**
   - Calendar grid with color intensity
   - Darker = more meetings
   - Click date for details

3. **Chart Library**
   - Use Core Graphics for custom drawing
   - Or integrate Charts framework (macOS 11+)
   - Consider: [Charts](https://github.com/danielgindi/Charts) (iOS/macOS)

### Implementation Steps

1. **Create CalendarAnalytics Class**
   - Methods to fetch and analyze events
   - Caching for performance
   - Background processing for large date ranges

2. **Create Analytics View Controller**
   - **New**: `AnalyticsViewController`
   - Tab bar or segmented control for different views
   - Date range picker
   - Export report button (PDF, CSV)

3. **Implement Visualizations**
   - Create custom NSView subclasses for each chart type
   - **New**: `MeetingTimePieChartView`
   - **New**: `BusiestDaysBarChartView`
   - **New**: `FocusTimeLineChartView`
   - **New**: `EventDensityHeatmapView`

4. **Data Caching**
   - Cache analytics results to avoid recomputation
   - Invalidate cache when events change
   - Store in memory with TTL

5. **Access Point**
   - Add "Analytics" button in main window
   - Or menu item: "View → Calendar Analytics"
   - Or preference pane tab

### Files to Modify/Create
- **New**: `Itsycal/CalendarAnalytics.h`
- **New**: `Itsycal/CalendarAnalytics.m`
- **New**: `Itsycal/AnalyticsViewController.h`
- **New**: `Itsycal/AnalyticsViewController.m`
- **New**: `Itsycal/MeetingTimePieChartView.h`
- **New**: `Itsycal/MeetingTimePieChartView.m`
- **New**: `Itsycal/BusiestDaysBarChartView.h`
- **New**: `Itsycal/BusiestDaysBarChartView.m`
- **New**: `Itsycal/FocusTimeLineChartView.h`
- **New**: `Itsycal/FocusTimeLineChartView.m`
- **New**: `Itsycal/EventDensityHeatmapView.h`
- **New**: `Itsycal/EventDensityHeatmapView.m`
- **Modify**: `Itsycal/ViewController.m` (add analytics button)

### Chart Implementation Example - Heatmap

```objc
@interface EventDensityHeatmapView : NSView

@property (nonatomic, strong) NSDictionary<NSDate *, NSNumber *> *densityData;
@property (nonatomic, strong) NSDate *startDate;
@property (nonatomic, strong) NSDate *endDate;

- (void)reloadData;

@end

@implementation EventDensityHeatmapView

- (void)drawRect:(NSRect)dirtyRect {
    // Draw calendar grid
    // Color each cell based on event density
    // Darker color = more events
    // Use NSColor with alpha or hue variation
    
    NSInteger maxDensity = [self maxDensityValue];
    
    for (NSDate *date in self.densityData) {
        NSNumber *density = self.densityData[date];
        CGFloat intensity = density.doubleValue / maxDensity;
        
        NSColor *cellColor = [self colorForIntensity:intensity];
        [cellColor setFill];
        
        NSRect cellRect = [self rectForDate:date];
        NSBezierPath *path = [NSBezierPath bezierPathWithRect:cellRect];
        [path fill];
    }
}

- (NSColor *)colorForIntensity:(CGFloat)intensity {
    // Light blue to dark blue gradient
    return [NSColor colorWithCalibratedRed:0.2 
                                     green:0.4 + (1 - intensity) * 0.6 
                                      blue:1.0 
                                     alpha:1.0];
}

@end
```

### Performance Considerations
- Cache analytics results
- Compute in background thread
- Lazy load visualizations
- Limit default date range (e.g., 3 months)
- Add "Load More" for historical data

### Future Enhancements
- Export reports to PDF
- Email analytics summary
- Comparison mode (this month vs last month)
- Meeting attendee analytics (who meets with most)
- Location-based analytics (remote vs office)

---

## 6. Enhanced Meeting Links

**Complexity: Medium** | **Priority: High** | **Estimated Effort: 1-2 weeks**

### Overview
Currently detects Zoom URLs. Expand to support all meeting platforms, show meeting status, countdown timer, and auto-open meeting links before meetings start.

### Technical Approach

This builds on existing Zoom URL detection in `EventCenter.m` (line 450).

#### URL Pattern Detection

Current implementation:
```objc
void (^GetZoomURL)(NSString *) = ^(NSString *str) {
    if (!str) return;
    NSDataDetector *detect = [[NSDataDetector alloc] initWithTypes:NSTextCheckingTypeLink error:nil];
    [detect enumerateMatchesInString:str options:0 range:NSMakeRange(0, str.length) usingBlock:^(NSTextCheckingResult *result, NSMatchingFlags flags, BOOL *stop) {
        if ([result.URL.host containsString:@"zoom.us"] || [result.URL.host containsString:@"zoom.com"]) {
            info.zoomURL = result.URL;
            *stop = YES;
        }
    }];
};
```

**Expand to detect:**
- Zoom: `zoom.us`, `zoom.com`
- Google Meet: `meet.google.com`, `g.co/meet`
- Microsoft Teams: `teams.microsoft.com`, `teams.live.com`
- Webex: `webex.com`, `meet.webex.com`
- GoToMeeting: `gotomeet.me`, `gotomeeting.com`
- Skype: `skype.com`, `join.skype.com`
- Discord: `discord.gg`, `discord.com`
- BlueJeans: `bluejeans.com`
- Whereby: `whereby.com`
- Jitsi: `meet.jit.si`
- Generic: Any URL with "meet", "join", "call" in path

#### Meeting Platform Enum

```objc
typedef NS_ENUM(NSInteger, MeetingPlatform) {
    MeetingPlatformUnknown,
    MeetingPlatformZoom,
    MeetingPlatformGoogleMeet,
    MeetingPlatformMicrosoftTeams,
    MeetingPlatformWebex,
    MeetingPlatformGoToMeeting,
    MeetingPlatformSkype,
    MeetingPlatformDiscord,
    MeetingPlatformBlueJeans,
    MeetingPlatformWhereby,
    MeetingPlatformJitsi,
    MeetingPlatformOther
};

@interface MeetingLinkInfo : NSObject
@property (nonatomic, strong) NSURL *url;
@property (nonatomic) MeetingPlatform platform;
@property (nonatomic, strong) NSString *platformName;
@property (nonatomic, strong) NSImage *platformIcon;
@end
```

#### Meeting Status

```objc
typedef NS_ENUM(NSInteger, MeetingStatus) {
    MeetingStatusUpcoming,     // More than 5 minutes away
    MeetingStatusStartingSoon, // 2-5 minutes away
    MeetingStatusJoinable,     // Within 2 minutes or in progress
    MeetingStatusInProgress,   // Started, not ended
    MeetingStatusEnded         // Past end time
};

@interface EventInfo (MeetingStatus)
- (MeetingStatus)currentMeetingStatus;
- (NSTimeInterval)timeUntilStart;
- (NSTimeInterval)timeUntilEnd;
- (NSString *)statusDescription;
@end
```

### Implementation Steps

#### 1. Enhance URL Detection in EventCenter

```objc
// In EventCenter.m
- (void)_extractMeetingLink:(EventInfo *)info {
    // Replace current GetZoomURL with more comprehensive detection
    
    void (^GetMeetingLink)(NSString *) = ^(NSString *str) {
        if (!str) return;
        NSDataDetector *detect = [[NSDataDetector alloc] initWithTypes:NSTextCheckingTypeLink error:nil];
        [detect enumerateMatchesInString:str options:0 range:NSMakeRange(0, str.length) usingBlock:^(NSTextCheckingResult *result, NSMatchingFlags flags, BOOL *stop) {
            NSURL *url = result.URL;
            MeetingPlatform platform = [self detectMeetingPlatform:url];
            if (platform != MeetingPlatformUnknown) {
                MeetingLinkInfo *linkInfo = [MeetingLinkInfo new];
                linkInfo.url = url;
                linkInfo.platform = platform;
                linkInfo.platformName = [self platformNameForType:platform];
                linkInfo.platformIcon = [self platformIconForType:platform];
                info.meetingLink = linkInfo;
                *stop = YES;
            }
        }];
    };
    
    if (info.event.location) GetMeetingLink(info.event.location);
    if (info.meetingLink) return;
    if (info.event.URL) GetMeetingLink(info.event.URL.absoluteString);
    if (info.meetingLink) return;
    if (info.event.hasNotes && info.event.notes) GetMeetingLink(info.event.notes);
}

- (MeetingPlatform)detectMeetingPlatform:(NSURL *)url {
    NSString *host = url.host.lowercaseString;
    
    if ([host containsString:@"zoom.us"] || [host containsString:@"zoom.com"]) {
        return MeetingPlatformZoom;
    }
    if ([host containsString:@"meet.google.com"] || [host containsString:@"g.co"]) {
        return MeetingPlatformGoogleMeet;
    }
    if ([host containsString:@"teams.microsoft.com"] || [host containsString:@"teams.live.com"]) {
        return MeetingPlatformMicrosoftTeams;
    }
    if ([host containsString:@"webex.com"]) {
        return MeetingPlatformWebex;
    }
    // ... etc for other platforms
    
    // Check for generic meeting URL patterns
    NSString *path = url.path.lowercaseString;
    if ([path containsString:@"meet"] || [path containsString:@"join"] || [path containsString:@"call"]) {
        return MeetingPlatformOther;
    }
    
    return MeetingPlatformUnknown;
}
```

#### 2. Meeting Status Calculation

```objc
// In EventInfo category
- (MeetingStatus)currentMeetingStatus {
    NSDate *now = [NSDate date];
    NSTimeInterval timeUntilStart = [self.event.startDate timeIntervalSinceDate:now];
    NSTimeInterval timeUntilEnd = [self.event.endDate timeIntervalSinceDate:now];
    
    if (timeUntilEnd < 0) {
        return MeetingStatusEnded;
    }
    if (timeUntilStart < 0) {
        return MeetingStatusInProgress;
    }
    if (timeUntilStart < 120) { // 2 minutes
        return MeetingStatusJoinable;
    }
    if (timeUntilStart < 300) { // 5 minutes
        return MeetingStatusStartingSoon;
    }
    return MeetingStatusUpcoming;
}

- (NSString *)statusDescription {
    MeetingStatus status = [self currentMeetingStatus];
    switch (status) {
        case MeetingStatusJoinable:
            return @"Join now";
        case MeetingStatusStartingSoon:
            return [NSString stringWithFormat:@"Starts in %@", [self formattedTimeUntilStart]];
        case MeetingStatusInProgress:
            return @"In progress";
        case MeetingStatusEnded:
            return @"Ended";
        default:
            return @"";
    }
}
```

#### 3. Update AgendaViewController

```objc
// Show meeting platform icon next to event
// Update meeting status in real-time
// Show countdown for upcoming meetings
// Make join button more prominent for joinable meetings

- (NSView *)tableView:(NSTableView *)tableView viewForTableColumn:(NSTableColumn *)tableColumn row:(NSInteger)row {
    // Existing implementation...
    
    if (eventInfo.meetingLink) {
        // Add platform icon
        NSImageView *platformIcon = [[NSImageView alloc] initWithFrame:NSMakeRect(0, 0, 16, 16)];
        platformIcon.image = eventInfo.meetingLink.platformIcon;
        [cellView addSubview:platformIcon];
        
        // Add status label
        NSTextField *statusLabel = [NSTextField labelWithString:[eventInfo statusDescription]];
        [cellView addSubview:statusLabel];
        
        // Update button text based on status
        MeetingStatus status = [eventInfo currentMeetingStatus];
        if (status == MeetingStatusJoinable || status == MeetingStatusInProgress) {
            button.title = [NSString stringWithFormat:@"Join %@", eventInfo.meetingLink.platformName];
            button.highlighted = YES; // Make prominent
        }
    }
}
```

#### 4. Meeting Countdown Timer

```objc
// In ViewController or new MeetingMonitorController

@interface MeetingMonitorController : NSObject
@property (nonatomic, weak) EventCenter *eventCenter;
@property (nonatomic, strong) NSTimer *monitorTimer;
- (void)startMonitoring;
- (void)stopMonitoring;
@end

@implementation MeetingMonitorController

- (void)startMonitoring {
    // Check every 30 seconds for upcoming meetings
    self.monitorTimer = [NSTimer scheduledTimerWithTimeInterval:30.0
                                                         target:self
                                                       selector:@selector(checkUpcomingMeetings)
                                                       userInfo:nil
                                                        repeats:YES];
    [self checkUpcomingMeetings];
}

- (void)checkUpcomingMeetings {
    NSDate *now = [NSDate date];
    NSDate *soon = [now dateByAddingTimeInterval:300]; // Next 5 minutes
    
    // Get events in next 5 minutes
    NSArray *upcomingEvents = [self eventsInRange:now to:soon];
    
    for (EventInfo *info in upcomingEvents) {
        MeetingStatus status = [info currentMeetingStatus];
        
        if (status == MeetingStatusStartingSoon) {
            // Show notification
            [self showMeetingNotification:info];
        }
        
        if (status == MeetingStatusJoinable) {
            // Auto-open meeting link if enabled
            if ([[NSUserDefaults standardUserDefaults] boolForKey:kAutoOpenMeetingLinks]) {
                [self autoOpenMeetingLink:info];
            }
        }
    }
}

- (void)showMeetingNotification:(EventInfo *)info {
    NSUserNotification *notification = [[NSUserNotification alloc] init];
    notification.title = info.event.title;
    notification.subtitle = [NSString stringWithFormat:@"Starts in %@", [info formattedTimeUntilStart]];
    notification.informativeText = @"Click to join";
    notification.userInfo = @{@"eventId": info.event.eventIdentifier, @"meetingURL": info.meetingLink.url.absoluteString};
    notification.soundName = NSUserNotificationDefaultSoundName;
    
    [[NSUserNotificationCenter defaultCenter] deliverNotification:notification];
}

- (void)autoOpenMeetingLink:(EventInfo *)info {
    NSTimeInterval timeUntilStart = [info timeUntilStart];
    NSTimeInterval autoOpenWindow = [[NSUserDefaults standardUserDefaults] doubleForKey:kAutoOpenMeetingWindow];
    
    if (timeUntilStart < autoOpenWindow && timeUntilStart > -60) { // Within window, not more than 1 min late
        [[NSWorkspace sharedWorkspace] openURL:info.meetingLink.url];
    }
}

@end
```

#### 5. Menu Bar Meeting Indicator

```objc
// In ViewController.m - enhance existing meeting indicator

- (void)updateMeetingIndicator {
    // Current implementation shows generic meeting indicator
    // Enhance to show:
    // - Time until next meeting
    // - Platform icon
    // - Click to join directly
    
    EventInfo *nextMeeting = [self nextMeetingWithLink];
    if (nextMeeting) {
        NSTimeInterval timeUntil = [nextMeeting timeUntilStart];
        
        if (timeUntil < 300 && timeUntil > -60) { // Within 5 minutes
            // Show countdown in menu bar
            NSString *countdown = [self formatCountdown:timeUntil];
            _statusItem.button.title = countdown;
            
            // Add action to join meeting
            _statusItem.button.action = @selector(joinNextMeeting:);
        }
    }
}

- (void)joinNextMeeting:(id)sender {
    EventInfo *nextMeeting = [self nextMeetingWithLink];
    if (nextMeeting && nextMeeting.meetingLink) {
        [[NSWorkspace sharedWorkspace] openURL:nextMeeting.meetingLink.url];
    }
}
```

### Files to Modify/Create
- **Modify**: `Itsycal/EventCenter.h` (add MeetingLinkInfo)
- **Modify**: `Itsycal/EventCenter.m` (enhance URL detection)
- **New**: `Itsycal/MeetingLinkInfo.h`
- **New**: `Itsycal/MeetingLinkInfo.m`
- **New**: `Itsycal/MeetingMonitorController.h`
- **New**: `Itsycal/MeetingMonitorController.m`
- **Modify**: `Itsycal/AgendaViewController.m` (show status, icons)
- **Modify**: `Itsycal/ViewController.m` (countdown timer)
- **Modify**: `Itsycal/Itsycal.h` (add constants)

### Platform Icons
- Include small icons for each platform (16x16 @ 2x)
- Use SF Symbols where available (macOS 11+)
- Fallback to custom icons for older macOS versions

### Preferences
Add new preferences:
```objc
NSString * const kAutoOpenMeetingLinks = @"AutoOpenMeetingLinks";
NSString * const kAutoOpenMeetingWindow = @"AutoOpenMeetingWindow"; // Seconds before start (default: 120)
NSString * const kShowMeetingCountdown = @"ShowMeetingCountdown";
NSString * const kShowMeetingNotifications = @"ShowMeetingNotifications";
```

### User Experience Enhancements

1. **Visual Indicators**
   - Different colors for different meeting platforms
   - Pulsing/animated indicator for meetings starting soon
   - Badge on menu bar icon showing number of upcoming meetings today

2. **Smart Join Button**
   - "Join Zoom" vs "Join Teams" vs "Join Meeting"
   - Disabled state if meeting hasn't started (for some platforms)
   - Opens directly in app if installed, else browser

3. **Notification Actions**
   - "Join Now" button in notification
   - "Snooze" to remind again in 5 minutes
   - Deep link to event details

4. **Preferences**
   - Toggle auto-open per platform
   - Customize auto-open timing
   - Select preferred app (desktop vs browser)

---

## Implementation Roadmap

### Phase 1: Core Enhancements (6-8 weeks)
1. **Smart Event Editing** (Weeks 1-3)
   - Highest impact on user workflow
   - Unlocks full calendar management capabilities
   
2. **Enhanced Meeting Links** (Weeks 4-5)
   - Builds on existing functionality
   - High user value for remote work

3. **Event Templates** (Weeks 6-7)
   - Improves efficiency for recurring workflows

### Phase 2: Advanced Features (6-8 weeks)
4. **Natural Language Event Creation** (Weeks 8-10)
   - Complex but high user delight
   - Differentiates from competitors

5. **Time Zone Support** (Weeks 11-13)
   - Essential for distributed teams
   - Moderate complexity

### Phase 3: Analytics (3-4 weeks)
6. **Calendar Analytics** (Weeks 14-17)
   - Nice-to-have feature
   - Lower priority but high engagement potential

---

## Technical Considerations

### Backward Compatibility
- Maintain support for macOS 10.13+ (or project's minimum version)
- Use `@available` checks for newer APIs
- Graceful degradation for missing features

### Localization
- All new strings must be localized
- Update all `.lproj` folders
- Consider RTL language support

### Performance
- Lazy loading for analytics
- Background processing for heavy operations
- Caching strategies for event data

### Testing
- Unit tests for parsing logic (natural language, URL detection)
- Integration tests for EventKit operations
- Manual testing across different macOS versions
- Test with various calendar providers (iCloud, Google, Exchange)

### Privacy & Permissions
- Calendar access already granted for read
- Write access needed for editing features
- No additional permissions needed for other features

### App Store Considerations
- All APIs used are public and allowed
- No private APIs
- Sandboxing compatible

---

## Dependencies & Resources

### External Libraries (Optional)
- **Charts/Graphs**: Core Plot, Charts framework, or custom drawing
- **Natural Language**: NSLinguisticTagger (built-in) or Chronic-style parser

### Design Assets Needed
- Meeting platform icons (Zoom, Teams, Meet, etc.)
- Chart/graph visual designs
- New button icons for templates, analytics

### Documentation
- User guide updates for new features
- Developer documentation for new classes
- API documentation for public interfaces

---

## Risk Assessment

### High Risk
- **Drag-and-drop**: Complex interaction, potential for bugs
- **Natural language parsing**: Difficult to handle all cases correctly

### Medium Risk
- **Time zones**: Edge cases with DST, date line, all-day events
- **Analytics performance**: Large date ranges could be slow

### Low Risk
- **Event templates**: Straightforward CRUD operations
- **Meeting links**: Extension of existing functionality
- **Event editing**: Well-documented EventKit APIs

---

## Conclusion

These six features would significantly enhance Itsycal's capabilities, transforming it from a calendar viewer into a comprehensive calendar management tool. The implementation plan prioritizes user impact, with core editing and meeting features first, followed by productivity enhancements like templates and natural language, and finally analytics for insights.

**Recommended Implementation Order:**
1. Smart Event Editing (unlocks full editing capability)
2. Enhanced Meeting Links (builds on existing, high remote work value)
3. Event Templates (improves efficiency)
4. Natural Language Event Creation (differentiator)
5. Time Zone Support (essential for distributed teams)
6. Calendar Analytics (engagement and insights)

**Total Estimated Effort**: 18-23 weeks (4.5-6 months) for full implementation

Each feature can be developed independently, allowing for incremental releases and user feedback integration throughout the development process.
