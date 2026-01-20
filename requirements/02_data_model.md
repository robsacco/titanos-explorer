# Titan Explore & More: Data Model Document

**Version:** 1.0  
**Date:** January 17, 2026  
**Status:** Planning  
**Owner:** Engineering Team

---

## Overview

This document defines the complete data model for **Titan Explore & More**, including all entities, relationships, and data flows. The system is designed to start with local-first architecture (SwiftData on iOS) with a clear migration path to Google Cloud backend services.

---

## Architecture Principles

### 1. Local-First Design
- All user data stored locally in SwiftData initially
- Offline-first functionality
- Sync to cloud when available
- User owns their data

### 2. Privacy by Design
- Granular privacy controls
- Encrypted sensitive data
- Minimal data sharing
- User consent required

### 3. Scalable Schema
- Designed for eventual cloud migration
- Normalized data structure
- Efficient indexing
- Support for sharding

### 4. AI-Ready Data
- Structured for ML/AI processing
- Embeddings for semantic search
- Feature vectors for matching
- Training data collection (opt-in)

---

## Core Entities

### 1. User Profile (Extended)

```swift
@Model
class ExploreUserProfile {
    // Identity (from existing TitanOS)
    @Attribute(.unique) var id: UUID
    var userId: UUID // Link to TitanOS User
    
    // Basic Info
    var firstName: String
    var lastName: String?
    var displayName: String
    var bio: String?
    var birthDate: Date
    var gender: Gender
    var pronouns: String?
    
    // Location
    var city: String
    var state: String?
    var country: String
    var latitude: Double?
    var longitude: Double?
    var locationAccuracy: LocationAccuracy // Exact, City, State
    
    // Photos
    @Relationship(deleteRule: .cascade) var photos: [UserPhoto] = []
    var primaryPhotoId: UUID?
    
    // Verification
    var isPhotoVerified: Bool = false
    var isPhoneVerified: Bool = false
    var isIDVerified: Bool = false
    var verificationDate: Date?
    
    // Psychometrics
    @Relationship(deleteRule: .cascade) var psychometrics: PsychometricProfile?
    
    // Preferences
    @Relationship(deleteRule: .cascade) var preferences: UserPreferences?
    
    // Privacy
    @Relationship(deleteRule: .cascade) var privacySettings: PrivacySettings?
    
    // Activity
    var lastActive: Date
    var createdAt: Date
    var updatedAt: Date
    
    // Status
    var accountStatus: AccountStatus // Active, Paused, Suspended, Deleted
    var subscriptionTier: SubscriptionTier // Free, Premium, PremiumPlus
    
    // Computed Properties
    var age: Int {
        Calendar.current.dateComponents([.year], from: birthDate, to: Date()).year ?? 0
    }
}

enum Gender: String, Codable {
    case male, female, nonBinary, other, preferNotToSay
}

enum LocationAccuracy: String, Codable {
    case exact, city, state, hidden
}

enum AccountStatus: String, Codable {
    case active, paused, suspended, deleted
}

enum SubscriptionTier: String, Codable {
    case free, premium, premiumPlus
}
```

### 2. User Photo

```swift
@Model
class UserPhoto {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var imageData: Data? // Stored locally
    var cloudURL: String? // Cloud storage URL
    var thumbnailData: Data?
    
    var order: Int // Display order
    var caption: String?
    
    var isVerified: Bool = false // Used for photo verification
    var isPrimary: Bool = false
    
    var uploadedAt: Date
    var updatedAt: Date
}
```

### 3. Psychometric Profile

```swift
@Model
class PsychometricProfile {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    // Big Five Personality (OCEAN)
    var openness: Double // 0-100
    var conscientiousness: Double
    var extraversion: Double
    var agreeableness: Double
    var neuroticism: Double
    
    // Enneagram
    var enneagramType: Int? // 1-9
    var enneagramWing: Int? // Wing number
    
    // Love Languages (0-100 each)
    var wordsOfAffirmation: Double
    var qualityTime: Double
    var receivingGifts: Double
    var actsOfService: Double
    var physicalTouch: Double
    
    // Attachment Style
    var attachmentStyle: AttachmentStyle?
    
    // Myers-Briggs (optional)
    var myersBriggs: String? // e.g., "INTJ"
    
    // Values (from TitanOS CoreValues)
    @Relationship var coreValues: [CoreValue] = []
    
    // Needs (from TitanOS)
    var primaryNeed: HumanNeed?
    var secondaryNeed: HumanNeed?
    
    // Assessment Metadata
    var assessmentCompletedAt: Date?
    var assessmentVersion: String?
    
    var createdAt: Date
    var updatedAt: Date
}

enum AttachmentStyle: String, Codable {
    case secure, anxious, avoidant, fearfulAvoidant
}

enum HumanNeed: String, Codable {
    case certainty, variety, significance, loveConnection, growth, contribution
}
```

### 4. User Preferences

```swift
@Model
class UserPreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    // Active Domains
    var activeDomains: [MatchDomain] = []
    
    // Love Preferences
    @Relationship(deleteRule: .cascade) var lovePreferences: LovePreferences?
    
    // Friendship Preferences
    @Relationship(deleteRule: .cascade) var friendshipPreferences: FriendshipPreferences?
    
    // Learning Preferences
    @Relationship(deleteRule: .cascade) var learningPreferences: LearningPreferences?
    
    // Activity Preferences
    @Relationship(deleteRule: .cascade) var activityPreferences: ActivityPreferences?
    
    // Event Preferences
    @Relationship(deleteRule: .cascade) var eventPreferences: EventPreferences?
    
    // General Preferences
    var maxDistance: Int = 25 // miles
    var ageRangeMin: Int?
    var ageRangeMax: Int?
    
    var createdAt: Date
    var updatedAt: Date
}

enum MatchDomain: String, Codable, CaseIterable {
    case love, friendship, learning, events, activities, entertainment, shopping, growth
}
```

### 5. Domain-Specific Preferences

```swift
@Model
class LovePreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var lookingFor: [RelationshipType] = []
    var genderPreference: [Gender] = []
    var dealBreakers: [String] = [] // e.g., "smoking", "wants kids"
    var mustHaves: [String] = [] // e.g., "spiritual", "active"
    
    var minCompatibilityScore: Int = 70
    var prioritizeValues: Bool = true
    var prioritizeGoals: Bool = true
    
    var createdAt: Date
    var updatedAt: Date
}

enum RelationshipType: String, Codable {
    case longTerm, shortTerm, casual, friendship, figureItOut
}

@Model
class FriendshipPreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var lookingFor: [FriendshipType] = []
    var interests: [String] = []
    var activityLevel: ActivityLevel?
    
    var minCompatibilityScore: Int = 60
    
    var createdAt: Date
    var updatedAt: Date
}

enum FriendshipType: String, Codable {
    case close, casual, activity, professional
}

enum ActivityLevel: String, Codable {
    case low, moderate, high, veryHigh
}

@Model
class LearningPreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var learningGoals: [String] = [] // e.g., "Python", "Spanish", "Guitar"
    var skillLevel: [String: SkillLevel] = [:] // Topic -> Level
    var learningStyle: LearningStyle?
    var availability: [DayOfWeek] = []
    
    var minCompatibilityScore: Int = 70
    
    var createdAt: Date
    var updatedAt: Date
}

enum SkillLevel: String, Codable {
    case beginner, intermediate, advanced, expert
}

enum LearningStyle: String, Codable {
    case visual, auditory, kinesthetic, reading
}

@Model
class ActivityPreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var activities: [String] = [] // e.g., "hiking", "yoga", "climbing"
    var skillLevel: [String: SkillLevel] = [:] // Activity -> Level
    var frequency: ActivityFrequency?
    var availability: [DayOfWeek] = []
    
    var minCompatibilityScore: Int = 60
    
    var createdAt: Date
    var updatedAt: Date
}

enum ActivityFrequency: String, Codable {
    case daily, weekly, biweekly, monthly, occasional
}

@Model
class EventPreferences {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var eventTypes: [EventType] = []
    var maxTicketPrice: Double?
    var preferredDays: [DayOfWeek] = []
    var preferredTimes: [TimeOfDay] = []
    
    var createdAt: Date
    var updatedAt: Date
}

enum EventType: String, Codable {
    case concert, conference, meetup, workshop, sports, art, food, networking, spiritual, wellness
}

enum TimeOfDay: String, Codable {
    case morning, afternoon, evening, night
}

enum DayOfWeek: String, Codable, CaseIterable {
    case monday, tuesday, wednesday, thursday, friday, saturday, sunday
}
```

### 6. Privacy Settings

```swift
@Model
class PrivacySettings {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    // Visibility
    var profileVisibility: ProfileVisibility = .public
    var showOnlineStatus: Bool = true
    var showLastActive: Bool = true
    var showLocation: Bool = true
    var locationAccuracy: LocationAccuracy = .city
    
    // Data Sharing (per domain)
    var sharePsychometrics: [MatchDomain: Bool] = [:]
    var shareGoals: [MatchDomain: Bool] = [:]
    var shareSchedule: [MatchDomain: Bool] = [:]
    var shareInterests: [MatchDomain: Bool] = [:]
    
    // Matching
    var allowAIIntroductions: Bool = true
    var allowGroupMatching: Bool = true
    var allowEventMatching: Bool = true
    
    // Communication
    var allowMessages: MessagePermission = .matchesOnly
    var allowVideoCalls: Bool = true
    var allowPhoneNumber: Bool = false
    
    // Blocked Users
    var blockedUserIds: [UUID] = []
    
    var createdAt: Date
    var updatedAt: Date
}

enum ProfileVisibility: String, Codable {
    case `public`, connectionsOnly, invisible
}

enum MessagePermission: String, Codable {
    case anyone, matchesOnly, connectionsOnly, none
}
```

### 7. Match

```swift
@Model
class Match {
    @Attribute(.unique) var id: UUID
    
    var userId: UUID // The user who sees this match
    var matchedUserId: UUID // The matched user
    
    var domain: MatchDomain
    var compatibilityScore: Double // 0-100
    var compatibilityFactors: [CompatibilityFactor] = []
    
    var status: MatchStatus = .pending
    var userAction: MatchAction? // Like, Pass, SuperLike
    var matchedUserAction: MatchAction?
    
    var matchedAt: Date? // When both liked
    var expiresAt: Date? // Matches expire after 30 days if no action
    
    var createdAt: Date
    var updatedAt: Date
    
    // Computed
    var isMutual: Bool {
        userAction == .like && matchedUserAction == .like
    }
}

enum MatchStatus: String, Codable {
    case pending, mutual, expired, unmatched
}

enum MatchAction: String, Codable {
    case like, pass, superLike
}

@Model
class CompatibilityFactor {
    @Attribute(.unique) var id: UUID
    var matchId: UUID
    
    var category: FactorCategory
    var description: String
    var weight: Double // 0-1
    var score: Double // 0-100
    
    var createdAt: Date
}

enum FactorCategory: String, Codable {
    case values, personality, goals, interests, schedule, location, lifestyle
}
```

### 8. Connection

```swift
@Model
class Connection {
    @Attribute(.unique) var id: UUID
    
    var userId: UUID
    var connectedUserId: UUID
    
    var domain: MatchDomain
    var originMatchId: UUID? // The match that created this connection
    
    var status: ConnectionStatus = .active
    var connectionType: ConnectionType
    
    var lastInteractionAt: Date
    var createdAt: Date
    var updatedAt: Date
    
    // Metadata
    var notes: String? // Private notes about connection
    var tags: [String] = [] // User-defined tags
    
    // Quality Metrics
    var messageCount: Int = 0
    var meetupCount: Int = 0
    var qualityScore: Double? // 0-100, user-rated
}

enum ConnectionStatus: String, Codable {
    case active, paused, ended, blocked
}

enum ConnectionType: String, Codable {
    case romantic, friend, learningPartner, activityPartner, mentor, mentee, mastermind
}
```

### 9. Message

```swift
@Model
class Message {
    @Attribute(.unique) var id: UUID
    
    var connectionId: UUID
    var senderId: UUID
    var recipientId: UUID
    
    var content: String
    var contentType: MessageContentType = .text
    
    var isRead: Bool = false
    var readAt: Date?
    
    var sentAt: Date
    var deliveredAt: Date?
    
    // Attachments
    var attachmentURL: String?
    var attachmentType: AttachmentType?
    
    // AI-Generated
    var isAIGenerated: Bool = false
    var aiPrompt: String? // If AI-generated, what was the prompt
}

enum MessageContentType: String, Codable {
    case text, image, video, audio, location, event
}

enum AttachmentType: String, Codable {
    case image, video, audio, document
}
```

### 10. Event

```swift
@Model
class Event {
    @Attribute(.unique) var id: UUID
    
    var title: String
    var description: String
    var eventType: EventType
    
    var creatorId: UUID?
    var isUserCreated: Bool = false
    
    // Location
    var venueName: String?
    var address: String?
    var city: String
    var state: String?
    var latitude: Double?
    var longitude: Double?
    
    // Timing
    var startTime: Date
    var endTime: Date
    var timezone: String
    
    // Capacity
    var maxAttendees: Int?
    var currentAttendees: Int = 0
    
    // Pricing
    var isFree: Bool = true
    var ticketPrice: Double?
    var ticketURL: String?
    
    // Matching
    var recommendedForUserIds: [UUID] = []
    var attendingUserIds: [UUID] = []
    
    // Media
    var imageURL: String?
    var externalURL: String?
    
    var createdAt: Date
    var updatedAt: Date
}
```

### 11. Learning Circle

```swift
@Model
class LearningCircle {
    @Attribute(.unique) var id: UUID
    
    var name: String
    var description: String
    var topic: String // e.g., "Python Programming"
    var skillLevel: SkillLevel
    
    var creatorId: UUID
    var memberIds: [UUID] = []
    var maxMembers: Int = 8
    
    // Goals
    var sharedGoal: String?
    var milestones: [Milestone] = []
    
    // Schedule
    var meetingFrequency: MeetingFrequency
    var preferredDays: [DayOfWeek] = []
    var preferredTime: TimeOfDay?
    
    // Resources
    var resources: [Resource] = []
    
    // Status
    var status: GroupStatus = .active
    
    var createdAt: Date
    var updatedAt: Date
}

enum MeetingFrequency: String, Codable {
    case daily, weekly, biweekly, monthly
}

enum GroupStatus: String, Codable {
    case active, paused, completed, disbanded
}

@Model
class Milestone {
    @Attribute(.unique) var id: UUID
    var learningCircleId: UUID
    
    var title: String
    var description: String?
    var dueDate: Date?
    var isCompleted: Bool = false
    var completedAt: Date?
    
    var createdAt: Date
}

@Model
class Resource {
    @Attribute(.unique) var id: UUID
    var learningCircleId: UUID
    
    var title: String
    var url: String?
    var resourceType: ResourceType
    var addedBy: UUID
    
    var createdAt: Date
}

enum ResourceType: String, Codable {
    case article, video, book, course, tool, other
}
```

### 12. Activity Group

```swift
@Model
class ActivityGroup {
    @Attribute(.unique) var id: UUID
    
    var name: String
    var description: String
    var activity: String // e.g., "Hiking", "Yoga"
    var skillLevel: SkillLevel
    
    var creatorId: UUID
    var memberIds: [UUID] = []
    var maxMembers: Int = 20
    
    // Schedule
    var isRecurring: Bool = false
    var recurrencePattern: RecurrencePattern?
    var nextSessionDate: Date?
    
    // Location
    var defaultLocation: String?
    var latitude: Double?
    var longitude: Double?
    
    // Safety
    var emergencyContact: String?
    var safetyGuidelines: String?
    
    // Status
    var status: GroupStatus = .active
    
    var createdAt: Date
    var updatedAt: Date
}

enum RecurrencePattern: String, Codable {
    case daily, weekly, biweekly, monthly
}
```

### 13. Mastermind Group

```swift
@Model
class MastermindGroup {
    @Attribute(.unique) var id: UUID
    
    var name: String
    var description: String
    var focus: String // e.g., "Entrepreneurship", "Career Growth"
    
    var facilitatorId: UUID
    var memberIds: [UUID] = []
    var maxMembers: Int = 10
    var minMembers: Int = 5
    
    // Structure
    var meetingDuration: Int // minutes
    var meetingFrequency: MeetingFrequency
    var programDuration: Int? // weeks
    
    // Format
    var meetingFormat: MastermindFormat
    var hasHotSeats: Bool = true
    var hasGoalReviews: Bool = true
    
    // Goals
    var sharedGoals: [String] = []
    var individualGoals: [UUID: String] = [:] // UserId -> Goal
    
    // Metrics
    var totalMeetings: Int = 0
    var successMetrics: [String: Double] = [:]
    
    // Status
    var status: GroupStatus = .active
    var startDate: Date?
    var endDate: Date?
    
    var createdAt: Date
    var updatedAt: Date
}

enum MastermindFormat: String, Codable {
    case structured, freeform, hybrid
}
```

### 14. Mentorship

```swift
@Model
class Mentorship {
    @Attribute(.unique) var id: UUID
    
    var mentorId: UUID
    var menteeId: UUID
    
    var domain: String // e.g., "Product Management", "Entrepreneurship"
    var goals: [String] = []
    
    // Program
    var programDuration: Int? // weeks
    var meetingFrequency: MeetingFrequency
    var startDate: Date
    var endDate: Date?
    
    // Progress
    var milestones: [Milestone] = []
    var checkIns: [CheckIn] = []
    
    // Feedback
    var mentorRating: Double?
    var menteeRating: Double?
    var testimonial: String?
    
    // Status
    var status: MentorshipStatus = .active
    
    var createdAt: Date
    var updatedAt: Date
}

enum MentorshipStatus: String, Codable {
    case active, paused, completed, cancelled
}

@Model
class CheckIn {
    @Attribute(.unique) var id: UUID
    var mentorshipId: UUID
    
    var date: Date
    var notes: String?
    var progress: String?
    var nextSteps: String?
    
    var createdAt: Date
}
```

### 15. Tribe (Community)

```swift
@Model
class Tribe {
    @Attribute(.unique) var id: UUID
    
    var name: String
    var description: String
    var tagline: String?
    
    var founderId: UUID
    var moderatorIds: [UUID] = []
    var memberIds: [UUID] = []
    
    // Identity
    var coreValues: [String] = []
    var interests: [String] = []
    var tribeType: TribeType
    
    // Settings
    var isPublic: Bool = true
    var requiresApproval: Bool = false
    var maxMembers: Int?
    
    // Engagement
    var challenges: [TribeChallenge] = []
    var events: [UUID] = [] // Event IDs
    
    // Media
    var logoURL: String?
    var bannerURL: String?
    
    // Metrics
    var totalMembers: Int = 0
    var activeMembers: Int = 0
    var engagementScore: Double = 0
    
    var createdAt: Date
    var updatedAt: Date
}

enum TribeType: String, Codable {
    case professional, hobby, spiritual, wellness, social, learning, support
}

@Model
class TribeChallenge {
    @Attribute(.unique) var id: UUID
    var tribeId: UUID
    
    var name: String
    var description: String
    var challengeType: ChallengeType
    
    var startDate: Date
    var endDate: Date
    
    var participantIds: [UUID] = []
    var completedIds: [UUID] = []
    
    var reward: String?
    
    var createdAt: Date
}

enum ChallengeType: String, Codable {
    case habit, goal, activity, learning, contribution
}
```

### 16. User Embedding (for AI Matching)

```swift
@Model
class UserEmbedding {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    // Embeddings for different aspects
    var personalityEmbedding: [Float] = [] // 768-dim vector
    var interestsEmbedding: [Float] = []
    var goalsEmbedding: [Float] = []
    var valuesEmbedding: [Float] = []
    
    // Combined embedding for quick matching
    var combinedEmbedding: [Float] = []
    
    // Metadata
    var embeddingVersion: String
    var generatedAt: Date
    var updatedAt: Date
}
```

### 17. Match Queue (for daily match generation)

```swift
@Model
class MatchQueue {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var domain: MatchDomain
    var queuedMatchIds: [UUID] = [] // Pre-computed matches
    var shownMatchIds: [UUID] = [] // Already shown today
    
    var lastRefreshedAt: Date
    var nextRefreshAt: Date
}
```

### 18. Analytics Event (for tracking and improvement)

```swift
@Model
class AnalyticsEvent {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    var eventType: AnalyticsEventType
    var eventData: [String: String] = [:]
    
    var timestamp: Date
}

enum AnalyticsEventType: String, Codable {
    case matchViewed, matchLiked, matchPassed, matchSuperLiked
    case connectionCreated, messageSent, messageReceived
    case eventRSVP, eventAttended
    case groupJoined, groupLeft
    case profileUpdated, photoAdded
    case subscriptionUpgraded, subscriptionCancelled
}
```

---

## Relationships Diagram

```
User Profile
    ├── Photos (1:many)
    ├── Psychometric Profile (1:1)
    ├── Preferences (1:1)
    │   ├── Love Preferences (1:1)
    │   ├── Friendship Preferences (1:1)
    │   ├── Learning Preferences (1:1)
    │   ├── Activity Preferences (1:1)
    │   └── Event Preferences (1:1)
    ├── Privacy Settings (1:1)
    ├── Matches (1:many)
    │   └── Compatibility Factors (1:many)
    ├── Connections (1:many)
    │   └── Messages (1:many)
    ├── Learning Circles (many:many)
    │   ├── Milestones (1:many)
    │   └── Resources (1:many)
    ├── Activity Groups (many:many)
    ├── Mastermind Groups (many:many)
    ├── Mentorships (1:many as mentor or mentee)
    │   ├── Milestones (1:many)
    │   └── Check-Ins (1:many)
    ├── Tribes (many:many)
    │   └── Challenges (1:many)
    ├── User Embedding (1:1)
    ├── Match Queue (1:many per domain)
    └── Analytics Events (1:many)

Events (standalone)
    └── Attendees (many:many with Users)
```

---

## Data Flow

### 1. User Onboarding Flow

```
1. User creates TitanOS account
    ↓
2. Invited to complete Explore profile
    ↓
3. Psychometric assessment (10 min)
    ↓
4. Select active domains (Love, Learning, etc.)
    ↓
5. Set preferences per domain
    ↓
6. Upload photos
    ↓
7. Review privacy settings
    ↓
8. Generate initial embeddings
    ↓
9. Compute initial match queue
    ↓
10. Show first matches
```

### 2. Daily Match Generation Flow

```
Scheduled Task (runs at 6 AM user local time)
    ↓
1. Fetch user profile, preferences, embeddings
    ↓
2. Query potential matches (filters: location, age, domain)
    ↓
3. Calculate compatibility scores for each
    ↓
4. Rank by score and diversity
    ↓
5. Select top 10 per domain
    ↓
6. Store in MatchQueue
    ↓
7. Send notification "New matches available"
    ↓
8. User opens app and views matches
```

### 3. Matching Algorithm Flow

```
Input: User A, User B, Domain

1. Load profiles and preferences
    ↓
2. Check basic filters (age, location, gender)
    ↓
3. Calculate compatibility factors:
    a. Values alignment (cosine similarity of embeddings)
    b. Personality compatibility (Big 5 distance)
    c. Goals alignment (semantic similarity)
    d. Interests overlap (Jaccard similarity)
    e. Schedule compatibility (availability overlap)
    ↓
4. Weight factors based on domain
    ↓
5. Compute weighted score (0-100)
    ↓
6. Generate explanation (top 3 factors)
    ↓
Output: Compatibility Score + Factors
```

### 4. Connection Creation Flow

```
1. User A likes User B
    ↓
2. Store MatchAction in Match entity
    ↓
3. Check if User B already liked User A
    ↓
4. If mutual:
    a. Update Match status to "mutual"
    b. Create Connection entity
    c. Send notifications to both users
    d. Enable messaging
    ↓
5. If not mutual:
    a. Wait for User B's action
    b. Match expires after 30 days if no action
```

### 5. Message Delivery Flow

```
1. User A sends message to User B
    ↓
2. Create Message entity locally
    ↓
3. Encrypt message content
    ↓
4. Sync to cloud (if online)
    ↓
5. Cloud delivers to User B's device
    ↓
6. User B's device decrypts and stores locally
    ↓
7. Send read receipt when User B opens message
    ↓
8. Update Message.isRead and Message.readAt
```

---

## Indexing Strategy

### Primary Indexes (for performance)

```swift
// User Profile
- id (unique)
- userId (for linking to TitanOS)
- city + state (for location-based queries)
- lastActive (for active user queries)

// Match
- userId + domain (for user's matches per domain)
- matchedUserId + domain (for reverse lookups)
- status (for filtering active matches)
- expiresAt (for cleanup jobs)

// Connection
- userId + connectedUserId (for checking existing connections)
- userId + status (for active connections)
- lastInteractionAt (for sorting)

// Message
- connectionId + sentAt (for chronological ordering)
- recipientId + isRead (for unread count)

// Event
- city + startTime (for location-based event discovery)
- eventType + startTime (for category browsing)

// Learning Circle / Activity Group / Mastermind
- memberIds (for user's groups)
- status (for active groups)
```

### Composite Indexes

```swift
// For complex queries
- (userId, domain, status) on Match
- (city, eventType, startTime) on Event
- (topic, skillLevel, status) on LearningCircle
```

---

## Data Migration Strategy

### Phase 1: Local-First (Months 1-6)

**Storage:** SwiftData (local SQLite)  
**Sync:** None (single-device)  
**Backup:** iCloud backup (automatic)

**Pros:**
- Fast development
- No backend infrastructure
- Privacy-first (data never leaves device)
- Offline-first

**Cons:**
- No cross-device sync
- Limited scalability
- No real-time updates
- Manual backup only

### Phase 2: Hybrid (Months 7-12)

**Storage:** SwiftData (local) + Firebase Firestore (cloud)  
**Sync:** Bi-directional sync on app launch and background  
**Backup:** Cloud backup

**Architecture:**
```
iOS App (SwiftData)
    ↕ (Sync)
Firebase Firestore
    ↕ (Replication)
Google Cloud SQL (PostgreSQL)
```

**Migration Steps:**
1. Add Firebase SDK to iOS app
2. Create Firestore schema matching SwiftData
3. Implement sync service (local → cloud, cloud → local)
4. Handle conflict resolution (last-write-wins)
5. Migrate existing users (one-time sync)

### Phase 3: Cloud-Native (Year 2+)

**Storage:** Google Cloud SQL (PostgreSQL) + Cloud Firestore  
**Sync:** Real-time via WebSockets  
**Backup:** Automated daily backups

**Architecture:**
```
iOS App (SwiftData cache)
    ↕ (WebSocket)
API Gateway (Cloud Run)
    ↕
Application Server (Go/Node.js)
    ↕
PostgreSQL (primary) + Firestore (real-time)
    ↕
Cloud Storage (media)
```

**Migration Steps:**
1. Build REST API on Google Cloud Run
2. Migrate Firestore data to PostgreSQL
3. Implement WebSocket server for real-time sync
4. Update iOS app to use API instead of direct Firestore
5. Migrate users in batches (by city)

---

## Data Privacy & Security

### Encryption

**At Rest:**
- SwiftData: iOS file encryption (automatic)
- Cloud: Google Cloud encryption (AES-256)

**In Transit:**
- TLS 1.3 for all API calls
- End-to-end encryption for messages (Signal Protocol)

### Data Minimization

**Principle:** Collect only what's necessary

**Shared Data by Domain:**
- Love: Name, age, photos, bio, values, personality
- Friendship: Name, age, photo, interests, availability
- Learning: Name, learning goals, skill level, availability
- Activities: Name, activity preferences, skill level, location

**Never Shared Without Consent:**
- Exact location (only city/state)
- Full psychometric scores (only compatibility factors)
- TitanOS tasks/goals (only high-level themes)
- Health data (only general readiness if opted in)

### User Control

**Data Portability:**
- Export all data as JSON
- Download all photos
- Delete account and all data

**Granular Privacy:**
- Control visibility per domain
- Hide specific profile fields
- Pause/unpause account
- Invisible mode (Premium)

---

## Performance Considerations

### Caching Strategy

**Local Cache (SwiftData):**
- User profile: Permanent
- Matches: 30 days
- Messages: Permanent
- Events: 90 days future
- Groups: While member

**Cloud Cache (Redis):**
- Match queue: 24 hours
- Compatibility scores: 7 days
- User embeddings: 30 days
- Event recommendations: 24 hours

### Query Optimization

**Expensive Queries:**
1. Match generation (compute compatibility for 1000s of users)
2. Event recommendations (filter + rank + personalize)
3. Embedding generation (ML inference)

**Optimizations:**
1. Pre-compute match queues daily (batch job)
2. Cache compatibility scores (invalidate on profile update)
3. Use approximate nearest neighbor for embedding search
4. Paginate results (10-20 per page)
5. Lazy load images and media

### Scalability Targets

**Year 1:**
- 100K users
- 1M matches per day
- 10M messages per day
- 10K events per month

**Year 3:**
- 10M users
- 100M matches per day
- 1B messages per day
- 100K events per month

---

## Conclusion

This data model provides a solid foundation for Titan Explore & More, balancing:
- **Richness:** Comprehensive data for intelligent matching
- **Privacy:** Granular controls and encryption
- **Performance:** Efficient indexing and caching
- **Scalability:** Clear migration path to cloud
- **Flexibility:** Extensible for new domains and features

**Next Steps:**
1. Review and approve data model
2. Implement SwiftData entities
3. Build matching algorithm
4. Create sync service (Phase 2)
5. Design API schema (Phase 3)

---

*Document prepared by: Engineering Team*  
*Last updated: January 17, 2026*  
*Next review: February 1, 2026*
