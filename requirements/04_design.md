# Titan Explore & More: Design Document

**Version:** 1.0  
**Date:** January 17, 2026  
**Status:** Planning  
**Owner:** Design Team

---

## Overview

This document defines the user experience, visual design, and interaction patterns for **Titan Explore & More**. The design philosophy prioritizes:
1. **Warmth & Authenticity** - Human-centered, not transactional
2. **Clarity & Simplicity** - Easy to understand and use
3. **Privacy & Safety** - User feels in control and protected
4. **Delight & Discovery** - Moments of joy and serendipity

---

## Design Principles

### 1. Human-First, Not Algorithm-First
- Show real people, not just profiles
- Emphasize shared humanity over metrics
- Use warm, conversational language
- Avoid gamification that feels manipulative

### 2. Quality Over Quantity
- Curated daily matches, not endless swiping
- Thoughtful recommendations, not spam
- Meaningful connections, not numbers
- Depth over breadth

### 3. Context-Aware
- Adapt to user's current state (energy, mood, time)
- Show relevant matches at the right time
- Respect user's schedule and availability
- Provide timely, not intrusive, notifications

### 4. Privacy by Design
- Clear, granular privacy controls
- Transparent about data usage
- User owns and controls their data
- Safety features are prominent, not hidden

### 5. Inclusive & Accessible
- Welcoming to all identities and orientations
- Accessible to users with disabilities
- Culturally sensitive
- Multiple languages (future)

---

## Visual Design System

### Color Palette

**Primary: Sunrise Gradient** (from TitanOS PositivityTheme)
```swift
LinearGradient(
    colors: [
        Color(hex: "FF6B6B"), // Coral
        Color(hex: "FFA500"), // Orange
        Color(hex: "FFD700")  // Gold
    ],
    startPoint: .topLeading,
    endPoint: .bottomTrailing
)
```

**Secondary: Domain Colors**
- Love: Rose (#FF6B9D)
- Friendship: Sky Blue (#4A90E2)
- Learning: Purple (#9B59B6)
- Events: Teal (#1ABC9C)
- Activities: Green (#2ECC71)
- Entertainment: Indigo (#5C6BC0)
- Shopping: Amber (#FFC107)
- Growth: Deep Purple (#673AB7)

**Neutrals:**
- Background: Off-White (#FDFBF7)
- Surface: White (#FFFFFF)
- Text Dark: Charcoal (#1F2937)
- Text Subtle: Gray (#6B7280)
- Border: Light Gray (#E5E7EB)

**Semantic:**
- Success: Green (#10B981)
- Warning: Yellow (#F59E0B)
- Error: Red (#EF4444)
- Info: Blue (#3B82F6)

### Typography

**Font Family:** SF Pro (iOS system font)

**Hierarchy:**
```swift
// Headings
.largeTitle: 34pt, Bold
.title: 28pt, Bold
.title2: 22pt, Semibold
.title3: 20pt, Semibold

// Body
.body: 17pt, Regular
.callout: 16pt, Regular
.subheadline: 15pt, Regular
.footnote: 13pt, Regular
.caption: 12pt, Regular

// Special
.hero: 48pt, Bold (for splash screens)
.stat: 36pt, Bold (for compatibility scores)
```

### Spacing

**Base Unit:** 8pt

**Scale:**
- XXS: 4pt
- XS: 8pt
- S: 12pt
- M: 16pt
- L: 24pt
- XL: 32pt
- XXL: 48pt
- XXXL: 64pt

### Corner Radius

- Small: 8pt (buttons, tags)
- Medium: 16pt (cards)
- Large: 24pt (modals, sheets)
- XLarge: 32pt (hero cards)
- Circle: 50% (avatars, icons)

### Shadows

```swift
// Card Shadow
.shadow(
    color: Color.black.opacity(0.05),
    radius: 10,
    x: 0,
    y: 4
)

// Elevated Shadow
.shadow(
    color: Color.black.opacity(0.1),
    radius: 20,
    x: 0,
    y: 10
)

// Floating Shadow
.shadow(
    color: Color.black.opacity(0.15),
    radius: 30,
    x: 0,
    y: 15
)
```

---

## Component Library

### 1. Match Card

**Purpose:** Display a potential match with key information

**Anatomy:**
```
┌─────────────────────────────────────┐
│                                     │
│         [Large Photo]               │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Name, Age                   │   │
│  │ Compatibility: 87%          │   │
│  │ ━━━━━━━━━━━━━━━━━━━━━━━━━  │   │
│  │                             │   │
│  │ 🎯 Shared Values            │   │
│  │ 🧠 Similar Personality      │   │
│  │ 🎨 Both Love Art            │   │
│  └─────────────────────────────┘   │
│                                     │
│  [❌ Pass]  [ℹ️ Info]  [❤️ Like]   │
└─────────────────────────────────────┘
```

**States:**
- Default: Full card visible
- Expanded: Tap "Info" to see full profile
- Liked: Card animates out to the right
- Passed: Card animates out to the left
- SuperLiked: Card animates up with sparkle effect

**Code:**
```swift
struct MatchCard: View {
    let match: Match
    @State private var offset: CGSize = .zero
    @State private var isExpanded = false
    
    var body: some View {
        ZStack(alignment: .bottom) {
            // Photo
            AsyncImage(url: match.primaryPhotoURL) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.2)
            }
            .frame(height: 500)
            .clipped()
            
            // Gradient Overlay
            LinearGradient(
                colors: [.clear, .black.opacity(0.7)],
                startPoint: .top,
                endPoint: .bottom
            )
            
            // Info Card
            VStack(alignment: .leading, spacing: 12) {
                HStack {
                    Text("\(match.matchedUser.firstName), \(match.matchedUser.age)")
                        .font(.title2)
                        .fontWeight(.bold)
                        .foregroundColor(.white)
                    
                    Spacer()
                    
                    CompatibilityBadge(score: match.compatibilityScore)
                }
                
                // Top 3 Compatibility Factors
                ForEach(match.compatibilityFactors.prefix(3)) { factor in
                    CompatibilityFactorRow(factor: factor)
                }
            }
            .padding()
            .background(.ultraThinMaterial)
            .cornerRadius(24, corners: [.topLeft, .topRight])
        }
        .cornerRadius(24)
        .shadow(color: .black.opacity(0.1), radius: 20, y: 10)
        .offset(offset)
        .gesture(
            DragGesture()
                .onChanged { gesture in
                    offset = gesture.translation
                }
                .onEnded { gesture in
                    handleSwipe(gesture.translation)
                }
        )
    }
    
    private func handleSwipe(_ translation: CGSize) {
        if translation.width > 100 {
            // Swipe right = Like
            withAnimation {
                offset = CGSize(width: 500, height: 0)
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                onLike()
            }
        } else if translation.width < -100 {
            // Swipe left = Pass
            withAnimation {
                offset = CGSize(width: -500, height: 0)
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                onPass()
            }
        } else {
            // Return to center
            withAnimation {
                offset = .zero
            }
        }
    }
}
```

### 2. Compatibility Badge

**Purpose:** Show compatibility score prominently

**Design:**
```swift
struct CompatibilityBadge: View {
    let score: Double
    
    var color: Color {
        switch score {
        case 90...100: return .green
        case 75..<90: return .blue
        case 60..<75: return .orange
        default: return .gray
        }
    }
    
    var body: some View {
        HStack(spacing: 4) {
            Image(systemName: "heart.fill")
                .font(.caption)
            Text("\(Int(score))%")
                .font(.subheadline)
                .fontWeight(.semibold)
        }
        .foregroundColor(.white)
        .padding(.horizontal, 12)
        .padding(.vertical, 6)
        .background(color)
        .cornerRadius(16)
    }
}
```

### 3. Message Bubble

**Purpose:** Display chat messages

**Design:**
```swift
struct MessageBubble: View {
    let message: Message
    let isFromCurrentUser: Bool
    
    var body: some View {
        HStack {
            if isFromCurrentUser { Spacer() }
            
            VStack(alignment: isFromCurrentUser ? .trailing : .leading, spacing: 4) {
                Text(message.content)
                    .font(.body)
                    .foregroundColor(isFromCurrentUser ? .white : .primary)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 10)
                    .background(
                        isFromCurrentUser
                            ? Color.blue
                            : Color.gray.opacity(0.2)
                    )
                    .cornerRadius(18)
                
                Text(message.sentAt, style: .time)
                    .font(.caption2)
                    .foregroundColor(.secondary)
            }
            
            if !isFromCurrentUser { Spacer() }
        }
    }
}
```

### 4. Event Card

**Purpose:** Display event with key details

**Design:**
```swift
struct EventCard: View {
    let event: Event
    
    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // Event Image
            AsyncImage(url: event.imageURL) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.2)
            }
            .frame(height: 200)
            .clipped()
            .cornerRadius(16)
            
            // Event Info
            VStack(alignment: .leading, spacing: 8) {
                Text(event.title)
                    .font(.headline)
                    .foregroundColor(.primary)
                
                HStack {
                    Label(event.startTime, systemImage: "calendar")
                    Label(event.venueName ?? "", systemImage: "mappin")
                }
                .font(.subheadline)
                .foregroundColor(.secondary)
                
                // Attending Matches
                if !event.attendingMatches.isEmpty {
                    HStack(spacing: -8) {
                        ForEach(event.attendingMatches.prefix(3)) { match in
                            AsyncImage(url: match.primaryPhotoURL) { image in
                                image
                                    .resizable()
                                    .aspectRatio(contentMode: .fill)
                            } placeholder: {
                                Color.gray
                            }
                            .frame(width: 32, height: 32)
                            .clipShape(Circle())
                            .overlay(Circle().stroke(Color.white, lineWidth: 2))
                        }
                        
                        if event.attendingMatches.count > 3 {
                            Text("+\(event.attendingMatches.count - 3)")
                                .font(.caption)
                                .foregroundColor(.white)
                                .frame(width: 32, height: 32)
                                .background(Color.blue)
                                .clipShape(Circle())
                                .overlay(Circle().stroke(Color.white, lineWidth: 2))
                        }
                    }
                }
            }
            .padding(.horizontal)
        }
        .background(Color.white)
        .cornerRadius(16)
        .shadow(color: .black.opacity(0.05), radius: 10, y: 4)
    }
}
```

---

## User Flows

### Flow 1: Onboarding

```
1. Welcome Screen
   ↓
2. Explain Value Proposition
   "Find people who truly get you"
   ↓
3. Select Active Domains
   [Love] [Friendship] [Learning] [Activities]
   ↓
4. Psychometric Assessment
   "Let's understand what makes you, you"
   - Big 5 Personality (10 questions)
   - Love Languages (5 questions)
   - Values (select top 5)
   ↓
5. Set Preferences
   - Age range
   - Distance
   - Gender preference (for Love)
   ↓
6. Upload Photos
   "Show your authentic self"
   - Minimum 2 photos
   - Photo verification
   ↓
7. Review Privacy Settings
   "You're in control"
   - What to share per domain
   - Visibility settings
   ↓
8. Generate First Matches
   "Finding your people..."
   [Loading animation]
   ↓
9. First Match!
   "Meet Sarah, 87% compatible"
```

**Design Notes:**
- Use illustrations, not stock photos
- Progress indicator at top
- Can skip and complete later
- Warm, encouraging copy
- Estimated time: 10 minutes

### Flow 2: Daily Match Discovery

```
1. Notification
   "New matches available! ✨"
   ↓
2. Open App → Explore Tab
   ↓
3. Domain Selector
   [Love] [Friendship] [Learning] [Events]
   ↓
4. Match Stack (selected domain)
   - See first match card
   - Swipe right (like) or left (pass)
   - Tap info for full profile
   ↓
5. If Mutual Like:
   "It's a match! 🎉"
   [Celebration animation]
   ↓
6. Start Conversation
   - AI-generated conversation starters
   - Or write your own
   ↓
7. Schedule Meetup
   - Suggest time based on availability
   - Add to calendar
```

**Design Notes:**
- Limit to 10 matches per domain per day
- Show progress (5/10 matches remaining)
- Celebrate mutual matches
- Encourage real meetups, not endless chatting

### Flow 3: Messaging

```
1. Connections Tab
   - List of all connections
   - Sorted by last interaction
   - Unread badge
   ↓
2. Select Connection
   ↓
3. Chat View
   - Message history
   - Type message
   - Send
   ↓
4. AI Assistance (optional)
   - Conversation starters
   - Relationship advice
   - Conflict resolution
   ↓
5. Schedule Meetup
   - Tap calendar icon
   - Suggest time
   - Add to both calendars
```

**Design Notes:**
- Clean, iMessage-like interface
- Read receipts (optional)
- Typing indicators
- Photo/video sharing
- Voice messages

### Flow 4: Event Discovery

```
1. Events Tab
   ↓
2. Personalized Feed
   - Events aligned with interests
   - Events with attending matches
   - Nearby events
   ↓
3. Select Event
   ↓
4. Event Detail
   - Description
   - Date, time, location
   - Ticket price
   - Attending matches
   ↓
5. RSVP
   - Add to calendar
   - Notify attending matches
   ↓
6. Day of Event
   - Reminder notification
   - Directions to venue
   - See who's there
   ↓
7. Post-Event
   - "How was it?" prompt
   - Connect with attendees
   - Share photos
```

**Design Notes:**
- Emphasize social aspect (who's going)
- Easy RSVP and calendar sync
- Safety features (share location with friend)
- Post-event connection prompts

---

## Screen Designs

### Home Screen (Explore Tab)

```
┌─────────────────────────────────────┐
│ ☰  Explore                    🔔 ⚙️ │
├─────────────────────────────────────┤
│                                     │
│  Good morning, Sarah! ☀️            │
│  You have 8 new matches today       │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Love        Friendship       │   │
│  │   3            2             │   │
│  ├─────────────────────────────┤   │
│  │ Learning    Events           │   │
│  │   2            1             │   │
│  └─────────────────────────────┘   │
│                                     │
│  Recent Activity                    │
│  ┌─────────────────────────────┐   │
│  │ 💬 New message from Alex     │   │
│  │ ❤️ You matched with Maria    │   │
│  │ 📅 Event tomorrow: Yoga      │   │
│  └─────────────────────────────┘   │
│                                     │
│  Suggested For You                  │
│  ┌───────┐ ┌───────┐ ┌───────┐    │
│  │ Event │ │ Group │ │ Event │    │
│  └───────┘ └───────┘ └───────┘    │
│                                     │
└─────────────────────────────────────┘
```

### Match Discovery Screen

```
┌─────────────────────────────────────┐
│ ←  Love Matches            5/10     │
├─────────────────────────────────────┤
│                                     │
│         [Match Card]                │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Emily, 29                   │   │
│  │ Compatibility: 87% ❤️       │   │
│  │ ━━━━━━━━━━━━━━━━━━━━━━━━━  │   │
│  │                             │   │
│  │ 🎯 Shared Values            │   │
│  │ • Growth & Contribution     │   │
│  │                             │   │
│  │ 🧠 Similar Personality      │   │
│  │ • Both introverted          │   │
│  │                             │   │
│  │ 🎨 Common Interests         │   │
│  │ • Meditation, Hiking        │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌───────┐ ┌───────┐ ┌───────┐    │
│  │   ❌   │ │   ℹ️   │ │   ❤️   │    │
│  │ Pass  │ │ Info  │ │ Like  │    │
│  └───────┘ └───────┘ └───────┘    │
│                                     │
└─────────────────────────────────────┘
```

### Connections Screen

```
┌─────────────────────────────────────┐
│ Connections                    🔍   │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 👤 Alex                   2  │   │
│  │ "See you tomorrow!"          │   │
│  │ 2h ago                       │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 👤 Maria                     │   │
│  │ "Thanks for the book rec!"   │   │
│  │ 1d ago                       │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 👤 David                     │   │
│  │ "How was the hike?"          │   │
│  │ 3d ago                       │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

### Chat Screen

```
┌─────────────────────────────────────┐
│ ← Alex                      📞 📹   │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────┐           │
│  │ Hey! How's it going?│           │
│  │ 2:30 PM             │           │
│  └─────────────────────┘           │
│                                     │
│           ┌─────────────────────┐  │
│           │ Great! Just finished│  │
│           │ my meditation       │  │
│           │ 2:32 PM             │  │
│           └─────────────────────┘  │
│                                     │
│  ┌─────────────────────┐           │
│  │ Nice! Want to grab  │           │
│  │ coffee tomorrow?    │           │
│  │ 2:35 PM             │           │
│  └─────────────────────┘           │
│                                     │
│           ┌─────────────────────┐  │
│           │ Yes! What time?     │  │
│           │ 2:36 PM             │  │
│           └─────────────────────┘  │
│                                     │
├─────────────────────────────────────┤
│ 💡 Suggest time   📷   🎤   📝     │
└─────────────────────────────────────┘
```

---

## Animations & Transitions

### 1. Match Card Swipe
```swift
// Swipe right (like)
.animation(.spring(response: 0.3, dampingFraction: 0.7))
.offset(x: liked ? 500 : 0)
.rotationEffect(.degrees(liked ? 15 : 0))

// Swipe left (pass)
.animation(.spring(response: 0.3, dampingFraction: 0.7))
.offset(x: passed ? -500 : 0)
.rotationEffect(.degrees(passed ? -15 : 0))
```

### 2. Match Celebration
```swift
// Confetti animation
struct ConfettiView: View {
    @State private var confetti: [ConfettiPiece] = []
    
    var body: some View {
        ZStack {
            ForEach(confetti) { piece in
                Circle()
                    .fill(piece.color)
                    .frame(width: 10, height: 10)
                    .offset(x: piece.x, y: piece.y)
                    .opacity(piece.opacity)
            }
        }
        .onAppear {
            generateConfetti()
        }
    }
}
```

### 3. Loading States
```swift
// Shimmer effect for loading cards
struct ShimmerView: View {
    @State private var phase: CGFloat = 0
    
    var body: some View {
        Rectangle()
            .fill(
                LinearGradient(
                    colors: [.gray.opacity(0.3), .gray.opacity(0.1), .gray.opacity(0.3)],
                    startPoint: .leading,
                    endPoint: .trailing
                )
            )
            .offset(x: phase)
            .onAppear {
                withAnimation(.linear(duration: 1.5).repeatForever(autoreverses: false)) {
                    phase = 300
                }
            }
    }
}
```

---

## Accessibility

### VoiceOver Support
```swift
// Match Card
.accessibilityLabel("\(match.matchedUser.firstName), age \(match.matchedUser.age), \(Int(match.compatibilityScore))% compatible")
.accessibilityHint("Swipe right to like, left to pass, or tap for more info")

// Compatibility Badge
.accessibilityLabel("\(Int(score))% compatible")

// Message Bubble
.accessibilityLabel("Message from \(message.sender.firstName): \(message.content)")
```

### Dynamic Type
```swift
// All text uses scalable fonts
Text("Hello")
    .font(.body)
    .dynamicTypeSize(.xSmall ... .xxxLarge)
```

### Color Contrast
- All text meets WCAG AA standards (4.5:1 for normal text)
- Interactive elements have 3:1 contrast ratio
- Focus indicators for keyboard navigation

---

## Conclusion

This design document provides a comprehensive blueprint for creating a warm, authentic, and delightful user experience for Titan Explore & More. Key principles:

1. **Human-First:** Real connections, not transactions
2. **Quality:** Curated matches, not endless swiping
3. **Privacy:** User control and transparency
4. **Delight:** Moments of joy and discovery

**Next Steps:**
1. Create high-fidelity mockups in Figma
2. Build interactive prototype
3. Conduct user testing
4. Iterate based on feedback
5. Implement in SwiftUI

---

*Document prepared by: Design Team*  
*Last updated: January 17, 2026*  
*Next review: February 1, 2026*
