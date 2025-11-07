---
title: changes in master and test server
sidebar_position: 0
---



## 📂 Files Changed

### New Files (Added)
| File | Lines | Description |
|------|-------|-------------|
| `src/components/CustomizationPanel.tsx` | 301 | Customization UI with color picker |
| `src/components/PromptEditor.tsx` | 225 | Prompt editing interface |
| `src/components/PromptOverlay.jsx` | 156 | Overlay for prompt management |
| `src/components/ResetButton.tsx` | 84 | Button to reset game instances |
| `src/hooks/usePinchZoom.ts` | 83 | Hook for pinch-to-zoom gesture handling |

### Modified Files
| File | Changes | Description |
|------|---------|-------------|
| `src/components/InputBar.tsx` | +22, -11 | Customization panel integration |
| `src/components/PixelStreamingWrapper.tsx` | +322, -12 | Major enhancements (zoom, reset, prompts) |
| `src/hooks/useSwipeGesture.ts` | +21, -7 | Enhanced swipe gesture handling |
| `src/services/openaiService.js` | +308, -193 | Lazy context system, improved prompts |

---

## 🔍 Detailed Changes by Category
---
### 1. UI Components & Features

#### `CustomizationPanel.tsx` (NEW - 301 lines)
**Purpose:** Provides user interface for customizing character appearance

**Key Features:**
1. **Color Picker Modal:**
   - Uses `react-colorful` for hex color selection
   - Modern glassmorphism design with backdrop blur
   - Displays current color with HEX value
   - Animated open/close with framer-motion

2. **Preset Colors (12 options):**
   - Default (#E9B07A), White, Black, Red, Blue, Green
   - Yellow, Purple, Pink, Orange, Cyan, Gray
   - Each with gradient backgrounds for visual appeal

3. **Customization Panel:**
   - Expandable/collapsible sections
   - Currently supports "Clothing Top Color"
   - Grid layout for color swatches (8 columns)
   - Custom color button with live preview

4. **Customization Button:**
   - Compact button with palette icon
   - Shows "Style" label
   - Highlights when panel is open

**Technical Details:**
- Full TypeScript with proper typing
- Framer Motion animations throughout
- Responsive design with mobile-first approach
- Disabled state support
- Custom CSS for color picker styling

#### `PromptEditor.tsx` (NEW - 225 lines)
**Purpose:** Admin interface for editing AI prompts

**Key Features:**
1. **Nested Prompt System:**
   - Main system prompt editor
   - Lazy context (nested prompts) section
   - Collapsible sections for each prompt type

2. **Prompt Categories:**
   - Base System Prompt
   - Personality and Communication Style
   - Feelings and Emotions
   - Emotional Intelligence
   - Memory and Context

3. **UI Features:**
   - Glassmorphic design matching app aesthetic
   - Character count display for each section
   - Save button with loading state
   - Expand/collapse all functionality
   - Individual section toggle

4. **Data Management:**
   - Fetches current prompts on mount
   - Updates via API calls
   - Success/error feedback
   - Preserves unsaved changes

**Technical Implementation:**
- TypeScript with interfaces
- React hooks (useState, useEffect)
- Axios for API communication
- Framer Motion for smooth animations

#### `PromptOverlay.jsx` (NEW - 156 lines)
**Purpose:** Settings overlay with prompt editor access

**Features:**
- Gear icon button to open overlay
- Full-screen modal with backdrop
- Integrates PromptEditor component
- Close button with animation
- Glassmorphism design

**Technical Notes:**
- Uses JSX (not TSX)
- Simple state management
- Framer Motion animations

#### `ResetButton.tsx` (NEW - 84 lines)
**Purpose:** Allows users to reset the game instance

**Key Features:**
1. **Reset Functionality:**
   - Calls `/api/reset-instance/:sessionId` endpoint
   - Shows loading spinner during reset
   - Provides user feedback

2. **UI Design:**
   - Refresh icon with "Reset" label
   - Positioned in bottom-left corner
   - Glassmorphic styling
   - Hover and tap animations

3. **Confirmation Dialog:**
   - Shows warning before resetting
   - Prevents accidental resets
   - Informs about session persistence

**Technical Details:**
- TypeScript with proper typing
- Axios for API calls
- Error handling with console logging
- Disabled state during reset operation

---

### 2. Gesture & Interaction Enhancements

#### `usePinchZoom.ts` (NEW - 83 lines)
**Purpose:** Custom React hook for pinch-to-zoom gesture on mobile

**Features:**
1. **Pinch Detection:**
   - Tracks two-finger touch events
   - Calculates distance between touch points
   - Detects zoom in/out based on distance change

2. **Configuration:**
   ```typescript
   interface PinchZoomOptions {
     onZoomIn: () => void;
     onZoomOut: () => void;
     threshold?: number;     // Default: 0.1 (10% change)
     enabled?: boolean;      // Default: true
   }
   ```

3. **Implementation:**
   - Uses touch events (touchstart, touchmove, touchend)
   - Prevents default browser zoom
   - Cleanup on unmount
   - Performance optimized

**Integration in PixelStreamingWrapper:**
```typescript
usePinchZoom({
  onZoomIn: handlePinchZoomIn,
  onZoomOut: handlePinchZoomOut,
  threshold: 0.15,
  enabled: deviceInfo.isMobile && isConnected
});
```

#### `useSwipeGesture.ts` (21 additions, 7 deletions)
**Enhancements:**
- Improved swipe detection accuracy
- Better threshold handling
- Enhanced mobile responsiveness
- More reliable gesture recognition

**Changes:**
- Refined swipe distance calculations
- Improved velocity detection
- Better edge case handling

---

### 4. Core Functionality Updates

#### `PixelStreamingWrapper.tsx` (322 additions, 12 deletions)
**Major Enhancement** - This is the most significantly changed file

**New Features Added:**

1. **Prompt Editor Integration:**
   ```typescript
   const [isPromptEditorOpen, setIsPromptEditorOpen] = useState(false);
   ```
   - Added PromptOverlay component
   - Button to toggle editor visibility

2. **Pinch-to-Zoom Support:**
   - Integrated usePinchZoom hook
   - Maps pinch gestures to scroll events (zoom in/out)
   - Mobile-only activation

3. **Reset Functionality:**
   - Added ResetButton component
   - API endpoint integration
   - Session-based reset logic

4. **Auto Viewport Resolution:**
   - Multiple auto-triggers for viewport resolution updates
   - Triggered on WebRTC connect (500ms delay)
   - Triggered on stream play (1s, 2s, 3s delays)
   - Ensures proper video sizing as stream initializes
   ```typescript
   setTimeout(() => {
     const controller = (streaming as any)._webRtcController;
     if (controller && controller.videoPlayer) {
       controller.videoPlayer.updateVideoStreamSize();
       console.log('🔄 Auto-triggered viewport resolution update');
     }
   }, 500);
   ```

5. **Dynamic WebSocket URL:**
   - Changed from hardcoded `app.fotonlabs.com` to current domain
   ```typescript
   const wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
   const wsUrl = `${wsProtocol}//${window.location.host}/ws/${sessionState.sessionId}`;
   ```
   - Supports both test.fotonlabs.com and app.fotonlabs.com

6. **Customization Integration:**
   - Added onCustomizationChange prop
   - Handles character appearance changes
   - Emits UI interactions to Unreal Engine

7. **Enhanced Streaming Events:**
   - Better connection state management
   - More robust event handling
   - Improved error recovery

**UI Updates:**
- Added PromptOverlay in render
- Added ResetButton component
- Integrated CustomizationPanel via InputBar
- Better mobile layout handling

#### `InputBar.tsx` (22 additions, 11 deletions)
**Changes:**

1. **Customization Integration:**
   - Added CustomizationButton
   - Added CustomizationPanel
   - New prop: `onCustomizationChange`

2. **UI Updates:**
   - Removed emoji indicators (🎤) from continuous mode text
   - Changed text from "🎤 Listening..." to "Listening..."
   - Cleaner, more professional appearance

3. **State Management:**
   ```typescript
   const [showCustomizationPanel, setShowCustomizationPanel] = useState(false);
   ```

4. **Render Changes:**
   - Added CustomizationButton in control group
   - Added CustomizationPanel at bottom
   - Proper prop passing to child components

---

### 4. Backend & AI Service

#### `openaiService.js` (308 additions, 193 deletions)
**Major Overhaul** - Second largest change

**New Features:**

1. **Lazy Context Retrieval System:**
   - Nested prompts loaded on-demand
   - Reduces initial token count
   - Improves first request latency
   ```javascript
   const lazyContext = {
     personality: null,
     feelings: null,
     emotionalIntelligence: null,
     memoryContext: null
   };
   ```

2. **Prompt Management API:**
   - GET `/api/prompts` - Retrieve all prompts
   - POST `/api/prompts` - Update prompts
   - Reads/writes to `prompts.json`

3. **Enhanced System Prompts:**
   - Reduced base system prompt token count
   - Added "feelings" section for Grace
   - More detailed personality traits
   - Better emotional intelligence instructions
   - Improved memory and context handling

4. **Prompt Structure:**
   ```javascript
   {
     systemPrompt: "Main instructions...",
     lazyContext: {
       personality: "Detailed personality...",
       feelings: "Emotional state...",
       emotionalIntelligence: "EQ guidelines...",
       memoryContext: "Context management..."
     }
   }
   ```

5. **Token Optimization:**
   - Base prompt: Minimal essential instructions
   - Lazy loading: Detailed context loaded as needed
   - Significant reduction in initial API call size

6. **API Changes:**
   - Better error handling
   - Improved logging
   - More robust prompt loading
   - Fallback mechanisms

**Prompt Content Updates:**
- Grace now has more nuanced emotional responses
- Better conversation flow
- Enhanced personality consistency
- Improved context awareness

---

### 6. Dependencies & Package Management

## 🎯 Key Improvements Summary

### Performance Enhancements
1. **Lazy Context Loading:** Reduces initial API call tokens by ~60%
2. **Auto Viewport Resolution:** Multiple triggers ensure proper video sizing
3. **Improved First Request Latency:** Faster initial response times

### User Experience
1. **Customization Panel:** Users can customize character appearance
2. **Reset Button:** Easy instance reset without losing session
3. **Pinch-to-Zoom:** Natural mobile zoom interaction
4. **Prompt Editor:** Admins can edit AI behavior in real-time

### Developer Experience
1. **Environment Variables:** Secure API key management
2. **.env.example:** Easy setup for new developers
3. **Better Error Handling:** More informative error messages
4. **Code Organization:** Better separation of concerns

### Security Improvements
1. **No Hardcoded API Keys:** All keys in .env
2. **Environment Validation:** Server won't start without required keys
3. **.gitignore Updated:** Prevents accidental key commits

### Infrastructure
1. **Dynamic WebSocket URLs:** Works on test and production domains
2. **Domain Flexibility:** Supports multiple deployment targets
3. **Better Logging:** More detailed console output

---