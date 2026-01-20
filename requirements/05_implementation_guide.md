# Titan Explore & More: Implementation Guide

**Version:** 1.0  
**Date:** January 17, 2026  
**Status:** Planning  
**Owner:** Engineering Team

---

## Overview

This document provides a step-by-step implementation guide for building **Titan Explore & More**, from initial setup through production deployment. The implementation is divided into 3 phases over 12 months.

---

## Phase 1: Foundation (Months 1-4)

### Month 1: Setup & Core Data Models

#### Week 1-2: Project Setup

**Tasks:**
1. Create new Swift Package: `TitanExplore`
2. Set up SwiftData schema
3. Configure dependencies
4. Set up development environment

**Implementation:**

```bash
# Create package structure
cd /Users/robsacco/Projects/titanos-refactored/titanos/packages
mkdir TitanExplore
cd TitanExplore

# Create Package.swift
cat > Package.swift << 'EOF'
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "TitanExplore",
    platforms: [.iOS(.v17)],
    products: [
        .library(
            name: "TitanExplore",
            targets: ["TitanExplore"]
        ),
    ],
    dependencies: [
        .package(path: "../TitanOSCore"),
        .package(path: "../TitanAI"),
        .package(url: "https://github.com/firebase/firebase-ios-sdk", from: "10.0.0")
    ],
    targets: [
        .target(
            name: "TitanExplore",
            dependencies: [
                "TitanOSCore",
                "TitanAI",
                .product(name: "FirebaseFirestore", package: "firebase-ios-sdk"),
                .product(name: "FirebaseStorage", package: "firebase-ios-sdk")
            ]
        ),
        .testTarget(
            name: "TitanExploreTests",
            dependencies: ["TitanExplore"]
        ),
    ]
)
EOF

# Create directory structure
mkdir -p Sources/TitanExplore/{Models,Services,Views,ViewModels,Utilities}
mkdir -p Tests/TitanExploreTests
```

#### Week 3-4: Core Data Models

**File:** `Sources/TitanExplore/Models/ExploreModels.swift`

```swift
import Foundation
import SwiftData

// MARK: - User Profile

@Model
class ExploreUserProfile {
    @Attribute(.unique) var id: UUID
    var userId: UUID // Link to TitanOS User
    
    var firstName: String
    var lastName: String?
    var displayName: String
    var bio: String?
    var birthDate: Date
    var gender: Gender
    var pronouns: String?
    
    var city: String
    var state: String?
    var country: String
    var latitude: Double?
    var longitude: Double?
    var locationAccuracy: LocationAccuracy
    
    @Relationship(deleteRule: .cascade) var photos: [UserPhoto] = []
    var primaryPhotoId: UUID?
    
    var isPhotoVerified: Bool = false
    var isPhoneVerified: Bool = false
    
    @Relationship(deleteRule: .cascade) var psychometrics: PsychometricProfile?
    @Relationship(deleteRule: .cascade) var preferences: UserPreferences?
    @Relationship(deleteRule: .cascade) var privacySettings: PrivacySettings?
    
    var lastActive: Date
    var createdAt: Date
    var updatedAt: Date
    
    var accountStatus: AccountStatus = .active
    var subscriptionTier: SubscriptionTier = .free
    
    init(
        id: UUID = UUID(),
        userId: UUID,
        firstName: String,
        birthDate: Date,
        gender: Gender,
        city: String,
        country: String
    ) {
        self.id = id
        self.userId = userId
        self.firstName = firstName
        self.displayName = firstName
        self.birthDate = birthDate
        self.gender = gender
        self.city = city
        self.country = country
        self.locationAccuracy = .city
        self.lastActive = Date()
        self.createdAt = Date()
        self.updatedAt = Date()
    }
}

// MARK: - Supporting Types

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

// MARK: - Psychometrics

@Model
class PsychometricProfile {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    
    // Big Five
    var openness: Double = 50
    var conscientiousness: Double = 50
    var extraversion: Double = 50
    var agreeableness: Double = 50
    var neuroticism: Double = 50
    
    // Love Languages
    var wordsOfAffirmation: Double = 20
    var qualityTime: Double = 20
    var receivingGifts: Double = 20
    var actsOfService: Double = 20
    var physicalTouch: Double = 20
    
    var assessmentCompletedAt: Date?
    var createdAt: Date
    var updatedAt: Date
    
    init(userId: UUID) {
        self.id = UUID()
        self.userId = userId
        self.createdAt = Date()
        self.updatedAt = Date()
    }
}

// MARK: - Match

@Model
class Match {
    @Attribute(.unique) var id: UUID
    var userId: UUID
    var matchedUserId: UUID
    
    var domain: MatchDomain
    var compatibilityScore: Double
    @Relationship(deleteRule: .cascade) var compatibilityFactors: [CompatibilityFactor] = []
    
    var status: MatchStatus = .pending
    var userAction: MatchAction?
    var matchedUserAction: MatchAction?
    
    var matchedAt: Date?
    var expiresAt: Date?
    
    var createdAt: Date
    var updatedAt: Date
    
    init(
        userId: UUID,
        matchedUserId: UUID,
        domain: MatchDomain,
        compatibilityScore: Double
    ) {
        self.id = UUID()
        self.userId = userId
        self.matchedUserId = matchedUserId
        self.domain = domain
        self.compatibilityScore = compatibilityScore
        self.expiresAt = Date().addingTimeInterval(30 * 24 * 60 * 60) // 30 days
        self.createdAt = Date()
        self.updatedAt = Date()
    }
}

enum MatchDomain: String, Codable, CaseIterable {
    case love, friendship, learning, events, activities, entertainment, shopping, growth
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
    var weight: Double
    var score: Double
    
    var createdAt: Date
    
    init(matchId: UUID, category: FactorCategory, description: String, weight: Double, score: Double) {
        self.id = UUID()
        self.matchId = matchId
        self.category = category
        self.description = description
        self.weight = weight
        self.score = score
        self.createdAt = Date()
    }
}

enum FactorCategory: String, Codable {
    case values, personality, goals, interests, schedule, location, lifestyle
}
```

### Month 2: Matching Algorithm

#### Week 1-2: Compatibility Calculation

**File:** `Sources/TitanExplore/Services/MatchingEngine.swift`

```swift
import Foundation

actor MatchingEngine {
    static let shared = MatchingEngine()
    
    // MARK: - Public API
    
    func generateMatches(
        for userId: UUID,
        domain: MatchDomain,
        limit: Int = 10
    ) async throws -> [Match] {
        let userProfile = try await fetchProfile(userId)
        let candidates = try await fetchCandidates(for: userProfile, domain: domain)
        
        var matches: [Match] = []
        
        for candidate in candidates {
            let score = try await calculateCompatibility(
                user: userProfile,
                candidate: candidate,
                domain: domain
            )
            
            if score >= getMinimumScore(for: domain) {
                let match = Match(
                    userId: userId,
                    matchedUserId: candidate.id,
                    domain: domain,
                    compatibilityScore: score
                )
                
                // Add compatibility factors
                let factors = try await generateCompatibilityFactors(
                    user: userProfile,
                    candidate: candidate,
                    domain: domain
                )
                match.compatibilityFactors = factors
                
                matches.append(match)
            }
        }
        
        // Sort by score and return top N
        return matches
            .sorted { $0.compatibilityScore > $1.compatibilityScore }
            .prefix(limit)
            .map { $0 }
    }
    
    // MARK: - Compatibility Calculation
    
    private func calculateCompatibility(
        user: ExploreUserProfile,
        candidate: ExploreUserProfile,
        domain: MatchDomain
    ) async throws -> Double {
        let weights = getWeights(for: domain)
        
        var totalScore: Double = 0
        var totalWeight: Double = 0
        
        // 1. Values Alignment
        if let valuesScore = try await calculateValuesAlignment(user, candidate) {
            totalScore += valuesScore * weights.values
            totalWeight += weights.values
        }
        
        // 2. Personality Compatibility
        if let personalityScore = calculatePersonalityCompatibility(user, candidate) {
            totalScore += personalityScore * weights.personality
            totalWeight += weights.personality
        }
        
        // 3. Interests Overlap
        if let interestsScore = calculateInterestsOverlap(user, candidate, domain) {
            totalScore += interestsScore * weights.interests
            totalWeight += weights.interests
        }
        
        // 4. Goals Alignment
        if let goalsScore = try await calculateGoalsAlignment(user, candidate, domain) {
            totalScore += goalsScore * weights.goals
            totalWeight += weights.goals
        }
        
        // 5. Location Proximity
        if let locationScore = calculateLocationScore(user, candidate) {
            totalScore += locationScore * weights.location
            totalWeight += weights.location
        }
        
        // Normalize to 0-100
        return totalWeight > 0 ? (totalScore / totalWeight) * 100 : 0
    }
    
    // MARK: - Individual Scoring Functions
    
    private func calculateValuesAlignment(
        _ user: ExploreUserProfile,
        _ candidate: ExploreUserProfile
    ) async throws -> Double? {
        guard let userValues = user.psychometrics?.coreValues,
              let candidateValues = candidate.psychometrics?.coreValues else {
            return nil
        }
        
        // Calculate Jaccard similarity
        let intersection = Set(userValues).intersection(Set(candidateValues))
        let union = Set(userValues).union(Set(candidateValues))
        
        return union.isEmpty ? 0 : Double(intersection.count) / Double(union.count)
    }
    
    private func calculatePersonalityCompatibility(
        _ user: ExploreUserProfile,
        _ candidate: ExploreUserProfile
    ) -> Double? {
        guard let userPsych = user.psychometrics,
              let candidatePsych = candidate.psychometrics else {
            return nil
        }
        
        // Calculate Euclidean distance in Big 5 space
        let distance = sqrt(
            pow(userPsych.openness - candidatePsych.openness, 2) +
            pow(userPsych.conscientiousness - candidatePsych.conscientiousness, 2) +
            pow(userPsych.extraversion - candidatePsych.extraversion, 2) +
            pow(userPsych.agreeableness - candidatePsych.agreeableness, 2) +
            pow(userPsych.neuroticism - candidatePsych.neuroticism, 2)
        )
        
        // Normalize: max distance is sqrt(5 * 100^2) = 223.6
        // Convert to similarity (0 = different, 1 = same)
        return 1 - (distance / 223.6)
    }
    
    private func calculateInterestsOverlap(
        _ user: ExploreUserProfile,
        _ candidate: ExploreUserProfile,
        _ domain: MatchDomain
    ) -> Double? {
        // Get interests based on domain
        let userInterests = getInterests(for: user, domain: domain)
        let candidateInterests = getInterests(for: candidate, domain: domain)
        
        guard !userInterests.isEmpty && !candidateInterests.isEmpty else {
            return nil
        }
        
        // Jaccard similarity
        let intersection = Set(userInterests).intersection(Set(candidateInterests))
        let union = Set(userInterests).union(Set(candidateInterests))
        
        return union.isEmpty ? 0 : Double(intersection.count) / Double(union.count)
    }
    
    private func calculateGoalsAlignment(
        _ user: ExploreUserProfile,
        _ candidate: ExploreUserProfile,
        _ domain: MatchDomain
    ) async throws -> Double? {
        // Extract goals from TitanOS integration
        let userGoals = try await ExploreIntegrationService.shared.extractGoalThemes(for: user.userId)
        let candidateGoals = try await ExploreIntegrationService.shared.extractGoalThemes(for: candidate.userId)
        
        // Get domain-specific goals
        let userDomainGoals = userGoals[domain.rawValue] ?? []
        let candidateDomainGoals = candidateGoals[domain.rawValue] ?? []
        
        guard !userDomainGoals.isEmpty && !candidateDomainGoals.isEmpty else {
            return nil
        }
        
        // Use AI to calculate semantic similarity
        return try await AIOrchestrator.shared.calculateSemanticSimilarity(
            userDomainGoals,
            candidateDomainGoals
        )
    }
    
    private func calculateLocationScore(
        _ user: ExploreUserProfile,
        _ candidate: ExploreUserProfile
    ) -> Double? {
        guard let userLat = user.latitude,
              let userLon = user.longitude,
              let candidateLat = candidate.latitude,
              let candidateLon = candidate.longitude else {
            return nil
        }
        
        let distance = haversineDistance(
            lat1: userLat, lon1: userLon,
            lat2: candidateLat, lon2: candidateLon
        )
        
        // Score based on distance (closer = better)
        // 0-5 miles: 1.0
        // 5-10 miles: 0.8
        // 10-25 miles: 0.6
        // 25-50 miles: 0.4
        // 50+ miles: 0.2
        switch distance {
        case 0..<5: return 1.0
        case 5..<10: return 0.8
        case 10..<25: return 0.6
        case 25..<50: return 0.4
        default: return 0.2
        }
    }
    
    // MARK: - Helpers
    
    private func getWeights(for domain: MatchDomain) -> CompatibilityWeights {
        switch domain {
        case .love:
            return CompatibilityWeights(
                values: 0.40,
                personality: 0.25,
                interests: 0.10,
                goals: 0.20,
                location: 0.05
            )
        case .friendship:
            return CompatibilityWeights(
                values: 0.30,
                personality: 0.20,
                interests: 0.35,
                goals: 0.10,
                location: 0.05
            )
        case .learning:
            return CompatibilityWeights(
                values: 0.15,
                personality: 0.15,
                interests: 0.20,
                goals: 0.50,
                location: 0.00
            )
        default:
            return CompatibilityWeights(
                values: 0.25,
                personality: 0.25,
                interests: 0.25,
                goals: 0.20,
                location: 0.05
            )
        }
    }
    
    private func getMinimumScore(for domain: MatchDomain) -> Double {
        switch domain {
        case .love: return 70
        case .friendship: return 60
        case .learning: return 70
        default: return 60
        }
    }
    
    private func haversineDistance(lat1: Double, lon1: Double, lat2: Double, lon2: Double) -> Double {
        let R = 3959.0 // Earth radius in miles
        let dLat = (lat2 - lat1) * .pi / 180
        let dLon = (lon2 - lon1) * .pi / 180
        
        let a = sin(dLat / 2) * sin(dLat / 2) +
                cos(lat1 * .pi / 180) * cos(lat2 * .pi / 180) *
                sin(dLon / 2) * sin(dLon / 2)
        
        let c = 2 * atan2(sqrt(a), sqrt(1 - a))
        return R * c
    }
}

struct CompatibilityWeights {
    let values: Double
    let personality: Double
    let interests: Double
    let goals: Double
    let location: Double
}
```

### Month 3: UI Implementation

#### Week 1-2: Core Views

**File:** `Sources/TitanExplore/Views/ExploreHomeView.swift`

```swift
import SwiftUI
import SwiftData

struct ExploreHomeView: View {
    @Environment(\.modelContext) private var modelContext
    @StateObject private var viewModel = ExploreViewModel()
    
    var body: some View {
        NavigationView {
            ScrollView {
                VStack(spacing: 24) {
                    // Header
                    headerSection
                    
                    // Domain Cards
                    domainCardsSection
                    
                    // Recent Activity
                    recentActivitySection
                    
                    // Suggested
                    suggestedSection
                }
                .padding()
            }
            .navigationTitle("Explore")
            .navigationBarTitleDisplayMode(.large)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button(action: { viewModel.showSettings = true }) {
                        Image(systemName: "gearshape")
                    }
                }
            }
            .sheet(isPresented: $viewModel.showSettings) {
                ExploreSettingsView()
            }
        }
        .onAppear {
            Task {
                await viewModel.loadData()
            }
        }
    }
    
    private var headerSection: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(viewModel.greeting)
                .font(.title2)
                .fontWeight(.semibold)
            
            Text("You have \(viewModel.totalNewMatches) new matches today")
                .font(.subheadline)
                .foregroundColor(.secondary)
        }
        .frame(maxWidth: .infinity, alignment: .leading)
    }
    
    private var domainCardsSection: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible())
        ], spacing: 16) {
            ForEach(MatchDomain.allCases, id: \.self) { domain in
                DomainCard(
                    domain: domain,
                    matchCount: viewModel.matchCounts[domain] ?? 0
                )
                .onTapGesture {
                    viewModel.selectedDomain = domain
                }
            }
        }
        .sheet(item: $viewModel.selectedDomain) { domain in
            MatchDiscoveryView(domain: domain)
        }
    }
    
    private var recentActivitySection: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("Recent Activity")
                .font(.headline)
            
            ForEach(viewModel.recentActivity) { activity in
                ActivityRow(activity: activity)
            }
        }
    }
    
    private var suggestedSection: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("Suggested For You")
                .font(.headline)
            
            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 16) {
                    ForEach(viewModel.suggestions) { suggestion in
                        SuggestionCard(suggestion: suggestion)
                    }
                }
            }
        }
    }
}

struct DomainCard: View {
    let domain: MatchDomain
    let matchCount: Int
    
    var body: some View {
        VStack(spacing: 12) {
            Image(systemName: domain.icon)
                .font(.system(size: 32))
                .foregroundColor(domain.color)
            
            Text(domain.displayName)
                .font(.subheadline)
                .fontWeight(.medium)
            
            if matchCount > 0 {
                Text("\(matchCount)")
                    .font(.title2)
                    .fontWeight(.bold)
                    .foregroundColor(domain.color)
            }
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(Color.white)
        .cornerRadius(16)
        .shadow(color: .black.opacity(0.05), radius: 10, y: 4)
    }
}

extension MatchDomain {
    var displayName: String {
        switch self {
        case .love: return "Love"
        case .friendship: return "Friendship"
        case .learning: return "Learning"
        case .events: return "Events"
        case .activities: return "Activities"
        case .entertainment: return "Entertainment"
        case .shopping: return "Shopping"
        case .growth: return "Growth"
        }
    }
    
    var icon: String {
        switch self {
        case .love: return "heart.fill"
        case .friendship: return "person.2.fill"
        case .learning: return "book.fill"
        case .events: return "calendar"
        case .activities: return "figure.run"
        case .entertainment: return "tv.fill"
        case .shopping: return "cart.fill"
        case .growth: return "chart.line.uptrend.xyaxis"
        }
    }
    
    var color: Color {
        switch self {
        case .love: return Color(hex: "FF6B9D")
        case .friendship: return Color(hex: "4A90E2")
        case .learning: return Color(hex: "9B59B6")
        case .events: return Color(hex: "1ABC9C")
        case .activities: return Color(hex: "2ECC71")
        case .entertainment: return Color(hex: "5C6BC0")
        case .shopping: return Color(hex: "FFC107")
        case .growth: return Color(hex: "673AB7")
        }
    }
}
```

**File:** `Sources/TitanExplore/Views/MatchDiscoveryView.swift`

```swift
import SwiftUI

struct MatchDiscoveryView: View {
    let domain: MatchDomain
    @StateObject private var viewModel: MatchDiscoveryViewModel
    @Environment(\.dismiss) private var dismiss
    
    init(domain: MatchDomain) {
        self.domain = domain
        self._viewModel = StateObject(wrappedValue: MatchDiscoveryViewModel(domain: domain))
    }
    
    var body: some View {
        NavigationView {
            ZStack {
                // Background
                Color.gray.opacity(0.1)
                    .ignoresSafeArea()
                
                // Match Stack
                if viewModel.matches.isEmpty {
                    emptyState
                } else {
                    matchStack
                }
            }
            .navigationTitle("\(domain.displayName) Matches")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("Close") {
                        dismiss()
                    }
                }
                
                ToolbarItem(placement: .navigationBarTrailing) {
                    Text("\(viewModel.currentIndex + 1)/\(viewModel.matches.count)")
                        .font(.subheadline)
                        .foregroundColor(.secondary)
                }
            }
        }
        .onAppear {
            Task {
                await viewModel.loadMatches()
            }
        }
    }
    
    private var matchStack: some View {
        ZStack {
            ForEach(Array(viewModel.matches.enumerated()), id: \.element.id) { index, match in
                if index >= viewModel.currentIndex {
                    MatchCard(
                        match: match,
                        onLike: { viewModel.likeMatch(match) },
                        onPass: { viewModel.passMatch(match) },
                        onSuperLike: { viewModel.superLikeMatch(match) }
                    )
                    .zIndex(Double(viewModel.matches.count - index))
                    .offset(
                        x: index == viewModel.currentIndex ? viewModel.dragOffset.width : 0,
                        y: index == viewModel.currentIndex ? viewModel.dragOffset.height : CGFloat(index - viewModel.currentIndex) * 10
                    )
                    .scaleEffect(index == viewModel.currentIndex ? 1.0 : 0.95)
                    .opacity(index == viewModel.currentIndex ? 1.0 : 0.5)
                }
            }
        }
        .padding()
    }
    
    private var emptyState: some View {
        VStack(spacing: 16) {
            Image(systemName: "checkmark.circle.fill")
                .font(.system(size: 64))
                .foregroundColor(.green)
            
            Text("All caught up!")
                .font(.title2)
                .fontWeight(.semibold)
            
            Text("Check back tomorrow for new matches")
                .font(.subheadline)
                .foregroundColor(.secondary)
        }
    }
}

struct MatchCard: View {
    let match: Match
    let onLike: () -> Void
    let onPass: () -> Void
    let onSuperLike: () -> Void
    
    @State private var showFullProfile = false
    
    var body: some View {
        ZStack(alignment: .bottom) {
            // Photo
            AsyncImage(url: match.matchedUser.primaryPhotoURL) { image in
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
                startPoint: .center,
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
                
                // Compatibility Factors
                ForEach(match.compatibilityFactors.prefix(3)) { factor in
                    HStack(spacing: 8) {
                        Image(systemName: factor.category.icon)
                            .foregroundColor(.white.opacity(0.8))
                        
                        Text(factor.description)
                            .font(.subheadline)
                            .foregroundColor(.white.opacity(0.9))
                    }
                }
            }
            .padding()
            .background(.ultraThinMaterial)
            .cornerRadius(24, corners: [.topLeft, .topRight])
        }
        .cornerRadius(24)
        .shadow(color: .black.opacity(0.1), radius: 20, y: 10)
        .overlay(alignment: .bottom) {
            // Action Buttons
            HStack(spacing: 24) {
                Button(action: onPass) {
                    Image(systemName: "xmark")
                        .font(.title2)
                        .foregroundColor(.white)
                        .frame(width: 60, height: 60)
                        .background(Color.red)
                        .clipShape(Circle())
                        .shadow(radius: 5)
                }
                
                Button(action: { showFullProfile = true }) {
                    Image(systemName: "info.circle")
                        .font(.title2)
                        .foregroundColor(.white)
                        .frame(width: 50, height: 50)
                        .background(Color.blue)
                        .clipShape(Circle())
                        .shadow(radius: 5)
                }
                
                Button(action: onLike) {
                    Image(systemName: "heart.fill")
                        .font(.title2)
                        .foregroundColor(.white)
                        .frame(width: 60, height: 60)
                        .background(Color.green)
                        .clipShape(Circle())
                        .shadow(radius: 5)
                }
            }
            .padding(.bottom, 32)
        }
        .sheet(isPresented: $showFullProfile) {
            FullProfileView(match: match)
        }
    }
}

extension FactorCategory {
    var icon: String {
        switch self {
        case .values: return "star.fill"
        case .personality: return "brain.head.profile"
        case .goals: return "target"
        case .interests: return "heart.fill"
        case .schedule: return "calendar"
        case .location: return "mappin"
        case .lifestyle: return "figure.walk"
        }
    }
}
```

### Month 4: Testing & Polish

#### Week 1-2: Unit Tests

**File:** `Tests/TitanExploreTests/MatchingEngineTests.swift`

```swift
import XCTest
@testable import TitanExplore

final class MatchingEngineTests: XCTestCase {
    var engine: MatchingEngine!
    
    override func setUp() async throws {
        engine = MatchingEngine.shared
    }
    
    func testCompatibilityCalculation() async throws {
        // Given
        let user = createTestUser(
            openness: 80,
            conscientiousness: 70,
            extraversion: 60,
            agreeableness: 75,
            neuroticism: 40
        )
        
        let candidate = createTestUser(
            openness: 85,
            conscientiousness: 65,
            extraversion: 55,
            agreeableness: 80,
            neuroticism: 35
        )
        
        // When
        let score = try await engine.calculateCompatibility(
            user: user,
            candidate: candidate,
            domain: .love
        )
        
        // Then
        XCTAssertGreaterThan(score, 70, "Similar personalities should have high compatibility")
    }
    
    func testMatchGeneration() async throws {
        // Given
        let userId = UUID()
        
        // When
        let matches = try await engine.generateMatches(
            for: userId,
            domain: .love,
            limit: 10
        )
        
        // Then
        XCTAssertLessThanOrEqual(matches.count, 10)
        XCTAssertTrue(matches.allSatisfy { $0.compatibilityScore >= 70 })
    }
}
```

---

## Phase 2: Expansion (Months 5-8)

### Month 5: Messaging & Connections

### Month 6: Events & Groups

### Month 7: AI Features

### Month 8: Testing & Optimization

---

## Phase 3: Scale (Months 9-12)

### Month 9: Cloud Migration

### Month 10: Performance Optimization

### Month 11: International Expansion

### Month 12: Launch Preparation

---

## Deployment Checklist

### Pre-Launch
- [ ] All features tested
- [ ] Performance benchmarks met
- [ ] Security audit completed
- [ ] Privacy policy updated
- [ ] Terms of service finalized
- [ ] App Store assets prepared
- [ ] Beta testing completed
- [ ] Analytics configured
- [ ] Crash reporting enabled
- [ ] Customer support ready

### Launch Day
- [ ] Submit to App Store
- [ ] Monitor crash reports
- [ ] Track user feedback
- [ ] Respond to reviews
- [ ] Monitor server load
- [ ] Check analytics

### Post-Launch
- [ ] Weekly updates based on feedback
- [ ] Monthly feature releases
- [ ] Quarterly major updates
- [ ] Continuous optimization

---

*Document prepared by: Engineering Team*  
*Last updated: January 17, 2026*  
*Next review: February 1, 2026*
