# Selection Assistant Feature - Technical Implementation Documentation

## Table of Contents

- [1. Overview](#1-overview)
- [2. Underlying Principles](#2-underlying-principles)
- [3. Technical Architecture](#3-technical-architecture)
- [4. Core Components](#4-core-components)
- [5. Implementation Details](#5-implementation-details)
- [6. Technology Stack](#6-technology-stack)
- [7. Data Flow](#7-data-flow)
- [8. Implementation Approach Comparison](#8-implementation-approach-comparison)
- [9. Advantages and Disadvantages](#9-advantages-and-disadvantages)
- [10. Platform Compatibility](#10-platform-compatibility)

---

## 1. Overview

The Selection Assistant is one of Cherry Studio's core features, enabling users to quickly perform translations, searches, AI queries, and other operations via a floating toolbar after selecting text in any application. This feature uses system-level text selection listening to provide a seamless cross-application text interaction experience.

### 1.1 Main Features

- **Global Text Selection Monitoring**: Listens for OS-level text selection events
- **Floating Toolbar**: Displays action buttons near selected text
- **Quick Actions**: Predefined operations like copy, search, translate, AI chat
- **Custom Actions**: Support for user-defined actions and AI assistants
- **Multiple Trigger Modes**: Show on selection, Ctrl-key trigger, keyboard shortcut
- **Application Filtering**: Whitelist/blacklist mode to filter specific applications

### 1.2 Key Characteristics

- **Cross-Application Support**: Works with almost all applications (browsers, editors, PDF readers, etc.)
- **Smart Positioning**: Toolbar intelligently positions based on selection location and screen boundaries
- **Non-Intrusive**: Does not affect normal application usage
- **High Performance**: Preloaded window mechanism ensures fast response
- **Platform Adaptation**: Special optimizations for Windows and macOS

---

## 2. Underlying Principles

### 2.1 Text Selection Detection Mechanism

The core of Selection Assistant is detecting OS-level text selection events. The implementation is based on the following technologies:

#### 2.1.1 System Hooks

Uses the `selection-hook` native module to implement low-level text selection listening:

```typescript
// Import from selection-hook module
import type {
  SelectionHookConstructor,
  SelectionHookInstance,
  TextSelectionData
} from 'selection-hook'

// Initialize the hook
this.selectionHook = new SelectionHook()
```

**Working Principles:**

1. **Windows Platform**:
   - Uses Windows API global hooks
   - Monitors WM_COPY and WM_PASTE messages
   - Detects text selection via clipboard and cursor position
   - Uses SetWindowsHookEx to register low-level keyboard and mouse hooks

2. **macOS Platform**:
   - Uses Accessibility API
   - Requires accessibility permission grant
   - Monitors NSPasteboard changes
   - Uses CGEventTap to listen for mouse and keyboard events

#### 2.1.2 Clipboard Monitoring

```typescript
// Key steps for detecting text selection:
// 1. Monitor mouse down and up events
// 2. Detect Ctrl+C (copy) operation
// 3. Read clipboard content
// 4. Get cursor position and selection rectangle
```

#### 2.1.3 Position Information Acquisition

```typescript
interface TextSelectionData {
  text: string                    // Selected text
  programName: string             // Program name
  posLevel: PositionLevel         // Position precision level
  mousePosStart: { x: number; y: number }  // Mouse start position
  mousePosEnd: { x: number; y: number }    // Mouse end position
  startTop: { x: number; y: number }       // Selection rect top-left
  startBottom: { x: number; y: number }    // Selection rect bottom-left
  endTop: { x: number; y: number }         // Selection rect top-right
  endBottom: { x: number; y: number }      // Selection rect bottom-right
  isFullscreen?: boolean          // Fullscreen mode (macOS)
}
```

### 2.2 Window Management Mechanism

#### 2.2.1 Electron BrowserWindow Features

Uses Electron's BrowserWindow API to create special window types:

**Toolbar Window Properties:**
```typescript
{
  frame: false,              // Frameless
  transparent: true,         // Transparent background
  alwaysOnTop: true,        // Always on top
  skipTaskbar: true,        // Not shown in taskbar
  focusable: false,         // Not focusable (Windows)
  type: 'toolbar',          // Windows: toolbar type
  type: 'panel',            // macOS: panel type
  hiddenInMissionControl: true  // macOS: hidden in Mission Control
}
```

**Action Window Properties:**
```typescript
{
  frame: false,              // Frameless
  transparent: true,         // Transparent background
  hasShadow: false,         // No shadow
  titleBarStyle: 'hidden',  // Hidden title bar (macOS)
  sandbox: true             // Sandbox enabled
}
```

#### 2.2.2 Window Hierarchy Management

```typescript
// Set toolbar to highest level
this.toolbarWindow.setAlwaysOnTop(true, 'screen-saver')

// macOS fullscreen app compatibility
this.toolbarWindow.setVisibleOnAllWorkspaces(true, {
  visibleOnFullScreen: true,
  skipTransformProcessType: true
})
```

### 2.3 IPC Communication Mechanism

Main Process and Renderer Process communicate via IPC channels:

```typescript
// Main process registers handlers
ipcMain.handle(IpcChannel.Selection_ProcessAction, (_, actionItem) => {
  selectionService?.processAction(actionItem)
})

// Renderer process calls
window.api.selection.processAction(actionItem)

// Main process sends events
this.toolbarWindow.webContents.send(
  IpcChannel.Selection_TextSelected, 
  selectionData
)

// Renderer process listens
window.electron.ipcRenderer.on(
  IpcChannel.Selection_TextSelected,
  (_, selectionData) => {
    // Handle selection data
  }
)
```

---

## 3. Technical Architecture

### 3.1 Overall Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Operating System                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Native Selection Hook (C++)                 │  │
│  │  - Windows: SetWindowsHookEx, WM_COPY               │  │
│  │  - macOS: Accessibility API, CGEventTap             │  │
│  └──────────────────┬───────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      │ Node Native Addon
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              Electron Main Process (Node.js)                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            SelectionService (Singleton)              │  │
│  │  - Text selection event handling                     │  │
│  │  - Window lifecycle management                       │  │
│  │  - Position calculation                              │  │
│  │  - Filter mode processing                            │  │
│  │  - IPC handler registration                          │  │
│  └──────────────────┬───────────────────────────────────┘  │
│                     │                                         │
│  ┌──────────────────┴───────────────────────────────────┐  │
│  │              ConfigManager                            │  │
│  │  - Settings persistence                               │  │
│  │  - Config subscriptions                               │  │
│  └──────────────────┬───────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      │ IPC Communication
                      ↓
┌─────────────────────────────────────────────────────────────┐
│            Electron Renderer Process (Chromium)              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Toolbar Window (React)                       │  │
│  │  - SelectionToolbar component                        │  │
│  │  - Action buttons rendering                          │  │
│  │  - Animation effects                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │      Action Window (React)                           │  │
│  │  - SelectionActionApp component                      │  │
│  │  - ActionTranslate / ActionGeneral                   │  │
│  │  - AI interaction                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Redux Store (State Management)                │  │
│  │  - selectionStore: settings and actions              │  │
│  │  - Sync with ConfigManager                           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Architecture Layers

#### 3.2.1 Native Layer

- **selection-hook module**: Node.js native addon implemented in C++
- Responsibilities: System-level event listening, clipboard operations, window info retrieval

#### 3.2.2 Service Layer

- **SelectionService**: Core service class, singleton pattern
- **ConfigManager**: Configuration management, persistent storage
- **ShortcutService**: Keyboard shortcut integration
- Responsibilities: Business logic, state management, window lifecycle

#### 3.2.3 UI Layer

- **SelectionToolbar**: Floating toolbar UI
- **SelectionActionApp**: Action window UI
- **Redux Store**: State management
- Responsibilities: User interaction, data display, animations

### 3.3 Design Patterns

#### 3.3.1 Singleton Pattern

```typescript
export class SelectionService {
  private static instance: SelectionService | null = null
  
  public static getInstance(): SelectionService | null {
    if (!SelectionService.instance) {
      SelectionService.instance = new SelectionService()
    }
    return SelectionService.instance
  }
  
  private constructor() {
    // Private constructor ensures singleton
  }
}
```

**Benefits**:
- Only one global instance, avoiding resource conflicts
- Centralized management of all selection events and windows

#### 3.3.2 Observer Pattern

```typescript
// Subscribe to config changes
configManager.subscribe(ConfigKeys.SelectionAssistantTriggerMode, 
  (triggerMode: TriggerMode) => {
    this.triggerMode = triggerMode
    this.processTriggerMode()
  }
)

// Listen to selection events
this.selectionHook.on('text-selection', this.processTextSelection)
```

#### 3.3.3 Factory Pattern

```typescript
// Preloaded window pool
private createPreloadedActionWindow(): BrowserWindow {
  // Create configured window instance
}

private popActionWindow(): BrowserWindow {
  // Get from pool or create new window
  const actionWindow = this.preloadedActionWindows.pop() 
    || this.createPreloadedActionWindow()
  return actionWindow
}
```

#### 3.3.4 Strategy Pattern

```typescript
enum TriggerMode {
  Selected = 'selected',    // Show on selection
  Ctrlkey = 'ctrlkey',     // Ctrl key trigger
  Shortcut = 'shortcut'     // Keyboard shortcut
}

private processTriggerMode(): void {
  switch (this.triggerMode) {
    case TriggerMode.Selected:
      this.selectionHook.setSelectionPassiveMode(false)
      break
    case TriggerMode.Ctrlkey:
      this.selectionHook.on('key-down', this.handleKeyDownCtrlkeyMode)
      this.selectionHook.setSelectionPassiveMode(true)
      break
    case TriggerMode.Shortcut:
      this.selectionHook.setSelectionPassiveMode(true)
      break
  }
}
```

---

## 4. Core Components

### 4.1 SelectionService

**Location**: `src/main/services/SelectionService.ts`

**Responsibilities**:
- Initialize and manage selection-hook instance
- Handle text selection events
- Manage lifecycle of toolbar and action windows
- Handle logic for different trigger modes
- Implement application filtering
- Handle IPC communication

**Key Methods**:

```typescript
class SelectionService {
  // Start service
  public start(): boolean
  
  // Stop service
  public stop(): boolean
  
  // Process text selection
  private processTextSelection(selectionData: TextSelectionData): void
  
  // Show toolbar
  private showToolbarAtPosition(point: Point, orientation: string): void
  
  // Calculate toolbar position
  private calculateToolbarPosition(refPoint: Point, orientation: string): Point
  
  // Process action
  public processAction(actionItem: ActionItem, isFullScreen: boolean): void
  
  // Create action window
  private createPreloadedActionWindow(): BrowserWindow
}
```

**State Management**:

```typescript
private initStatus: boolean = false      // Initialization status
private started: boolean = false         // Running status
private triggerMode = TriggerMode.Selected  // Trigger mode
private isFollowToolbar = true           // Action window follows toolbar
private filterMode = 'default'           // Filter mode
private filterList: string[] = []        // Filter list
private toolbarWindow: BrowserWindow | null = null     // Toolbar window
private actionWindows = new Set<BrowserWindow>()       // Action windows set
private preloadedActionWindows: BrowserWindow[] = []   // Preloaded window pool
```

### 4.2 SelectionToolbar

**Location**: `src/renderer/src/windows/selection/toolbar/SelectionToolbar.tsx`

**Responsibilities**:
- Render floating toolbar UI
- Display action buttons
- Handle user click events
- Implement animations

**Component Structure**:

```tsx
<Container>
  <LogoWrapper>
    <Logo />  {/* App icon, draggable */}
  </LogoWrapper>
  <ActionWrapper>
    <ActionIcons>
      <ActionButton>  {/* Copy button */}
        <ActionIcon />
        <ActionTitle />
      </ActionButton>
      <ActionButton>  {/* Search button */}
        ...
      </ActionButton>
      {/* More action buttons */}
    </ActionIcons>
  </ActionWrapper>
</Container>
```

**Features**:
- Uses styled-components for styling
- CSS variables support custom styles
- Animations: rotate, scale, fade in/out
- Responsive layout (compact/standard mode)

### 4.3 SelectionActionApp

**Location**: `src/renderer/src/windows/selection/action/SelectionActionApp.tsx`

**Responsibilities**:
- Render action window UI
- Execute specific actions (translate, AI chat, etc.)
- Handle window controls (minimize, close, pin, opacity)
- Implement auto-scroll

**Window Control Features**:

```tsx
- Pin: Keep window on top
- Minimize: Minimize to taskbar
- Close: Close window
- Opacity: Adjust window opacity 20%-100%
- Auto Close: Auto-close when losing focus
- Auto Pin: Auto-pin when opening
```

**Content Components**:
- **ActionTranslate**: Translation feature
- **ActionGeneral**: General AI chat feature

### 4.4 Configuration Management

**Location**: `src/main/services/ConfigManager.ts`

**Related Config Keys**:

```typescript
ConfigKeys.SelectionAssistantEnabled        // Enabled status
ConfigKeys.SelectionAssistantTriggerMode    // Trigger mode
ConfigKeys.SelectionAssistantFollowToolbar  // Follow toolbar
ConfigKeys.SelectionAssistantRemeberWinSize // Remember window size
ConfigKeys.SelectionAssistantFilterMode     // Filter mode
ConfigKeys.SelectionAssistantFilterList     // Filter list
```

**Predefined Blacklist**:

**Windows**:
```typescript
[
  'explorer.exe',          // File Explorer
  'excel.exe',            // Excel
  'powerpnt.exe',         // PowerPoint
  'photoshop.exe',        // Photoshop
  'snipaste.exe',         // Screenshot tool
  // ... more
]
```

**macOS**:
```typescript
[
  'com.apple.finder'      // Finder
]
```

### 4.5 Redux Store

**Location**: `src/renderer/src/store/selectionStore`

**State Definition**:

```typescript
interface SelectionState {
  selectionEnabled: boolean      // Enabled status
  triggerMode: TriggerMode       // Trigger mode
  isCompact: boolean            // Compact mode
  isAutoClose: boolean          // Auto close
  isAutoPin: boolean            // Auto pin
  isFollowToolbar: boolean      // Follow toolbar
  isRemeberWinSize: boolean     // Remember window size
  filterMode: FilterMode        // Filter mode
  filterList: string[]          // Filter list
  actionWindowOpacity: number   // Window opacity
  actionItems: ActionItem[]     // Action items
}
```

---

## 5. Implementation Details

### 5.1 Text Selection Processing Flow

```
1. User selects text in any application
   ↓
2. selection-hook detects selection event
   ↓
3. Get selected text and position info
   ↓
4. Trigger 'text-selection' event
   ↓
5. SelectionService.processTextSelection()
   ↓
6. Check application filter
   ↓
7. Calculate toolbar position
   ↓
8. Show toolbar window
   ↓
9. Send IPC event to toolbar window
   ↓
10. SelectionToolbar receives data and displays
```

### 5.2 Position Calculation Algorithm

Toolbar position calculation considers the following factors:

**Input**:
- `refPoint`: Reference point coordinates (mouse position or selection rect)
- `orientation`: Desired direction (topLeft, topRight, bottomMiddle, etc.)

**Processing Steps**:

```typescript
// 1. Calculate initial position based on orientation
switch (orientation) {
  case 'topLeft':
    posPoint.x = refPoint.x - toolbarWidth
    posPoint.y = refPoint.y - toolbarHeight
    break
  case 'bottomMiddle':
    posPoint.x = refPoint.x - toolbarWidth / 2
    posPoint.y = refPoint.y
    break
  // ... other orientations
}

// 2. Get display info
const display = screen.getDisplayNearestPoint(refPoint)

// 3. Boundary check and adjustment
posPoint.x = Math.max(
  display.workArea.x, 
  Math.min(posPoint.x, display.workArea.x + display.workArea.width - toolbarWidth)
)
posPoint.y = Math.max(
  display.workArea.y,
  Math.min(posPoint.y, display.workArea.y + display.workArea.height - toolbarHeight)
)

// 4. Special case adjustments (exceeds screen top/bottom)
if (exceedsTop) posPoint.y += 32
if (exceedsBottom) posPoint.y -= 32
```

**Position Precision Levels**:

```typescript
enum PositionLevel {
  NONE,           // No position info, use cursor position
  MOUSE_SINGLE,   // Single mouse click
  MOUSE_DUAL,     // Mouse drag selection
  SEL_FULL,       // Full selection rect
  SEL_DETAILED    // Detailed selection rect (multi-line)
}
```

### 5.3 Trigger Mode Implementation

#### 5.3.1 Selected Mode

```typescript
// Auto trigger, show toolbar immediately on text selection
this.selectionHook.setSelectionPassiveMode(false)
this.selectionHook.on('text-selection', this.processTextSelection)
```

#### 5.3.2 Ctrl Key Mode

```typescript
// Trigger after holding Ctrl key for 350ms
this.selectionHook.on('key-down', this.handleKeyDownCtrlkeyMode)
this.selectionHook.on('key-up', this.handleKeyUpCtrlkeyMode)
this.selectionHook.setSelectionPassiveMode(true)

private handleKeyDownCtrlkeyMode = (data: KeyboardEventData) => {
  if (this.lastCtrlkeyDownTime === 0) {
    this.lastCtrlkeyDownTime = Date.now()
    return
  }
  
  if (Date.now() - this.lastCtrlkeyDownTime >= 350) {
    const selectionData = this.selectionHook!.getCurrentSelection()
    if (selectionData) {
      this.processTextSelection(selectionData)
    }
  }
}
```

#### 5.3.3 Shortcut Mode

```typescript
// Trigger via global keyboard shortcut
this.selectionHook.setSelectionPassiveMode(true)

// Called by ShortcutService
public processSelectTextByShortcut(): void {
  const selectionData = this.selectionHook.getCurrentSelection()
  if (selectionData) {
    this.processTextSelection(selectionData)
  }
}
```

### 5.4 Window Preloading Mechanism

To improve response speed, implements a window preload pool:

```typescript
// Preload count
private readonly PRELOAD_ACTION_WINDOW_COUNT = 1

// Create preloaded windows during initialization
private async initPreloadedActionWindows(): Promise<void> {
  for (let i = 0; i < this.PRELOAD_ACTION_WINDOW_COUNT; i++) {
    await this.pushNewActionWindow()
  }
}

// Pop window from pool when needed
private popActionWindow(): BrowserWindow {
  const actionWindow = this.preloadedActionWindows.pop() 
    || this.createPreloadedActionWindow()
  
  // Asynchronously create new window to replenish pool
  this.pushNewActionWindow()
  
  return actionWindow
}
```

**Benefits**:
- Windows are already loaded and immediately available
- Avoids loading delay when displaying
- Improves user experience

### 5.5 Platform-Specific Handling

#### 5.5.1 Windows Special Handling

```typescript
// Window type
type: 'toolbar'

// Not focusable
focusable: false

// Coordinate conversion (physical ↔ logical)
const mousePoint = screen.screenToDipPoint({ x: data.x, y: data.y })
```

#### 5.5.2 macOS Special Handling

```typescript
// Requires accessibility permission
systemPreferences.isTrustedAccessibilityClient(false)

// Window type
type: 'panel'

// Fullscreen app compatibility
this.toolbarWindow.setVisibleOnAllWorkspaces(true, {
  visibleOnFullScreen: true,
  skipTransformProcessType: true
})

// Show inactive (avoid bringing other windows to front)
this.toolbarWindow.showInactive()

// Focus handling when hiding (avoid bringing other windows to front)
const focusableWindows: BrowserWindow[] = []
for (const window of BrowserWindow.getAllWindows()) {
  if (window.isFocusable()) {
    focusableWindows.push(window)
    window.setFocusable(false)
  }
}
this.toolbarWindow.hide()
setTimeout(() => {
  focusableWindows.forEach(window => window.setFocusable(true))
}, 50)
```

### 5.6 Application Filtering Implementation

```typescript
enum FilterMode {
  DEFAULT = 'default',      // Default mode (use predefined blacklist)
  WHITELIST = 'whitelist',  // Whitelist mode (only enable in listed apps)
  BLACKLIST = 'blacklist'   // Blacklist mode (exclude listed apps)
}

private setHookGlobalFilterMode(mode: string, list: string[]): void {
  const predefinedBlacklist = isWin 
    ? SELECTION_PREDEFINED_BLACKLIST.WINDOWS 
    : SELECTION_PREDEFINED_BLACKLIST.MAC
  
  let combinedList: string[] = list
  let combinedMode = mode
  
  // Only merge predefined blacklist in selected mode
  if (this.triggerMode === TriggerMode.Selected) {
    switch (mode) {
      case 'blacklist':
        // Merge user blacklist with predefined blacklist
        combinedList = [...new Set([...list, ...predefinedBlacklist])]
        break
      case 'whitelist':
        combinedList = [...list]
        break
      case 'default':
        // Use predefined blacklist
        combinedList = [...predefinedBlacklist]
        combinedMode = 'blacklist'
        break
    }
  }
  
  this.selectionHook.setGlobalFilterMode(modeMap[combinedMode], combinedList)
}
```

### 5.7 Fine-Tuned Lists

Behavior optimization for specific applications:

```typescript
// Exclude cursor detection for apps (avoid false triggers)
EXCLUDE_CLIPBOARD_CURSOR_DETECT: {
  WINDOWS: ['acrobat.exe', 'wps.exe', 'cajviewer.exe'],
  MAC: []
}

// Delay clipboard read for apps (wait for app to complete copy)
INCLUDE_CLIPBOARD_DELAY_READ: {
  WINDOWS: ['acrobat.exe', 'wps.exe', 'cajviewer.exe', 'foxitphantom.exe'],
  MAC: []
}
```

---

## 6. Technology Stack

### 6.1 Core Technologies

| Technology | Version | Purpose |
|-----------|---------|---------|
| Electron | Latest | Desktop app framework |
| React | 18.x | UI framework |
| TypeScript | 5.x | Type system |
| Redux Toolkit | Latest | State management |
| styled-components | 6.x | CSS-in-JS |
| Ant Design | 5.x | UI component library |
| selection-hook | 1.0.12 | Native selection listening |

### 6.2 Build Tools

- **electron-vite**: Electron app building
- **Vite**: Fast dev server and build tool
- **rolldown-vite**: Experimental bundler

### 6.3 Development Tools

- **Biome**: Code formatting and linting
- **Vitest**: Unit testing framework
- **TypeScript**: Type checking

### 6.4 Dependencies

```
cherry-studio
├── electron (main framework)
├── selection-hook (core dependency, native module)
│   └── Provides system-level text selection listening
├── React (UI layer)
│   ├── SelectionToolbar component
│   └── SelectionActionApp component
├── Redux (state management)
│   └── selectionStore
└── Ant Design (UI components)
    ├── Button, Switch, Radio
    ├── Slider, Tooltip
    └── ...
```

---

## 7. Data Flow

### 7.1 Initialization Flow

```
1. app.ready event triggers
   ↓
2. initSelectionService()
   ↓
3. SelectionService.getInstance()
   ↓
4. new SelectionHook() (load native module)
   ↓
5. Check OS support and permissions
   ↓
6. SelectionService.start()
   ↓
7. Create toolbar window (hidden)
   ↓
8. Initialize preloaded action window pool
   ↓
9. Register event listeners
   ↓
10. Start selection-hook
```

### 7.2 Selection to Display Flow

```
[User Action]
User selects text
   ↓
[Native Layer]
selection-hook detects selection
   ↓
Get text content and position
   ↓
[Service Layer]
Trigger 'text-selection' event
   ↓
SelectionService.processTextSelection()
   ↓
Check application filter
   ↓
Calculate toolbar position
   ↓
Show toolbar window
   ↓
[IPC]
Send Selection_TextSelected event
   ↓
[UI Layer]
SelectionToolbar receives event
   ↓
Update state and UI
   ↓
Display action buttons
```

### 7.3 Action Execution Flow

```
[User Action]
Click toolbar button
   ↓
[UI Layer]
handleAction(actionItem)
   ↓
Handle based on action type:
├─ copy: Copy to clipboard
├─ search: Open search engine
├─ quote: Quote to main window
└─ other: Open action window
   ↓
[IPC]
window.api.selection.processAction()
   ↓
[Service Layer]
SelectionService.processAction()
   ↓
Get window from preload pool
   ↓
Send action data to window
   ↓
Show action window
   ↓
[UI Layer]
SelectionActionApp receives data
   ↓
Render based on action type:
├─ translate: ActionTranslate
└─ other: ActionGeneral
   ↓
Execute AI call or other processing
   ↓
Display result
```

### 7.4 Configuration Sync Flow

```
[UI Layer - Settings]
User modifies settings
   ↓
dispatch(setSelectionEnabled(value))
   ↓
window.api.selection.setEnabled(value)
   ↓
[IPC]
Selection_SetEnabled
   ↓
[Service Layer]
configManager.setSelectionAssistantEnabled(value)
   ↓
Save to config file
   ↓
Trigger subscription callback
   ↓
SelectionService receives notification
   ↓
Update service state
   ↓
[Sync]
storeSyncService.syncToRenderer()
   ↓
[Redux]
Sync to all renderer processes
```

---

## 8. Implementation Approach Comparison

### 8.1 Text Selection Detection Approaches

#### Approach A: Native Hook (Current)

**Implementation**:
- Node.js native addon written in C++
- Directly calls OS APIs

**Pros**:
- ✅ Best performance
- ✅ Precise position information
- ✅ Supports any application
- ✅ Fast response

**Cons**:
- ❌ High development and maintenance cost
- ❌ Need to handle OS differences
- ❌ macOS requires accessibility permission
- ❌ Potential security risks

#### Approach B: Clipboard Polling

**Implementation**:
```typescript
setInterval(() => {
  const text = clipboard.readText()
  if (text !== lastText) {
    // Detected new text
  }
}, 100)
```

**Pros**:
- ✅ Simple implementation
- ✅ Good cross-platform compatibility
- ✅ No special permissions needed

**Cons**:
- ❌ Cannot get selection position
- ❌ Cannot distinguish user selection from other copy operations
- ❌ Polling consumes resources
- ❌ Response delay

#### Approach C: Accessibility API (No Hook)

**Implementation**:
- Uses OS accessibility APIs
- Listens for text selection events

**Pros**:
- ✅ Officially supported method
- ✅ Relatively safe
- ✅ Can get partial position info

**Cons**:
- ❌ Medium implementation complexity
- ❌ Many compatibility issues
- ❌ Some apps don't support
- ❌ macOS requires accessibility permission

#### Comparison Table

| Feature | Native Hook | Clipboard Polling | Accessibility API |
|---------|------------|-------------------|-------------------|
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Position Accuracy | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ |
| Compatibility | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Development Difficulty | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Maintenance Cost | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| User Experience | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

**Conclusion**: Cherry Studio chose the Native Hook approach because it provides the best performance and user experience, despite higher development and maintenance costs.

### 8.2 Window Management Approaches

#### Approach A: Preloaded Window Pool (Current)

**Implementation**:
```typescript
// Pre-create windows
private preloadedActionWindows: BrowserWindow[] = []

// Get from pool when needed
private popActionWindow(): BrowserWindow {
  return this.preloadedActionWindows.pop() || this.createPreloadedActionWindow()
}
```

**Pros**:
- ✅ Extremely fast response (windows already loaded)
- ✅ Good user experience
- ✅ High resource utilization

**Cons**:
- ❌ Extra memory usage
- ❌ Slightly higher implementation complexity

#### Approach B: Create On Demand

**Implementation**:
```typescript
// Create new window each time
private createActionWindow(): BrowserWindow {
  const window = new BrowserWindow({...})
  window.loadFile('...')
  return window
}
```

**Pros**:
- ✅ Low memory usage
- ✅ Simple implementation

**Cons**:
- ❌ Noticeable delay on first display
- ❌ Poor user experience

#### Approach C: Singleton Window Reuse

**Implementation**:
```typescript
// Create only one window, reuse it
private actionWindow: BrowserWindow | null = null
```

**Pros**:
- ✅ Minimal memory usage
- ✅ Simple management

**Cons**:
- ❌ Cannot open multiple action windows simultaneously
- ❌ Limited functionality

**Conclusion**: Current approach balances performance and resource usage, suitable for most use cases.

### 8.3 State Management Approaches

#### Approach A: Redux + IPC (Current)

**Architecture**:
```
Settings UI → Redux → IPC → ConfigManager → File System
              ↓                    ↓
         Other Windows ← IPC ←  Subscribe
```

**Pros**:
- ✅ Good state consistency
- ✅ Traceable and debuggable
- ✅ Supports multi-window sync

**Cons**:
- ❌ Complex architecture
- ❌ IPC communication overhead

#### Approach B: Direct IPC

**Architecture**:
```
Settings UI → IPC → ConfigManager → File System
```

**Pros**:
- ✅ Simple and direct
- ✅ Reduces intermediate layers

**Cons**:
- ❌ Difficult to manage complex state
- ❌ Multi-window sync difficult

**Conclusion**: Redux approach suits complex apps requiring multi-window collaboration.

---

## 9. Advantages and Disadvantages

### 9.1 Overall Advantages

#### 9.1.1 User Experience

✅ **Globally Available**
- Works in any application
- No need to switch apps or windows
- True "select and use"

✅ **Fast Response**
- Preloaded window mechanism ensures instant response
- Toolbar display delay < 100ms
- Action window open delay < 50ms

✅ **Smart Positioning**
- Auto-adapts to screen boundaries
- Follows selection position
- Multi-monitor support

✅ **Non-Intrusive**
- Doesn't affect original app functionality
- Transparent window design
- Auto-hide mechanism

#### 9.1.2 Functionality

✅ **Flexible Trigger Modes**
- Show on selection (for frequent use)
- Ctrl key trigger (avoid false triggers)
- Keyboard shortcut (manual control)

✅ **Application Filtering**
- Predefined blacklist (exclude incompatible apps)
- Whitelist mode (only enable in specified apps)
- Blacklist mode (exclude specified apps)

✅ **Custom Actions**
- Supports built-in actions (copy, search, translate)
- Supports custom AI assistants
- Supports custom search engines

✅ **Rich Configuration Options**
- Toolbar style (compact/standard)
- Window behavior (follow/center, auto-close/pin)
- Opacity adjustment
- Custom CSS

#### 9.1.3 Technical Implementation

✅ **High Performance**
- Native module implements core functionality
- Minimizes IPC communication
- Async processing doesn't block main thread

✅ **Maintainability**
- Clear architectural layering
- Single responsibility principle
- Complete type definitions

✅ **Extensibility**
- Pluggable action mechanism
- Redux state management supports complex scenarios
- Reserved extension interfaces

### 9.2 Overall Disadvantages

#### 9.2.1 System Compatibility

❌ **Platform Limitations**
- Only supports Windows and macOS
- Linux not supported (selection-hook limitation)
- Different behavior across OSes

❌ **Permission Requirements**
- macOS requires accessibility permission
- May be blocked by security software
- May be restricted in enterprise environments

#### 9.2.2 Application Compatibility

❌ **Specific App Issues**
- PDF readers (inaccurate position detection)
- Office apps (may conflict)
- Remote desktop (cannot use)
- Some games (blocked by anti-cheat)

❌ **Fine-Tuning Needs**
- Different apps need different configs
- High cost maintaining fine-tuned lists
- New apps may need blacklist addition

#### 9.2.3 Technical Challenges

❌ **Native Module Maintenance**
- Requires C++ development skills
- Different platforms need separate compilation
- Electron upgrades may cause compatibility issues
- Difficult debugging

❌ **Complex Window Management**
- macOS fullscreen app compatibility issues
- Complex window hierarchy and focus management
- Multi-monitor scenarios need special handling

❌ **Memory Usage**
- Preloaded windows use memory (~50-100MB)
- Memory increases with multiple action windows open
- Potential memory leak risks in long-running sessions

#### 9.2.4 User Experience Issues

❌ **Learning Curve**
- Multiple trigger modes may confuse users
- Many configuration options
- Takes time to adapt

❌ **False Triggers**
- Selection mode easily triggers accidentally
- May interfere with normal operations in some apps

❌ **Performance Impact**
- Continuous background listening consumes CPU
- Perceptible delay on low-performance devices

### 9.3 Security Considerations

#### Risks

❌ **Privacy Risks**
- Can access selected text in any application
- May leak sensitive info (passwords, private conversations)
- Needs clear privacy policy

❌ **Security Risks**
- System-level hooks could be maliciously exploited
- Need to ensure code security
- Regular security audits needed

#### Mitigation Measures

✅ **Transparency**
- Open source code, auditable
- Clearly inform users of features and permissions

✅ **User Control**
- Users can disable anytime
- Application filtering mechanism
- Don't upload or log selected text (unless user initiates action)

---

## 10. Platform Compatibility

### 10.1 Windows

**Supported Versions**: Windows 10 and above

**Feature Support**:
- ✅ Full feature support
- ✅ Toolbar window type: `toolbar`
- ✅ No special permissions needed
- ✅ Coordinate system: Physical coordinates, needs conversion

**Known Issues**:
- Some apps (e.g., Excel) may conflict with toolbar
- Cannot use in Remote Desktop sessions
- Some screenshot tools may interfere with detection

**Optimization Measures**:
- Predefined blacklist excludes incompatible apps
- Fine-tuned list optimizes specific apps
- Auto coordinate conversion handles DPI scaling

### 10.2 macOS

**Supported Versions**: macOS 10.15 and above

**Feature Support**:
- ✅ Full feature support
- ✅ Toolbar window type: `panel`
- ✅ Fullscreen app compatibility
- ⚠️ Requires accessibility permission

**Known Issues**:
- Needs authorization in "System Preferences > Security & Privacy > Accessibility"
- Complex window management in fullscreen apps (Dock icon may disappear)
- Focus management needs special handling (avoid bringing other windows to front)

**Optimization Measures**:
- Provides permission setup guidance
- Special focus and hierarchy management logic
- `setVisibleOnAllWorkspaces` supports fullscreen apps
- `showInactive()` avoids activating windows

### 10.3 Linux

**Support Status**: ❌ Not supported

**Reasons**:
- selection-hook library doesn't support Linux
- Large differences in X11/Wayland text selection APIs
- Needs additional development and testing work

**Future Plans**:
- May add support in future versions
- Needs community contributions or separate native module

### 10.4 Platform Difference Handling

#### Detect Platform

```typescript
import { isMac, isWin } from '@main/constant'

const isSupportedOS = isWin || isMac

if (!isSupportedOS) {
  // Unsupported platform
  return false
}
```

#### Platform-Specific Code

```typescript
// Windows specific
if (isWin) {
  this.toolbarWindow = new BrowserWindow({
    type: 'toolbar',
    focusable: false
  })
}

// macOS specific
if (isMac) {
  this.toolbarWindow = new BrowserWindow({
    type: 'panel',
    hiddenInMissionControl: true,
    acceptFirstMouse: true
  })
}
```

#### Config File Differences

```typescript
// Windows config
SELECTION_PREDEFINED_BLACKLIST.WINDOWS: [
  'explorer.exe',
  'excel.exe',
  // ...
]

// macOS config
SELECTION_PREDEFINED_BLACKLIST.MAC: [
  'com.apple.finder'
]
```

---

## 11. Summary

### 11.1 Core Strengths

1. **True Global Functionality**: Cross-application text selection and operations
2. **Excellent Performance**: Native implementation + preloading mechanism
3. **Flexible Configuration**: Multiple trigger modes and filtering options
4. **Good User Experience**: Smart positioning, fast response, non-intrusive

### 11.2 Main Challenges

1. **Platform Compatibility**: Different OSes need special handling
2. **Application Compatibility**: Some apps need fine-tuning
3. **Technical Complexity**: High cost for native module development and maintenance
4. **Security and Privacy**: Need clear security measures

### 11.3 Suitable Scenarios

✅ **Suitable for**:
- Users who frequently need translation
- Users who need quick information lookup
- Users who need AI-assisted reading
- Cross-application collaboration scenarios

❌ **Not suitable for**:
- Linux users
- Users who don't want to grant accessibility permission
- Low-performance devices
- Enterprise restricted environments

### 11.4 Future Outlook

Possible improvement directions:

1. **Expand Platform Support**
   - Linux support
   - Better Windows 11 integration

2. **Enhanced Features**
   - OCR integration (image text recognition)
   - Handwriting recognition
   - Real-time multi-language translation

3. **Performance Optimization**
   - Reduce memory usage
   - Optimize response speed
   - Lower CPU usage

4. **Improved Compatibility**
   - Expand app support
   - Smarter app detection
   - Automated fine-tuning

5. **User Experience**
   - Simplify configuration interface
   - Intelligent action recommendations
   - Better visual feedback

---

## 12. References

### 12.1 Related Files

- `src/main/services/SelectionService.ts` - Core service implementation
- `src/main/configs/SelectionConfig.ts` - Configuration definitions
- `src/renderer/src/windows/selection/toolbar/SelectionToolbar.tsx` - Toolbar UI
- `src/renderer/src/windows/selection/action/SelectionActionApp.tsx` - Action window UI
- `src/renderer/src/hooks/useSelectionAssistant.ts` - React Hook
- `src/renderer/src/types/selectionTypes.d.ts` - Type definitions

### 12.2 Dependencies

- [selection-hook](https://www.npmjs.com/package/selection-hook) - Native text selection listening module
- [Electron](https://www.electronjs.org/) - Desktop application framework
- [React](https://react.dev/) - UI framework

### 12.3 Related API Documentation

- [Electron BrowserWindow](https://www.electronjs.org/docs/latest/api/browser-window)
- [Electron screen](https://www.electronjs.org/docs/latest/api/screen)
- [Windows Hooks API](https://docs.microsoft.com/en-us/windows/win32/winmsg/hooks)
- [macOS Accessibility API](https://developer.apple.com/documentation/accessibility)

---

**Document Version**: 1.0  
**Creation Date**: 2025-11-23  
**Author**: Cherry Studio Development Team  
**Maintenance Status**: Active
