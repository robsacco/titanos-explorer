# Titan Explore & More: Integration Document

**Version:** 1.0  
**Date:** January 17, 2026  
**Status:** Planning  
**Owner:** Engineering Team

---

## Overview

This document defines how **Titan Explore & More** integrates with existing TitanOS systems and external services. The integration strategy prioritizes:
1. **Seamless UX** - Explore feels native to TitanOS
2. **Data Reuse** - Leverage existing user data
3. **Privacy** - Respect user consent and controls
4. **Modularity** - Clean separation of concerns

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    TitanOS Core                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Identity   │  │     Goals    │  │    Health    │ │
│  │   Profile    │  │   (RPM)      │  │  (HealthKit) │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Schedule    │  │  Meditation  │  │   AI Coach   │ │
│  │  (Calendar)  │  │              │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│              Integration Layer (New)                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │         ExploreIntegrationService                │  │
│  │  - Data Sync   - Privacy Filter  - AI Context   │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│            Titan Explore & More (New)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Matching   │  │  Messaging   │  │    Events    │ │
│  │   Engine     │  │              │  │  Discovery   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Groups     │  │  Mentorship  │  │    Tribes    │ │
│  │              │  │              │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│              External Services                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ Firebase │  │  Google  │  │  Gemini  │  │ Claude │ │
│  │          │  │  Cloud   │  │   AI     │  │   AI   │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## Integration Points

### 1. TitanOS Core → Explore

#### 1.1 Identity & Profile

**Source:** `ProfileManager.shared.profile`

**Data Flow:**
```swift
TitanOS UserProfile
    ↓
ExploreIntegrationService.syncProfile()
    ↓
ExploreUserProfile (with privacy filtering)
```

**Mapped Fields:**
```swift
// TitanOS → Explore
profile.name → exploreProfile.firstName
profile.birthDate → exploreProfile.birthDate
profile.coreValues → psychometricProfile.coreValues
profile.primaryNeed → psychometricProfile.primaryNeed
profile.secondaryNeed → psychometricProfile.secondaryNeed
```

**Privacy Filter:**
- User must explicitly opt-in to Explore
- Can choose which fields to share
- Can set different visibility per domain

**Implementation:**
```swift
class ExploreIntegrationService {
    static let shared = ExploreIntegrationService()
    
    func syncProfileFromTitanOS() async throws {
        let titanProfile = ProfileManager.shared.profile
        let privacySettings = await getPrivacySettings()
        
        var exploreProfile = ExploreUserProfile(
            userId: titanProfile.id,
            firstName: titanProfile.name,
            birthDate: titanProfile.birthDate
        )
        
        // Apply privacy filters
        if privacySettings.shareCoreValues {
            exploreProfile.psychometrics?.coreValues = titanProfile.coreValues
        }
        
        if privacySettings.sharePrimaryNeed {
            exploreProfile.psychometrics?.primaryNeed = titanProfile.primaryNeed
        }
        
        // Save to Explore database
        try await saveExploreProfile(exploreProfile)
    }
}
```

#### 1.2 Goals & Outcomes (RPM)

**Source:** `RPMOutcome` entities from SwiftData

**Data Flow:**
```swift
TitanOS RPM Outcomes
    ↓
ExploreIntegrationService.extractGoalThemes()
    ↓
ExploreUserProfile.learningGoals / activityGoals
```

**Extraction Logic:**
```swift
func extractGoalThemes(from outcomes: [RPMOutcome]) -> [String: [String]] {
    var themes: [String: [String]] = [
        "learning": [],
        "activities": [],
        "career": [],
        "relationships": [],
        "wellness": []
    ]
    
    for outcome in outcomes where !outcome.isCompleted {
        // Use AI to categorize and extract themes
        let category = await AIOrchestrator.shared.categorizeGoal(outcome.title)
        let theme = await AIOrchestrator.shared.extractTheme(outcome.title)
        
        themes[category]?.append(theme)
    }
    
    return themes
}
```

**Privacy:**
- User controls which goal categories are shared
- Only high-level themes shared, not specific tasks
- Example: "Learning Python" ✅, "Build trading bot for crypto" ❌

#### 1.3 Schedule & Availability

**Source:** `CalendarManager.shared` + `RPMAction` due dates

**Data Flow:**
```swift
TitanOS Calendar Events
    ↓
ExploreIntegrationService.calculateAvailability()
    ↓
UserPreferences.availability
```

**Availability Calculation:**
```swift
func calculateAvailability() async -> [DayOfWeek: [TimeSlot]] {
    let calendar = await CalendarManager.shared.fetchEvents(
        from: Date(),
        to: Date().addingTimeInterval(7 * 24 * 60 * 60) // Next 7 days
    )
    
    var availability: [DayOfWeek: [TimeSlot]] = [:]
    
    for day in DayOfWeek.allCases {
        let dayEvents = calendar.filter { $0.day == day }
        let freeSlots = calculateFreeSlots(from: dayEvents)
        availability[day] = freeSlots
    }
    
    return availability
}
```

**Privacy:**
- Only share free/busy status, not event details
- User can manually override availability
- Can set "Do Not Disturb" periods

#### 1.4 Health & Readiness

**Source:** `HealthManager` + HealthKit

**Data Flow:**
```swift
HealthKit Data
    ↓
TitanOS HealthManager.readinessScore
    ↓
ExploreIntegrationService.getEnergyLevel()
    ↓
Used for match timing recommendations
```

**Integration:**
```swift
func getEnergyLevel() async -> EnergyLevel {
    let readiness = await HealthManager.shared.calculateReadinessScore()
    
    switch readiness {
    case 0..<40: return .low
    case 40..<60: return .moderate
    case 60..<80: return .high
    default: return .veryHigh
    }
}

func recommendMeetingTime(for connection: Connection) async -> Date? {
    let userEnergy = await getEnergyLevel()
    let availability = await calculateAvailability()
    
    // Recommend times when user has high energy and availability
    return findOptimalSlot(energy: userEnergy, availability: availability)
}
```

**Privacy:**
- User opts-in to share energy level
- Only general level shared (low/moderate/high), not raw scores
- Never shares specific health metrics

#### 1.5 Meditation & Spiritual Practice

**Source:** `TMSession` entities + Cosmic Guide usage

**Data Flow:**
```swift
TitanOS Meditation Data
    ↓
ExploreIntegrationService.extractSpiritualProfile()
    ↓
ExploreUserProfile.spiritualInterests
```

**Extraction:**
```swift
func extractSpiritualProfile() async -> SpiritualProfile {
    let sessions = await fetchTMSessions()
    let cosmicReports = await fetchCosmicReports()
    
    return SpiritualProfile(
        meditates: !sessions.isEmpty,
        meditationFrequency: calculateFrequency(sessions),
        spiritualInterests: cosmicReports.map { $0.type }
    )
}
```

**Use Cases:**
- Match with other meditators
- Find meditation retreat partners
- Connect with spiritually-aligned people

#### 1.6 AI Context & Coaching

**Source:** `AIOrchestrator` + `ContextGenerator`

**Data Flow:**
```swift
TitanOS AI Context
    ↓
ExploreIntegrationService.generateMatchContext()
    ↓
AI-powered introductions and recommendations
```

**Integration:**
```swift
func generateIntroductionMessage(
    for match: Match,
    user: ExploreUserProfile,
    matchedUser: ExploreUserProfile
) async throws -> String {
    // Build context from TitanOS
    let userContext = await ContextGenerator.shared.generateContext()
    
    // Add Explore-specific context
    let exploreContext = """
    You are helping \(user.firstName) connect with \(matchedUser.firstName).
    
    Compatibility: \(match.compatibilityScore)%
    Top factors:
    \(match.compatibilityFactors.map { "- \($0.description)" }.joined(separator: "\n"))
    
    Shared interests: \(findSharedInterests(user, matchedUser))
    """
    
    // Generate introduction
    let prompt = """
    \(userContext)
    
    \(exploreContext)
    
    Write a warm, authentic introduction message (2-3 sentences) that:
    1. References a specific compatibility factor
    2. Suggests a conversation starter
    3. Feels personal, not generic
    """
    
    return try await AIOrchestrator.shared.generateResponse(
        prompt: prompt,
        preferredModel: .claude // Better for creative writing
    )
}
```

---

### 2. Explore → TitanOS Core

#### 2.1 Calendar Integration

**Use Case:** Schedule meetups with matches

**Flow:**
```swift
User schedules meetup in Explore
    ↓
ExploreIntegrationService.createCalendarEvent()
    ↓
CalendarManager.shared.addEvent()
    ↓
Syncs to Apple Calendar
```

**Implementation:**
```swift
func scheduleMeetup(
    with connection: Connection,
    date: Date,
    location: String,
    notes: String?
) async throws {
    // Create event in TitanOS calendar
    let event = CalendarEvent(
        title: "Meetup with \(connection.connectedUser.firstName)",
        startTime: date,
        duration: 3600, // 1 hour default
        location: location,
        notes: notes,
        source: .explore
    )
    
    try await CalendarManager.shared.addEvent(event)
    
    // Send calendar invite to match
    try await sendCalendarInvite(to: connection.connectedUserId, event: event)
}
```

#### 2.2 Task Creation

**Use Case:** Create accountability tasks for learning partners

**Flow:**
```swift
Learning Circle creates shared goal
    ↓
ExploreIntegrationService.createSharedTask()
    ↓
RPMAction created for each member
    ↓
Appears in TitanOS task list
```

**Implementation:**
```swift
func createSharedTask(
    for learningCircle: LearningCircle,
    title: String,
    dueDate: Date
) async throws {
    for memberId in learningCircle.memberIds {
        let action = RPMAction(
            title: title,
            dueDate: dueDate,
            outcome: nil, // Or link to learning outcome
            source: .explore,
            metadata: [
                "learningCircleId": learningCircle.id.uuidString,
                "sharedTask": "true"
            ]
        )
        
        try await createAction(for: memberId, action: action)
    }
}
```

#### 2.3 Notifications

**Use Case:** Notify user of new matches, messages, events

**Flow:**
```swift
Explore event occurs (new match, message, etc.)
    ↓
ExploreIntegrationService.sendNotification()
    ↓
NotificationManager.shared.scheduleNotification()
    ↓
User receives notification
```

**Implementation:**
```swift
func notifyNewMatch(_ match: Match) async {
    let notification = NotificationContent(
        title: "New Match!",
        body: "You have a \(match.compatibilityScore)% match with \(match.matchedUser.firstName)",
        category: .explore,
        userInfo: [
            "matchId": match.id.uuidString,
            "domain": match.domain.rawValue
        ]
    )
    
    await NotificationManager.shared.scheduleCreateNotification(
        title: notification.title,
        body: notification.body,
        timeInterval: 1,
        userInfo: notification.userInfo
    )
}
```

#### 2.4 AI Learning

**Use Case:** Improve AI recommendations based on Explore interactions

**Flow:**
```swift
User interacts with Explore (likes, messages, meetups)
    ↓
ExploreIntegrationService.trackInteraction()
    ↓
AIOrchestrator learns preferences
    ↓
Improves future recommendations in both Explore and TitanOS
```

**Implementation:**
```swift
func trackInteraction(_ event: AnalyticsEvent) async {
    // Store in Explore analytics
    try? await saveAnalyticsEvent(event)
    
    // Update AI context for better recommendations
    if event.eventType == .matchLiked {
        await AIOrchestrator.shared.updateUserPreferences(
            userId: event.userId,
            preference: "likes_\(event.eventData["matchedUserTrait"] ?? "")"
        )
    }
}
```

---

### 3. External Service Integrations

#### 3.1 Firebase / Google Cloud

**Purpose:** Backend infrastructure for Phase 2+

**Services:**
- **Firestore:** Real-time database for matches, messages
- **Cloud Storage:** Photo and media storage
- **Cloud Functions:** Serverless compute for matching algorithm
- **Cloud Run:** API server
- **Cloud SQL:** PostgreSQL for relational data

**Integration:**
```swift
// Firebase SDK
import FirebaseCore
import FirebaseFirestore
import FirebaseStorage
import FirebaseAuth

class FirebaseService {
    static let shared = FirebaseService()
    private let db = Firestore.firestore()
    private let storage = Storage.storage()
    
    func syncMatch(_ match: Match) async throws {
        let matchData: [String: Any] = [
            "userId": match.userId.uuidString,
            "matchedUserId": match.matchedUserId.uuidString,
            "domain": match.domain.rawValue,
            "compatibilityScore": match.compatibilityScore,
            "status": match.status.rawValue,
            "createdAt": Timestamp(date: match.createdAt)
        ]
        
        try await db.collection("matches").document(match.id.uuidString).setData(matchData)
    }
    
    func uploadPhoto(_ photo: UserPhoto) async throws -> String {
        guard let imageData = photo.imageData else {
            throw ExploreError.noImageData
        }
        
        let ref = storage.reference().child("photos/\(photo.userId)/\(photo.id).jpg")
        let metadata = StorageMetadata()
        metadata.contentType = "image/jpeg"
        
        _ = try await ref.putDataAsync(imageData, metadata: metadata)
        let url = try await ref.downloadURL()
        
        return url.absoluteString
    }
}
```

#### 3.2 Gemini AI

**Purpose:** AI-powered features

**Use Cases:**
- Generate introduction messages
- Analyze compatibility
- Recommend conversation starters
- Moderate content

**Integration:**
```swift
class ExploreAIService {
    private let orchestrator = AIOrchestrator.shared
    
    func generateIntroduction(for match: Match) async throws -> [String] {
        let prompt = """
        Generate 3 different introduction message options for a dating app match.
        
        User A: \(match.user.firstName), \(match.user.age)
        User B: \(match.matchedUser.firstName), \(match.matchedUser.age)
        
        Compatibility: \(match.compatibilityScore)%
        Shared interests: \(match.sharedInterests.joined(separator: ", "))
        
        Make each message:
        - 2-3 sentences
        - Reference a specific shared interest
        - Ask an open-ended question
        - Feel authentic and warm
        
        Return as JSON array of strings.
        """
        
        let response = try await orchestrator.generateResponse(
            prompt: prompt,
            preferredModel: .gemini
        )
        
        return parseIntroductions(response)
    }
    
    func moderateMessage(_ message: String) async throws -> ModerationResult {
        let prompt = """
        Analyze this message for inappropriate content:
        "\(message)"
        
        Check for:
        - Harassment or hate speech
        - Sexual content
        - Spam or scams
        - Personal information (phone, email, address)
        
        Return JSON: {"safe": true/false, "reason": "explanation"}
        """
        
        let response = try await orchestrator.generateResponse(
            prompt: prompt,
            preferredModel: .gemini
        )
        
        return parseModerationResult(response)
    }
}
```

#### 3.3 Claude AI

**Purpose:** High-quality, empathetic AI interactions

**Use Cases:**
- Relationship advice
- Conflict resolution
- Deep compatibility analysis
- Personalized coaching

**Integration:**
```swift
func provideRelationshipAdvice(
    for connection: Connection,
    situation: String
) async throws -> String {
    let context = await buildRelationshipContext(connection)
    
    let prompt = """
    \(context)
    
    The user is seeking advice about this situation:
    \(situation)
    
    Provide empathetic, actionable advice that:
    1. Acknowledges their feelings
    2. Considers both perspectives
    3. Suggests specific next steps
    4. Respects their autonomy
    
    Keep it under 200 words.
    """
    
    return try await AIOrchestrator.shared.generateResponse(
        prompt: prompt,
        preferredModel: .claude // Better for empathetic responses
    )
}
```

#### 3.4 Location Services

**Purpose:** Location-based matching and event discovery

**Integration:**
```swift
import CoreLocation

class LocationService: NSObject, CLLocationManagerDelegate {
    static let shared = LocationService()
    private let locationManager = CLLocationManager()
    
    func requestLocation() async throws -> CLLocation {
        locationManager.delegate = self
        locationManager.requestWhenInUseAuthorization()
        
        return try await withCheckedThrowingContinuation { continuation in
            self.locationContinuation = continuation
            locationManager.requestLocation()
        }
    }
    
    func updateUserLocation(_ location: CLLocation) async throws {
        let geocoder = CLGeocoder()
        let placemarks = try await geocoder.reverseGeocodeLocation(location)
        
        guard let placemark = placemarks.first else { return }
        
        var profile = try await fetchExploreProfile()
        profile.city = placemark.locality ?? ""
        profile.state = placemark.administrativeArea
        profile.country = placemark.country ?? ""
        profile.latitude = location.coordinate.latitude
        profile.longitude = location.coordinate.longitude
        
        try await saveExploreProfile(profile)
    }
}
```

#### 3.5 Maps & Directions

**Purpose:** Event locations, meetup planning

**Integration:**
```swift
import MapKit

func showEventLocation(_ event: Event) {
    guard let lat = event.latitude, let lon = event.longitude else { return }
    
    let coordinate = CLLocationCoordinate2D(latitude: lat, longitude: lon)
    let placemark = MKPlacemark(coordinate: coordinate)
    let mapItem = MKMapItem(placemark: placemark)
    mapItem.name = event.venueName
    
    mapItem.openInMaps(launchOptions: [
        MKLaunchOptionsDirectionsModeKey: MKLaunchOptionsDirectionsModeWalking
    ])
}
```

#### 3.6 Video Calling

**Purpose:** Remote connections

**Options:**
1. **WebRTC** (self-hosted)
2. **Agora** (third-party)
3. **Twilio Video** (third-party)

**Recommendation:** Start with Agora for ease of integration

**Integration:**
```swift
import AgoraRtcKit

class VideoCallService: NSObject, AgoraRtcEngineDelegate {
    static let shared = VideoCallService()
    private var agoraKit: AgoraRtcEngineKit?
    
    func startCall(with connection: Connection) async throws {
        // Initialize Agora
        agoraKit = AgoraRtcEngineKit.sharedEngine(
            withAppId: SecretsManager.shared.agoraAppId,
            delegate: self
        )
        
        // Configure video
        agoraKit?.enableVideo()
        agoraKit?.setVideoEncoderConfiguration(
            AgoraVideoEncoderConfiguration(
                size: CGSize(width: 640, height: 480),
                frameRate: .fps30,
                bitrate: AgoraVideoBitrateStandard,
                orientationMode: .adaptative
            )
        )
        
        // Join channel
        let channelId = connection.id.uuidString
        try await agoraKit?.joinChannel(
            byToken: nil,
            channelId: channelId,
            info: nil,
            uid: 0
        )
    }
}
```

---

## Data Sync Strategy

### Sync Triggers

1. **App Launch:** Sync all data
2. **Background Refresh:** Sync matches and messages
3. **User Action:** Immediate sync (like, message, etc.)
4. **Periodic:** Every 15 minutes when app is active

### Sync Flow

```swift
class SyncService {
    static let shared = SyncService()
    
    func performFullSync() async throws {
        // 1. Sync profile changes
        try await syncProfile()
        
        // 2. Sync matches
        try await syncMatches()
        
        // 3. Sync messages
        try await syncMessages()
        
        // 4. Sync events
        try await syncEvents()
        
        // 5. Sync groups
        try await syncGroups()
    }
    
    private func syncProfile() async throws {
        let localProfile = try await fetchLocalProfile()
        let cloudProfile = try await fetchCloudProfile()
        
        // Conflict resolution: last-write-wins
        if localProfile.updatedAt > cloudProfile.updatedAt {
            try await uploadProfile(localProfile)
        } else if cloudProfile.updatedAt > localProfile.updatedAt {
            try await downloadProfile(cloudProfile)
        }
    }
    
    private func syncMatches() async throws {
        // Fetch new matches from cloud
        let lastSync = UserDefaults.standard.object(forKey: "lastMatchSync") as? Date ?? Date.distantPast
        let newMatches = try await fetchMatches(since: lastSync)
        
        // Save locally
        for match in newMatches {
            try await saveMatch(match)
        }
        
        // Upload local changes
        let localChanges = try await fetchLocalMatchChanges(since: lastSync)
        for match in localChanges {
            try await uploadMatch(match)
        }
        
        UserDefaults.standard.set(Date(), forKey: "lastMatchSync")
    }
}
```

### Conflict Resolution

**Strategy:** Last-Write-Wins (LWW)

**Implementation:**
```swift
func resolveConflict<T: Syncable>(local: T, cloud: T) -> T {
    return local.updatedAt > cloud.updatedAt ? local : cloud
}

protocol Syncable {
    var id: UUID { get }
    var updatedAt: Date { get }
}
```

---

## API Design (Phase 3)

### REST API Endpoints

```
Authentication:
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
DELETE /api/v1/auth/logout

Profile:
GET    /api/v1/profile
PUT    /api/v1/profile
POST   /api/v1/profile/photos
DELETE /api/v1/profile/photos/:id

Matching:
GET    /api/v1/matches?domain={domain}
POST   /api/v1/matches/:id/like
POST   /api/v1/matches/:id/pass
GET    /api/v1/matches/:id/compatibility

Connections:
GET    /api/v1/connections
GET    /api/v1/connections/:id
DELETE /api/v1/connections/:id

Messages:
GET    /api/v1/connections/:id/messages
POST   /api/v1/connections/:id/messages
PUT    /api/v1/messages/:id/read

Events:
GET    /api/v1/events?city={city}&type={type}
GET    /api/v1/events/:id
POST   /api/v1/events/:id/rsvp
DELETE /api/v1/events/:id/rsvp

Groups:
GET    /api/v1/groups?type={type}
POST   /api/v1/groups
GET    /api/v1/groups/:id
POST   /api/v1/groups/:id/join
DELETE /api/v1/groups/:id/leave
```

### WebSocket Events

```
// Client → Server
{
  "type": "message.send",
  "connectionId": "uuid",
  "content": "Hello!"
}

{
  "type": "typing.start",
  "connectionId": "uuid"
}

{
  "type": "presence.update",
  "status": "online"
}

// Server → Client
{
  "type": "message.received",
  "connectionId": "uuid",
  "message": { ... }
}

{
  "type": "match.new",
  "match": { ... }
}

{
  "type": "connection.created",
  "connection": { ... }
}
```

---

## Testing Strategy

### Unit Tests

```swift
class ExploreIntegrationServiceTests: XCTestCase {
    func testProfileSync() async throws {
        // Given
        let titanProfile = UserProfile(name: "John", birthDate: Date())
        ProfileManager.shared.profile = titanProfile
        
        // When
        try await ExploreIntegrationService.shared.syncProfileFromTitanOS()
        
        // Then
        let exploreProfile = try await fetchExploreProfile()
        XCTAssertEqual(exploreProfile.firstName, "John")
    }
    
    func testGoalExtraction() async throws {
        // Given
        let outcome = RPMOutcome(title: "Learn Python for data science")
        
        // When
        let themes = await ExploreIntegrationService.shared.extractGoalThemes(from: [outcome])
        
        // Then
        XCTAssertTrue(themes["learning"]?.contains("Python") ?? false)
    }
}
```

### Integration Tests

```swift
class FirebaseSyncTests: XCTestCase {
    func testMatchSync() async throws {
        // Given
        let match = Match(userId: UUID(), matchedUserId: UUID(), domain: .love)
        
        // When
        try await FirebaseService.shared.syncMatch(match)
        
        // Then
        let cloudMatch = try await FirebaseService.shared.fetchMatch(match.id)
        XCTAssertEqual(cloudMatch.id, match.id)
    }
}
```

### End-to-End Tests

```swift
class ExploreE2ETests: XCTestCase {
    func testFullMatchingFlow() async throws {
        // 1. Create two users
        let user1 = try await createTestUser()
        let user2 = try await createTestUser()
        
        // 2. Generate match
        let match = try await MatchingEngine.shared.generateMatch(user1, user2, .love)
        
        // 3. Both users like
        try await match.like(by: user1.id)
        try await match.like(by: user2.id)
        
        // 4. Verify connection created
        let connection = try await fetchConnection(user1.id, user2.id)
        XCTAssertNotNil(connection)
        
        // 5. Send message
        try await sendMessage(from: user1.id, to: user2.id, content: "Hi!")
        
        // 6. Verify message received
        let messages = try await fetchMessages(connection.id)
        XCTAssertEqual(messages.count, 1)
    }
}
```

---

## Monitoring & Observability

### Metrics to Track

```swift
// Performance
- Match generation time (target: <1s)
- API response time (target: <200ms)
- Message delivery time (target: <500ms)
- Photo upload time (target: <3s)

// Engagement
- Daily active users
- Matches viewed per day
- Match action rate (like/pass)
- Message response rate
- Connection rate

// Quality
- Match satisfaction score
- Connection quality score
- Report rate
- Churn rate

// Technical
- Sync success rate
- Error rate
- Crash rate
- API uptime
```

### Logging

```swift
class ExploreLogger {
    static func logMatchGeneration(userId: UUID, duration: TimeInterval, matchCount: Int) {
        Logger.info(
            "Match generation completed",
            category: .explore,
            metadata: [
                "userId": userId.uuidString,
                "duration": "\(duration)s",
                "matchCount": "\(matchCount)"
            ]
        )
    }
    
    static func logError(_ error: Error, context: String) {
        Logger.error(
            "Explore error: \(error.localizedDescription)",
            category: .explore,
            metadata: ["context": context]
        )
    }
}
```

---

## Security Considerations

### Authentication

```swift
// JWT-based authentication
class AuthService {
    func authenticate(userId: UUID) async throws -> String {
        let payload = [
            "sub": userId.uuidString,
            "exp": Date().addingTimeInterval(3600).timeIntervalSince1970,
            "iat": Date().timeIntervalSince1970
        ]
        
        return try JWT.encode(payload, algorithm: .hs256(SecretsManager.shared.jwtSecret))
    }
}
```

### Data Encryption

```swift
// End-to-end encryption for messages
class EncryptionService {
    func encryptMessage(_ message: String, for recipientId: UUID) throws -> Data {
        let recipientPublicKey = try fetchPublicKey(recipientId)
        return try CryptoKit.encrypt(message, with: recipientPublicKey)
    }
    
    func decryptMessage(_ data: Data, from senderId: UUID) throws -> String {
        let privateKey = try fetchPrivateKey()
        return try CryptoKit.decrypt(data, with: privateKey)
    }
}
```

### Rate Limiting

```swift
class RateLimiter {
    private var requestCounts: [UUID: Int] = [:]
    private let maxRequests = 100
    private let timeWindow: TimeInterval = 60
    
    func checkLimit(for userId: UUID) throws {
        let count = requestCounts[userId] ?? 0
        
        if count >= maxRequests {
            throw ExploreError.rateLimitExceeded
        }
        
        requestCounts[userId] = count + 1
        
        // Reset after time window
        DispatchQueue.main.asyncAfter(deadline: .now() + timeWindow) {
            self.requestCounts[userId] = max(0, (self.requestCounts[userId] ?? 0) - 1)
        }
    }
}
```

---

## Conclusion

This integration document provides a comprehensive blueprint for connecting Titan Explore & More with TitanOS and external services. Key takeaways:

1. **Seamless Integration:** Explore feels native to TitanOS
2. **Privacy-First:** User controls all data sharing
3. **Modular Design:** Clean separation of concerns
4. **Scalable Architecture:** Clear path from local to cloud
5. **AI-Powered:** Leverages existing AI infrastructure

**Next Steps:**
1. Implement `ExploreIntegrationService`
2. Build Firebase sync layer
3. Create API endpoints
4. Add WebSocket support
5. Implement end-to-end encryption

---

*Document prepared by: Engineering Team*  
*Last updated: January 17, 2026*  
*Next review: February 1, 2026*
