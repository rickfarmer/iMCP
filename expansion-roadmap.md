# iMCP Expansion Roadmap

## Executive Summary

This document outlines a comprehensive expansion plan for iMCP, proposing 12 new service integrations to enhance macOS automation capabilities. Each service has been evaluated for implementation complexity, required permissions, and potential impact.

**Current State:** 11 active services (Calendar, Contacts, Location, Maps, Messages, Reminders, Weather, Capture, Operator, Files, Utilities)

**Proposed Additions:** 12 new services across system management, productivity, and developer tools

---

## Priority Matrix

### High Priority (Immediate Impact, Reasonable Complexity)
1. **Mail Service** - Critical productivity tool, AppleScript ready
2. **Notifications Service** - Simple API, high utility for automation
3. **Clipboard Service** - Frequent use case, straightforward implementation
4. **Safari Service** - Browser automation, file-based data access

### Medium Priority (High Value, Moderate Complexity)
5. **Audio Service** - System control utility
6. **Network Service** - Developer and IT use cases
7. **Spotlight Service** - Powerful search capability
8. **Power Service** - Battery monitoring, energy management

### Lower Priority (Complex or Niche Use Cases)
9. **Finder Service** - Advanced file operations (overlaps with Files service)
10. **System Preferences Service** - Security and compatibility concerns
11. **Display Service** - Private APIs, compatibility issues
12. **Time Machine Service** - Limited API access, private verbs

---

## Detailed Service Specifications

## 1. Mail Service

**Priority:** HIGH
**Complexity:** Medium
**Implementation:** AppleScript + MailKit

### Use Cases
- Read emails from specific mailboxes/accounts
- Search emails by sender, subject, date range, or content
- Send emails programmatically with attachments
- Manage mailboxes (archive, delete, mark as read/unread)
- Access email metadata (flags, labels, headers)
- Filter by read/unread status, starred, flagged

### Implementation Approach
**Primary Method:** AppleScript bridge via `NSAppleScript`
- Mail.app has comprehensive AppleScript dictionary
- Full support for reading, searching, and composing emails
- Can access messages, mailboxes, accounts, and attachments

**Alternative Method:** MailKit framework (macOS 14+)
- Native Swift API for Mail extensions
- May require Mail extension architecture
- More restricted but officially supported

**Code Structure:**
```swift
final class MailService: Service {
    var tools: [Tool] {
        // mail_fetch: Search and retrieve emails
        // mail_send: Compose and send emails
        // mail_search: Full-text search across mailboxes
        // mail_manage: Archive, delete, mark operations
    }
}
```

### Required Permissions
- `com.apple.security.automation.apple-events` (already in entitlements)
- AppleScript targeting approval for Mail.app
- User consent dialog on first use

### Technical Considerations
- Mail.app must be running for AppleScript to work
- Large mailboxes may require pagination
- Attachment handling needs temporary file management
- Modern macOS requires explicit permission for each target app

### Tools Specification

#### `mail_fetch`
```json
{
  "name": "mail_fetch",
  "description": "Fetch emails from Mail.app",
  "parameters": {
    "mailbox": "string (optional) - Mailbox name",
    "account": "string (optional) - Account name",
    "sender": "string (optional) - Sender email/name",
    "subject": "string (optional) - Subject search term",
    "start": "datetime (optional) - Start date",
    "end": "datetime (optional) - End date",
    "read": "boolean (optional) - Filter by read status",
    "flagged": "boolean (optional) - Filter by flag status",
    "limit": "integer (default: 50) - Max results"
  }
}
```

#### `mail_send`
```json
{
  "name": "mail_send",
  "description": "Send an email via Mail.app",
  "parameters": {
    "to": "array[string] - Recipient addresses",
    "cc": "array[string] (optional) - CC addresses",
    "bcc": "array[string] (optional) - BCC addresses",
    "subject": "string - Email subject",
    "body": "string - Email body (HTML or plain text)",
    "attachments": "array[string] (optional) - File paths"
  }
}
```

#### `mail_search`
```json
{
  "name": "mail_search",
  "description": "Full-text search across Mail.app",
  "parameters": {
    "query": "string - Search query",
    "mailbox": "string (optional) - Limit to mailbox",
    "limit": "integer (default: 50)"
  }
}
```

### References
- [How to Use AppleScript with Mail.app for Email Automation - GeeksforGeeks](https://www.geeksforgeeks.org/techtips/how-to-use-applescript-with-mail-app-for-email-automation/)
- [Use scripts as rule actions in Mail on Mac - Apple Support](https://support.apple.com/guide/mail/use-scripts-as-rule-actions-mlhlp1171/mac)
- [Automate tasks in Mail on Mac - Apple Support](https://support.apple.com/guide/mail/automate-mail-tasks-mlhlp1120/mac)

---

## 2. Safari Service

**Priority:** HIGH
**Complexity:** Low-Medium
**Implementation:** File-based (plist) + AppleScript

### Use Cases
- Access bookmarks hierarchy
- Read Reading List items
- Get open tabs from all windows
- Search browsing history
- Export bookmark collections
- Add items to Reading List

### Implementation Approach
**Primary Method:** Direct plist parsing
- Bookmarks stored in `~/Library/Safari/Bookmarks.plist`
- Reading List is part of bookmarks plist (identifier: "com.apple.ReadingList")
- History in `~/Library/Safari/History.db` (SQLite)

**Secondary Method:** AppleScript for limited operations
- `add reading list item` command available
- `show bookmarks` command
- Tab access via AppleScript

**Code Structure:**
```swift
final class SafariService: Service {
    private let bookmarksPath = "~/Library/Safari/Bookmarks.plist"
    private let historyPath = "~/Library/Safari/History.db"

    var tools: [Tool] {
        // safari_bookmarks: Read bookmark hierarchy
        // safari_reading_list: Access Reading List
        // safari_tabs: Get currently open tabs
        // safari_history: Search browsing history
        // safari_add_to_reading_list: Add URL to Reading List
    }
}
```

### Required Permissions
- `com.apple.security.files.user-selected.read-write` (already in entitlements)
- May need user to grant access via file picker for first use
- Full Disk Access for unrestricted Safari folder access

### Technical Considerations
- Bookmarks.plist uses binary plist format (PropertyListSerialization)
- Reading List items have additional metadata (preview text, date added)
- History.db is SQLite3, similar to Messages implementation
- Safari must be closed when writing to these files (read-only is safer)
- macOS 10.14+ has privacy restrictions for ~/Library/Safari/

### Tools Specification

#### `safari_bookmarks`
```json
{
  "name": "safari_bookmarks",
  "description": "Retrieve Safari bookmarks",
  "parameters": {
    "folder": "string (optional) - Bookmark folder name",
    "search": "string (optional) - Search term"
  }
}
```

#### `safari_reading_list`
```json
{
  "name": "safari_reading_list",
  "description": "Access Safari Reading List items",
  "parameters": {
    "unread": "boolean (optional) - Filter by unread status",
    "limit": "integer (default: 50)"
  }
}
```

#### `safari_tabs`
```json
{
  "name": "safari_tabs",
  "description": "Get currently open Safari tabs",
  "parameters": {
    "window": "integer (optional) - Specific window index"
  }
}
```

#### `safari_history`
```json
{
  "name": "safari_history",
  "description": "Search Safari browsing history",
  "parameters": {
    "query": "string (optional) - Search term",
    "start": "datetime (optional) - Start date",
    "end": "datetime (optional) - End date",
    "limit": "integer (default: 100)"
  }
}
```

### References
- [How to get Safari Bookmarks to Object in macOS - Stack Overflow](https://stackoverflow.com/questions/57722287/how-to-get-safari-bookmarks-to-object-in-macos-mojave-with-applescript)
- [Exporting Links from Safari Reading List via Shortcuts for Mac - MacStories](https://www.macstories.net/mac/exporting-links-from-safari-reading-list-via-shortcuts-for-mac/)
- [AppleScript access to Safari Reading List - Apple Community](https://discussions.apple.com/thread/7053341)

---

## 3. Notifications Service

**Priority:** HIGH
**Complexity:** Low
**Implementation:** UNUserNotificationCenter (macOS 10.14+)

### Use Cases
- Create system notifications with custom titles, bodies, and actions
- Schedule notifications for future delivery
- Set notification priority and interruption level
- Add action buttons to notifications
- Handle notification responses
- Clear delivered notifications

### Implementation Approach
**Primary Method:** UNUserNotificationCenter (modern API)
```swift
import UserNotifications

final class NotificationsService: NSObject, Service, UNUserNotificationCenterDelegate {
    private let center = UNUserNotificationCenter.current()

    func activate() async throws {
        try await center.requestAuthorization(options: [.alert, .sound, .badge])
        center.delegate = self
    }
}
```

**Note:** NSUserNotification is deprecated; use UNUserNotificationCenter exclusively

### Required Permissions
- No special entitlements needed
- User consent dialog on first use
- Notifications can be managed in System Settings > Notifications

### Technical Considerations
- App must be registered in System Settings > Notifications
- Notification delivery depends on Focus mode and DND settings
- Actions require UNNotificationAction setup
- Responses are delivered via delegate methods
- Scheduled notifications persist across app restarts

### Tools Specification

#### `notifications_send`
```json
{
  "name": "notifications_send",
  "description": "Send a system notification",
  "parameters": {
    "title": "string - Notification title",
    "body": "string - Notification body",
    "subtitle": "string (optional) - Notification subtitle",
    "sound": "string (optional) - Sound name (default, critical, or none)",
    "badge": "integer (optional) - Badge count",
    "identifier": "string (optional) - Unique identifier",
    "threadIdentifier": "string (optional) - Thread/group identifier"
  }
}
```

#### `notifications_schedule`
```json
{
  "name": "notifications_schedule",
  "description": "Schedule a notification for future delivery",
  "parameters": {
    "title": "string - Notification title",
    "body": "string - Notification body",
    "trigger": "datetime - When to deliver",
    "repeats": "boolean (default: false) - Repeat notification"
  }
}
```

#### `notifications_clear`
```json
{
  "name": "notifications_clear",
  "description": "Clear delivered notifications",
  "parameters": {
    "identifiers": "array[string] (optional) - Specific IDs to clear, or all if omitted"
  }
}
```

### References
- [What's the Difference Between NSUserNotification and UNUserNotification - Stack Overflow](https://stackoverflow.com/questions/64272748/whats-the-difference-between-nsusernotification-and-unusernotification-in-macos)
- [UNUserNotificationCenter - Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)

---

## 4. Clipboard Service

**Priority:** HIGH
**Complexity:** Low
**Implementation:** NSPasteboard

### Use Cases
- Read current clipboard content (text, images, files, URLs)
- Write content to clipboard
- Monitor clipboard changes for automation
- Access multiple pasteboard types (general, find, drag)
- Support for rich content types (RTF, HTML, attributed strings)
- Clipboard history (requires continuous monitoring)

### Implementation Approach
**Primary Method:** NSPasteboard API with polling
```swift
import AppKit

final class ClipboardService: Service {
    private var lastChangeCount: Int = 0
    private var monitoringTimer: Timer?

    func startMonitoring() {
        monitoringTimer = Timer.scheduledTimer(withTimeInterval: 0.5, repeats: true) { _ in
            let currentCount = NSPasteboard.general.changeCount
            if currentCount != self.lastChangeCount {
                self.lastChangeCount = currentCount
                // Clipboard changed
            }
        }
    }
}
```

**Key Insight:** No notification API exists for clipboard changes - polling via `changeCount` is the official approach

### Required Permissions
- No special entitlements needed
- Reading clipboard requires no permission
- Writing is unrestricted

### Technical Considerations
- Polling every 0.5s is recommended (changeCount comparison is fast)
- Multiple pasteboard types: `.general`, `.find`, `.drag`, `.ruler`, `.font`
- Support for multiple data types per item (e.g., both HTML and plain text)
- Large clipboard content (images, files) should be handled carefully
- Clipboard history requires persistent storage and memory management

### Tools Specification

#### `clipboard_read`
```json
{
  "name": "clipboard_read",
  "description": "Read current clipboard content",
  "parameters": {
    "type": "string (default: auto) - Content type: text, image, file, url, or auto"
  },
  "returns": {
    "text": "string (if text)",
    "image": "base64 data (if image)",
    "files": "array[string] - file paths (if files)",
    "url": "string (if URL)"
  }
}
```

#### `clipboard_write`
```json
{
  "name": "clipboard_write",
  "description": "Write content to clipboard",
  "parameters": {
    "content": "string - Content to write",
    "type": "string (default: text) - Content type: text, html, rtf"
  }
}
```

#### `clipboard_monitor`
```json
{
  "name": "clipboard_monitor",
  "description": "Start/stop clipboard monitoring",
  "parameters": {
    "enabled": "boolean - Enable or disable monitoring",
    "includeHistory": "boolean (default: false) - Store clipboard history"
  }
}
```

### References
- [Observe NSPasteboard - Swift 4 - Medium](https://fedevitale.medium.com/watch-for-nspasteboard-fad29d2f874e)
- [Can I receive a callback whenever an NSPasteboard is written to? - Stack Overflow](https://stackoverflow.com/questions/5033266/can-i-receive-a-callback-whenever-an-nspasteboard-is-written-to)
- [NSPasteboard - Apple Developer Documentation](https://developer.apple.com/documentation/appkit/nspasteboard)

---

## 5. Audio Service

**Priority:** MEDIUM
**Complexity:** Medium-High
**Implementation:** Core Audio APIs

### Use Cases
- Get/set system volume
- List available audio input/output devices
- Switch default audio device
- Monitor volume changes
- Mute/unmute audio
- Get device properties (name, manufacturer, sample rate)

### Implementation Approach
**Primary Method:** Core Audio AudioUnit APIs
- `kAudioOutputUnitProperty_CurrentDevice` for device selection
- Core Audio properties for volume control
- Device enumeration via AudioObjectGetPropertyData

**Note:** AVAudioSession is iOS-only; macOS requires lower-level Core Audio

```swift
import CoreAudio
import AVFoundation

final class AudioService: Service {
    func getDefaultOutputDevice() -> AudioDeviceID {
        var deviceID = AudioDeviceID()
        var propertySize = UInt32(MemoryLayout<AudioDeviceID>.size)
        var propertyAddress = AudioObjectPropertyAddress(
            mSelector: kAudioHardwarePropertyDefaultOutputDevice,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )
        AudioObjectGetPropertyData(AudioObjectID(kAudioObjectSystemObject),
                                  &propertyAddress, 0, nil, &propertySize, &deviceID)
        return deviceID
    }
}
```

### Required Permissions
- No special entitlements needed
- System controls are accessible without permission

### Technical Considerations
- System volume is read-only from app perspective (user controls only)
- App-specific volume is controllable
- Device switching requires Core Audio unit property changes
- Aggregate devices combine multiple inputs/outputs
- Sample rate changes may require device restart
- Some operations require stopping audio playback

### Tools Specification

#### `audio_devices_list`
```json
{
  "name": "audio_devices_list",
  "description": "List all audio input/output devices",
  "parameters": {
    "type": "string (optional) - Filter by: input, output, or all (default)"
  }
}
```

#### `audio_device_get`
```json
{
  "name": "audio_device_get",
  "description": "Get current default audio device",
  "parameters": {
    "type": "string - Device type: input or output"
  }
}
```

#### `audio_device_set`
```json
{
  "name": "audio_device_set",
  "description": "Set default audio device",
  "parameters": {
    "deviceId": "integer - Audio device ID",
    "type": "string - Device type: input or output"
  }
}
```

#### `audio_volume_get`
```json
{
  "name": "audio_volume_get",
  "description": "Get current system or app volume",
  "parameters": {
    "scope": "string (default: system) - system or app"
  }
}
```

### References
- [How to set AVAudioEngine input and output devices - Stack Overflow](https://stackoverflow.com/questions/61827898/how-to-set-avaudioengine-input-and-output-devices-swift-macos)
- [AudioKit on macOS: get/set system device volume - Stack Overflow](https://stackoverflow.com/questions/51618968/audiokit-on-macos-get-set-system-device-volume)

---

## 6. Network Service

**Priority:** MEDIUM
**Complexity:** Medium
**Implementation:** SystemConfiguration + NetworkExtension

### Use Cases
- Get current network status (connected, disconnected, connecting)
- List available WiFi networks
- Get current WiFi network name (SSID)
- Network diagnostics (ping, traceroute, DNS lookup)
- VPN status and control
- Network interface information (IP, MAC, subnet)

### Implementation Approach
**Primary Method:** SystemConfiguration framework
```swift
import SystemConfiguration
import NetworkExtension

final class NetworkService: Service {
    func getCurrentWiFiSSID() -> String? {
        guard let interfaces = CNCopySupportedInterfaces() as? [String] else { return nil }
        for interface in interfaces {
            if let info = CNCopyCurrentNetworkInfo(interface as CFString) as? [String: Any],
               let ssid = info[kCNNetworkInfoKeySSID as String] as? String {
                return ssid
            }
        }
        return nil
    }
}
```

**For VPN:** NetworkExtension framework
**For Diagnostics:** Process spawning (ping, traceroute, dig commands)

### Required Permissions
- `com.apple.security.network.client` (already in entitlements)
- Location permission required for WiFi SSID access (privacy restriction)
- VPN control requires NEVPNManager authorization

### Technical Considerations
- WiFi SSID access requires location permission (iOS/macOS consistency)
- VPN management needs user approval per VPN configuration
- Network diagnostics via CLI tools are most reliable
- Real-time monitoring requires SCNetworkReachability callbacks
- WiFi network scanning requires private APIs or CLI tools

### Tools Specification

#### `network_status`
```json
{
  "name": "network_status",
  "description": "Get current network connectivity status",
  "parameters": {}
}
```

#### `network_wifi_current`
```json
{
  "name": "network_wifi_current",
  "description": "Get current WiFi network information",
  "parameters": {}
}
```

#### `network_interfaces`
```json
{
  "name": "network_interfaces",
  "description": "List all network interfaces with details",
  "parameters": {}
}
```

#### `network_diagnostic`
```json
{
  "name": "network_diagnostic",
  "description": "Run network diagnostic tool",
  "parameters": {
    "tool": "string - Tool to run: ping, traceroute, or dns",
    "target": "string - Hostname or IP address",
    "count": "integer (default: 4) - Number of packets/hops"
  }
}
```

---

## 7. Spotlight Service

**Priority:** MEDIUM
**Complexity:** Low-Medium
**Implementation:** NSMetadataQuery

### Use Cases
- Search files across entire system using Spotlight
- Filter by file type, date, content, metadata
- Search with natural language queries
- Get file metadata (creator, modified date, tags, etc.)
- Real-time search results as user types
- Search within specific scopes (home directory, documents, etc.)

### Implementation Approach
**Primary Method:** NSMetadataQuery
```swift
import Foundation

final class SpotlightService: Service {
    func search(query: String, scope: [String] = []) -> [NSMetadataItem] {
        let metadataQuery = NSMetadataQuery()
        metadataQuery.predicate = NSPredicate(fromMetadataQueryString: query)
        if !scope.isEmpty {
            metadataQuery.searchScopes = scope
        }
        metadataQuery.start()
        // Handle results via notifications
        return []
    }
}
```

**Alternative:** `mdfind` CLI tool for simpler implementation

### Required Permissions
- Full Disk Access recommended for comprehensive results
- Some system areas restricted without FDA

### Technical Considerations
- NSMetadataQuery is asynchronous (results via notifications)
- Large result sets should be paginated
- Spotlight indexing may not be complete immediately after file changes
- Query syntax is powerful but complex (Spotlight predicates)
- `mdfind` CLI provides simpler interface but less control

### Tools Specification

#### `spotlight_search`
```json
{
  "name": "spotlight_search",
  "description": "Search files using Spotlight",
  "parameters": {
    "query": "string - Spotlight search query",
    "scope": "string (optional) - Search scope path",
    "kind": "string (optional) - File kind filter (document, image, etc.)",
    "limit": "integer (default: 100)"
  }
}
```

#### `spotlight_metadata`
```json
{
  "name": "spotlight_metadata",
  "description": "Get Spotlight metadata for a file",
  "parameters": {
    "path": "string - File path"
  }
}
```

---

## 8. Power Service

**Priority:** MEDIUM
**Complexity:** Low
**Implementation:** IOKit

### Use Cases
- Get battery level and charging status
- Estimate remaining battery time
- Get power source information (AC, battery, UPS)
- Monitor low power mode status
- Get energy impact of running processes
- Schedule sleep/wake times

### Implementation Approach
**Primary Method:** IOKit IOPowerSources API
```swift
import IOKit.ps

final class PowerService: Service {
    func getBatteryInfo() -> [String: Any]? {
        let snapshot = IOPSCopyPowerSourcesInfo().takeRetainedValue()
        let sources = IOPSCopyPowerSourcesList(snapshot).takeRetainedValue() as? [CFTypeRef]

        if let source = sources?.first {
            return IOPSGetPowerSourceDescription(snapshot, source).takeUnretainedValue() as? [String: Any]
        }
        return nil
    }
}
```

### Required Permissions
- No special entitlements needed
- Public IOKit APIs

### Technical Considerations
- Desktop Macs may not have battery
- Multiple power sources possible (battery + UPS)
- Time remaining estimates may be inaccurate
- Low power mode detection requires checking system preferences
- Process energy impact requires Activity Monitor APIs (private)

### Tools Specification

#### `power_battery_status`
```json
{
  "name": "power_battery_status",
  "description": "Get current battery status",
  "parameters": {}
}
```

#### `power_source_info`
```json
{
  "name": "power_source_info",
  "description": "Get information about all power sources",
  "parameters": {}
}
```

---

## 9. Finder Service

**Priority:** LOW-MEDIUM
**Complexity:** Medium
**Implementation:** AppleScript + NSWorkspace

### Use Cases
- Get Finder selection
- Navigate to specific locations
- Reveal files in Finder
- Get/set file labels and tags
- Move files to trash
- Duplicate files
- Get Quick Look preview

### Implementation Approach
**Primary Method:** AppleScript for Finder-specific operations
**Secondary:** NSWorkspace for file system operations

**Note:** Significant overlap with existing Files service - consider carefully

### Required Permissions
- Same as Files service (Full Disk Access)

### Technical Considerations
- Files service already provides file operations
- Finder service would add GUI-specific features
- Tags and labels are file system metadata (xattrs)
- Quick Look can be shown via QLPreviewPanel

---

## 10. System Preferences Service

**Priority:** LOW
**Complexity:** High
**Implementation:** Private APIs / defaults command

### Use Cases
- Read system settings
- Modify user preferences
- Get system information (OS version, hardware specs)
- Open specific preference panes

### Implementation Approach
**Warning:** Most system settings are in private domains or require root access

**Primary Method:** `defaults` command for user defaults
**Alternative:** Private framework calls (not recommended)

### Required Permissions
- Varies by setting
- Many require admin privileges
- SIP restrictions apply

### Technical Considerations
- **Security Concern:** Modifying system settings is risky
- Many settings require admin password
- Changes may not take effect until relaunch
- Different macOS versions have different preference structures
- Consider read-only access initially

---

## 11. Display Service

**Priority:** LOW
**Complexity:** High
**Implementation:** Private CoreDisplay APIs

### Use Cases
- Get/set display brightness
- List connected displays
- Change display resolution
- Manage multiple display arrangement
- Night Shift control

### Implementation Approach
**Primary Method:** Private DisplayServices framework (required for Apple Silicon)
**Legacy:** IOKit (deprecated, doesn't work on M1+)

```swift
// Requires private DisplayServices.framework
extern int DisplayServicesGetBrightness(CGDirectDisplayID display, float *brightness)
extern void DisplayServicesSetBrightness(CGDirectDisplayID display, float brightness)
```

### Required Permissions
- None technically, but uses private APIs

### Technical Considerations
- **Compatibility Issues:** Different methods needed for Intel vs Apple Silicon
- IOKit brightness APIs deprecated in 10.9, broken in 10.12.4+
- External displays require DDC protocol (different than built-in)
- Night Shift introduced breaking changes
- Using private APIs risks App Store rejection
- MonitorControl open-source project demonstrates working implementation

### References
- [Adjust screen brightness in Mac OS X app - Stack Overflow](https://stackoverflow.com/questions/32691397/adjust-screen-brightness-in-mac-os-x-app)
- [MonitorControl/MonitorControl GitHub](https://github.com/MonitorControl/MonitorControl)
- [Reverse Engineering CoreDisplay API - Alex DeLorenzo](https://alexdelorenzo.dev/programming/2018/08/16/reverse_engineering_private_apple_apis)

---

## 12. Time Machine Service

**Priority:** LOW
**Complexity:** Medium-High
**Implementation:** tmutil command-line tool

### Use Cases
- Check if backup is running
- Get last backup date
- Get backup size and destination
- List available backups
- Check backup health
- Trigger manual backup

### Implementation Approach
**Primary Method:** Shell execution of `tmutil` commands
```swift
final class TimeMachineService: Service {
    func getStatus() async throws -> [String: Any] {
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/tmutil")
        process.arguments = ["status"]
        // Parse plist output
    }
}
```

**Note:** `tmutil status` is marked as "private verb" but still functional

### Required Permissions
- No special entitlements needed
- Some operations may require admin privileges

### Technical Considerations
- No official public API exists
- tmutil is a private tool (not guaranteed stable)
- Output format is plist (can convert to JSON)
- Some verbs require root access
- Backup operations are slow (long-running tasks)
- Read-only operations are safer and more reliable

### Tools Specification

#### `timemachine_status`
```json
{
  "name": "timemachine_status",
  "description": "Get Time Machine backup status",
  "parameters": {}
}
```

#### `timemachine_latest_backup`
```json
{
  "name": "timemachine_latest_backup",
  "description": "Get information about latest backup",
  "parameters": {}
}
```

### References
- [How do I know if a time machine backup is running? - Ask Different](https://apple.stackexchange.com/questions/316418/how-do-i-know-if-a-time-machine-backup-is-running)
- [Time Machine progress from the command line - Ask Different](https://apple.stackexchange.com/questions/162464/time-machine-progress-from-the-command-line)

---

## Implementation Roadmap

### Phase 1: Quick Wins (1-2 weeks)
**Target:** High-value, low-complexity services
1. **Notifications Service** (2-3 days)
   - Simple API, immediate utility
   - Foundation for automation workflows
2. **Clipboard Service** (2-3 days)
   - Straightforward NSPasteboard implementation
   - High demand from users
3. **Power Service** (2-3 days)
   - IOKit API well-documented
   - Useful for laptop users

### Phase 2: Core Productivity (2-3 weeks)
**Target:** Essential productivity integrations
1. **Mail Service** (5-7 days)
   - Complex but high-impact
   - AppleScript implementation requires testing
   - Permission handling needs careful UX
2. **Safari Service** (4-5 days)
   - File-based access, similar to Messages
   - Bookmark parsing straightforward
   - History database similar to existing patterns
3. **Audio Service** (3-4 days)
   - Core Audio requires more research
   - Device management is complex
   - Start with read-only operations

### Phase 3: System Integration (2-3 weeks)
**Target:** System-level capabilities
1. **Network Service** (4-5 days)
   - Multiple APIs to coordinate
   - WiFi requires location permission
   - Diagnostics via CLI tools
2. **Spotlight Service** (3-4 days)
   - NSMetadataQuery is asynchronous
   - Result handling needs optimization
   - Consider mdfind fallback

### Phase 4: Advanced Features (As Needed)
**Target:** Specialized or complex services
1. **Finder Service** (3-4 days if needed)
   - Evaluate overlap with Files service first
   - May be redundant
2. **Time Machine Service** (3-4 days)
   - Shell wrapper around tmutil
   - Read-only to start
3. **Display Service** (5-7 days)
   - Private API research required
   - Compatibility testing across hardware
   - May defer indefinitely due to complexity
4. **System Preferences Service** (Not recommended)
   - Security and stability concerns
   - Defer until strong use case emerges

---

## Implementation Complexity Assessment

### Low Complexity (< 1 week each)
- **Notifications Service**: Well-documented API, straightforward implementation
- **Clipboard Service**: Simple NSPasteboard, polling pattern established
- **Power Service**: Clean IOKit API, good examples available
- **Spotlight Service**: NSMetadataQuery or mdfind, both proven approaches

### Medium Complexity (1-2 weeks each)
- **Safari Service**: File parsing similar to Messages, multiple data sources
- **Mail Service**: AppleScript bridge requires careful error handling
- **Network Service**: Multiple frameworks, various permission requirements
- **Audio Service**: Core Audio APIs less familiar, device management complex
- **Time Machine Service**: CLI wrapper, plist parsing, limited API

### High Complexity (2+ weeks each)
- **Finder Service**: Overlaps with Files, unclear value proposition
- **Display Service**: Private APIs, hardware-specific code, compatibility issues
- **System Preferences Service**: Security concerns, private domains, root access

---

## Most Impactful Additions (Priority Order)

### Tier 1: Must-Have (Build First)
1. **Mail Service**
   - **Impact:** Critical for productivity workflows
   - **Use Cases:** Email automation, search, management
   - **Justification:** Email is central to professional workflows; currently missing from iMCP

2. **Safari Service**
   - **Impact:** Browser data access unlocks automation
   - **Use Cases:** Bookmark management, Reading List, research workflows
   - **Justification:** Web browser integration is essential for modern workflows

3. **Notifications Service**
   - **Impact:** User feedback and automation alerts
   - **Use Cases:** Task reminders, event alerts, workflow completion
   - **Justification:** Enables iMCP to communicate proactively with users

4. **Clipboard Service**
   - **Impact:** Core copy-paste workflows
   - **Use Cases:** Text processing, clipboard history, automation
   - **Justification:** Frequently requested, enables text manipulation automation

### Tier 2: High Value (Build Soon)
5. **Audio Service**
   - **Impact:** System control for workflows
   - **Use Cases:** Device switching, volume automation
   - **Justification:** Useful for presentation and recording workflows

6. **Network Service**
   - **Impact:** Network-aware automation
   - **Use Cases:** VPN management, WiFi switching, diagnostics
   - **Justification:** Important for developers and IT professionals

7. **Spotlight Service**
   - **Impact:** System-wide search capability
   - **Use Cases:** File discovery, content search
   - **Justification:** Leverages existing macOS search infrastructure

8. **Power Service**
   - **Impact:** Battery awareness
   - **Use Cases:** Energy monitoring, automation triggers
   - **Justification:** Useful for laptop users, automation conditions

### Tier 3: Specialized (Build Later)
9. **Finder Service**
   - **Impact:** GUI file operations
   - **Consideration:** Overlap with Files service may make this redundant

10. **Time Machine Service**
    - **Impact:** Backup monitoring
    - **Consideration:** Limited API, niche use case

11. **Display Service**
    - **Impact:** Display management
    - **Consideration:** High complexity, private APIs, compatibility issues

12. **System Preferences Service**
    - **Impact:** System configuration
    - **Consideration:** Security concerns, recommend deferring

---

## Technical Architecture Recommendations

### Service Template Pattern
All new services should follow the established pattern:

```swift
import Foundation
import OSLog

private let log = Logger.service("service-name")

final class ServiceNameService: Service {
    static let shared = ServiceNameService()

    // MARK: - Activation

    var isActivated: Bool {
        get async {
            // Check permission status
            return true
        }
    }

    func activate() async throws {
        // Request permissions if needed
    }

    // MARK: - Tools

    var tools: [Tool] {
        Tool(
            name: "service_operation",
            description: "Description of operation",
            inputSchema: .object(
                properties: [:],
                additionalProperties: false
            ),
            annotations: .init(
                title: "Operation Title",
                readOnlyHint: true,
                openWorldHint: false
            )
        ) { arguments in
            // Implementation
        }
    }
}
```

### Permission Handling
- Request permissions in `activate()` method
- Check status in `isActivated` computed property
- Show user-friendly dialogs explaining why permission is needed
- Store permission state in UserDefaults if needed
- Handle permission denial gracefully

### Error Handling
- Use OSLog for debugging
- Return structured errors via NSError
- Include helpful error messages for users
- Log permission issues at .error level
- Log successful operations at .debug level

### JSON-LD Output Format
Continue using Schema.org vocabularies for tool results:
```json
{
  "@context": "https://schema.org",
  "@type": "Type",
  "property": "value"
}
```

---

## Entitlements Requirements

### Already Available
- `com.apple.security.app-sandbox` - Required for App Store
- `com.apple.security.automation.apple-events` - For AppleScript (Mail, Finder)
- `com.apple.security.files.user-selected.read-write` - File access (Safari, Files)
- `com.apple.security.network.client` - Network access (Network service)

### May Need Adding
- None of the proposed services require additional entitlements
- Full Disk Access (FDA) is system-level, not an entitlement
- Most services use public APIs or user-selected file access

---

## Risk Assessment

### Low Risk Services
- **Notifications, Clipboard, Power, Spotlight**: All use public, stable APIs
- Implementation straightforward
- No security or privacy concerns beyond standard permissions

### Medium Risk Services
- **Mail, Safari, Network, Audio**: Use public APIs but require careful permission handling
- AppleScript bridges can be fragile across macOS versions
- File-based access requires security-scoped bookmarks

### High Risk Services
- **Display**: Uses private APIs, compatibility issues
- **System Preferences**: Security implications
- **Time Machine**: Relies on undocumented CLI tool
- Recommendation: Defer high-risk services until demand justifies complexity

---

## Success Metrics

### User Impact
- **Mail Service**: Target 70%+ of users benefit (email is universal)
- **Safari Service**: Target 60%+ of users benefit (browser automation)
- **Notifications**: Target 50%+ of users benefit (feedback mechanism)
- **Clipboard**: Target 40%+ of users benefit (text workflows)

### Technical Goals
- No crashes related to new services
- Permission requests < 3 per service
- Tool execution time < 2 seconds for 90% of operations
- < 5% error rate in tool execution

### Adoption Metrics
- Services enabled by 50%+ of users within 1 month
- Average tools per user increases by 30%
- User retention improves by 20%

---

## Conclusion

This expansion roadmap proposes 12 new services for iMCP, prioritized by impact and implementation complexity. The recommended build order focuses on high-value, achievable services first:

**Immediate Build (Phase 1-2):**
1. Mail Service
2. Safari Service
3. Notifications Service
4. Clipboard Service

**Near-Term (Phase 3):**
5. Audio Service
6. Network Service
7. Spotlight Service
8. Power Service

**Future Consideration:**
9. Finder Service (evaluate overlap with Files)
10. Time Machine Service (limited API)
11. Display Service (private APIs)
12. System Preferences Service (defer indefinitely)

This approach balances user value with technical feasibility, leveraging iMCP's existing architecture patterns while avoiding high-risk implementations that rely on private APIs or unstable CLI tools.

---

## Appendix: Reference Links

### Mail Service
- [How to Use AppleScript with Mail.app for Email Automation - GeeksforGeeks](https://www.geeksforgeeks.org/techtips/how-to-use-applescript-with-mail-app-for-email-automation/)
- [Use scripts as rule actions in Mail on Mac - Apple Support](https://support.apple.com/guide/mail/use-scripts-as-rule-actions-mlhlp1171/mac)
- [Automate tasks in Mail on Mac - Apple Support](https://support.apple.com/guide/mail/automate-mail-tasks-mlhlp1120/mac)

### Safari Service
- [How to get Safari Bookmarks to Object in macOS - Stack Overflow](https://stackoverflow.com/questions/57722287/how-to-get-safari-bookmarks-to-object-in-macos-mojave-with-applescript)
- [Exporting Links from Safari Reading List via Shortcuts for Mac - MacStories](https://www.macstories.net/mac/exporting-links-from-safari-reading-list-via-shortcuts-for-mac/)

### Notifications Service
- [What's the Difference Between NSUserNotification and UNUserNotification - Stack Overflow](https://stackoverflow.com/questions/64272748/whats-the-difference-between-nsusernotification-and-unusernotification-in-macos)
- [UNUserNotificationCenter - Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)

### Clipboard Service
- [Observe NSPasteboard - Swift 4 - Medium](https://fedevitale.medium.com/watch-for-nspasteboard-fad29d2f874e)
- [Can I receive a callback whenever an NSPasteboard is written to? - Stack Overflow](https://stackoverflow.com/questions/5033266/can-i-receive-a-callback-whenever-an-nspasteboard-is-written-to)

### Audio Service
- [How to set AVAudioEngine input and output devices - Stack Overflow](https://stackoverflow.com/questions/61827898/how-to-set-avaudioengine-input-and-output-devices-swift-macos)
- [AudioKit on macOS: get/set system device volume - Stack Overflow](https://stackoverflow.com/questions/51618968/audiokit-on-macos-get-set-system-device-volume)

### Display Service
- [Adjust screen brightness in Mac OS X app - Stack Overflow](https://stackoverflow.com/questions/32691397/adjust-screen-brightness-in-mac-os-x-app)
- [MonitorControl/MonitorControl GitHub](https://github.com/MonitorControl/MonitorControl)

### Time Machine Service
- [How do I know if a time machine backup is running? - Ask Different](https://apple.stackexchange.com/questions/316418/how-do-i-know-if-a-time-machine-backup-is-running)
- [Time Machine progress from the command line - Ask Different](https://apple.stackexchange.com/questions/162464/time-machine-progress-from-the-command-line)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-30
**Author:** Research and design for iMCP expansion
