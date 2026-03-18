# Gamify-Laravel — Models & Relationships

## Models Overview

### 1. User
The central model. Represents anyone who uses the app (players and admins).

- **Key fields:** name, username, email, password, role (admin or player)
- **Computed fields:** `experience` (sum of all points), `level` (determined by XP)

### 2. UserProfile
Stores extra information about a user — bio, date of birth, social media links, and avatar.

- **Key fields:** bio, date_of_birth, twitter, facebook, linkedin, github
- Each user has exactly one profile.

### 3. Badge
An achievement that users can earn. Badges are triggered by specific events (called "actuators") such as answering a question, logging in, or updating a profile.

- **Key fields:** name, description, required_repetitions, active, actuators
- Badges can be tagged (e.g. tagged with "math" so they only trigger on math questions).
- Has an image stored via Spatie Media Library.

### 4. Level
Represents a rank tier (e.g. "Beginner", "Expert"). Users are assigned a level based on their total XP.

- **Key fields:** name, required_points, active
- Levels are looked up by comparing a user's XP to the `required_points` threshold.

### 5. Question
A quiz question that users can answer to earn points.

- **Key fields:** name, question (body text), solution, type (single/multi answer), hidden, status
- Has a lifecycle: draft → pending → published (or scheduled for future).
- Tracks who created and last updated it (via `BlameableTrait`).
- Can be tagged for categorisation.

### 6. QuestionChoice
An individual answer option for a question.

- **Key fields:** text, score
- A positive score means the choice is correct. Zero or negative means incorrect.

### 7. Point
A record of points awarded to a user (like a transaction log).

- **Key fields:** points, description
- Fires events (`PointCreated`, `PointDeleted`) when created or deleted — these are used to keep the XP cache in sync.

### 8. UserResponse *(pivot model)*
Represents a user's answer to a question. Stored in the `users_questions` pivot table.

- **Key fields:** points (XP earned), answers (comma-separated choice IDs)
- Users always get at least 1 XP even for wrong answers.

### 9. UserBadgeProgress *(pivot model)*
Tracks a user's progress toward earning a badge. Stored in the `users_badges` pivot table.

- **Key fields:** repetitions (how many times the trigger event has occurred), unlocked_at (null = still locked)

### 10. LinkedSocialAccount
Stores OAuth social login connections (e.g. GitHub, Facebook).

- **Key fields:** provider_name, provider_id

---

## Relationships Map

```
User
 ├── has one ──────── UserProfile (bio, avatar, social links)
 ├── has many ─────── Point (XP transaction log)
 ├── has many ─────── LinkedSocialAccount (social logins)
 ├── belongs to many ── Badge (through users_badges pivot / UserBadgeProgress)
 └── belongs to many ── Question (through users_questions pivot / UserResponse)

Question
 ├── has many ─────── QuestionChoice (answer options)
 ├── belongs to ───── User (creator, via BlameableTrait)
 └── belongs to many ── User (users who answered, via UserResponse pivot)

QuestionChoice
 └── belongs to ───── Question

Badge
 └── belongs to many ── User (users who earned it, via UserBadgeProgress pivot)

Level
 └── (no direct relationships — looked up by XP threshold)

Point
 └── belongs to ───── User

LinkedSocialAccount
 └── belongs to ───── User

UserProfile
 └── belongs to ───── User
```

---

## Interesting Things

1. **Event-driven badge system:** Badges use "actuators" — bitwise flags that define which event triggers them (e.g. login, answering questions, uploading avatar). This makes the badge system very flexible without needing separate code for each badge type.

2. **Pivot models with behaviour:** Instead of plain pivot tables, `UserResponse` and `UserBadgeProgress` are custom Pivot classes. `UserResponse` has logic to calculate scores and parse answer choices. `UserBadgeProgress` tracks repetitions and unlock timestamps.

3. **Cached XP:** A user's `experience` is computed from the sum of their `points` records, cached for 10 minutes for performance. The cache is invalidated via `PointCreated`/`PointDeleted` events.

4. **Levels are thresholds, not relationships:** Users don't have a `level_id`. Instead, their level is computed at runtime by comparing their XP to the levels table. This means level boundaries can be adjusted without migrating user data.

5. **Question lifecycle:** Questions go through a state machine (draft → pending → scheduled → published) with validation — they must have at least 2 choices and at least 1 correct choice before they can be published.

6. **Presenter pattern:** Models use the `UsesPresenters` trait to separate display/formatting logic from the model itself, keeping the codebase cleaner.
