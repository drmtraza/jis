# Firestore Schema: Check-In Circles Feature

This document outlines the Firestore database schema for the "Check-In Circles" feature.

## Collections

### 1. `checkInQuestionSets`

Defines a set of questions for a specific period (e.g., a week).

*   **Document ID:** `setID` (String, auto-generated)
*   **Fields:**
    *   `setName`: (String) Descriptive name for the set (e.g., "Week 1: Deepening Communication").
    *   `weekIdentifier`: (String or Number) Unique identifier for the period this set applies to (e.g., "2024-W34" or sequential `weekNumber: 1`).
    *   `themeDescription`: (String, Optional) A brief overview of the week's theme or topic.
    *   `isActive`: (Boolean) Controls whether this set is currently active and can be assigned or started by couples. Managed by admins.
    *   `isDefault`: (Boolean, Optional) If `true`, this set might be used as an introductory or default set if no other specific set is scheduled for a new couple.
    *   `questionCount`: (Number, Denormalized) The total number of questions in this set. Useful for UI display (e.g., "Progress: 3 of 7 questions"). This should be updated if questions are added/removed from the set (e.g., via a Cloud Function or admin process).
    *   `createdAt`: (Timestamp) Timestamp of when the question set was created.
    *   `updatedAt`: (Timestamp) Timestamp of the last update to the question set.

### 2. `checkInQuestions`

Stores individual questions that belong to a `CheckInQuestionSet`.

*   **Document ID:** `questionID` (String, auto-generated)
*   **Fields:**
    *   `setID`: (String, FK) References `checkInQuestionSets.setID`. The question set this question belongs to.
    *   `text`: (String) The full text of the question.
    *   `questionType`: (String) Defines the type of response expected. Examples:
        *   `"scale_1_5"` (Numerical scale 1 to 5)
        *   `"scale_1_7_emoji"` (Numerical scale 1 to 7, potentially with client-side emoji mapping)
        *   `"boolean_yes_no"`
    *   `scaleLabels`: (Map, Optional) Provides labels for scale points if needed for display.
        *   Example for `scale_1_5`: `{"1": "Strongly Disagree", "3": "Neutral", "5": "Strongly Agree"}`
        *   Example for `scale_1_7_emoji`: `{"1": "😞", "4": "😐", "7": "😄"}` (Client can map these to numeric values 1-7)
    *   `order`: (Integer) The sequence number of this question within its set (e.g., 1, 2, 3...).
    *   `createdAt`: (Timestamp) Timestamp of when the question was created.
    *   `updatedAt`: (Timestamp) Timestamp of the last update to the question.

### 3. `coupleCheckIns`

Tracks the state and progress of a specific check-in (a specific question set for a specific week/period) for a specific couple.

*   **Document ID:** `coupleCheckInID` (String, auto-generated)
*   **Fields:**
    *   `pairID`: (String, FK) References `pairs.pairID`. Identifies the couple.
    *   `setID`: (String, FK) References `checkInQuestionSets.setID`. Identifies the question set for this check-in instance.
    *   `weekIdentifier`: (String or Number, Denormalized) Copied from `checkInQuestionSets.weekIdentifier`. Used for efficient querying of a couple's check-in for a specific period.
    *   `userIDs`: (Array of Strings) Contains the two `userID`s of the individuals in the pair (e.g., `[userID_A, userID_B]`). The order can be deterministic (e.g., sorted alphabetically, or husband first if roles are consistently applied).
    *   `userResponsesInfo`: (Map) Stores the status and summary for each user's participation in this check-in. Keys are the actual `userID`s.
        *   `[userID_A]`: (Map)
            *   `status`: (String) Enum: `"pending"`, `"in_progress"`, `"completed"`.
            *   `responsesCount`: (Number) How many questions this user has answered for this specific check-in.
            *   `completedAt`: (Timestamp, Optional) When this user submitted their final response for this check-in.
        *   `[userID_B]`: (Map) (Same structure as `[userID_A]`)
    *   `completionMode`: (String) Enum:
        *   `"individual"`: Each user answers privately, then responses are revealed to both. (Recommended for V1)
        *   `"together"`: Couple answers together on one device (adds UI/UX complexity for attributing responses if needed).
        *   `"not_started"`: Default initial state.
    *   `responsesVisibleToCouple`: (Boolean)
        *   If `completionMode` is `"individual"`, this becomes `true` only when both users have their `status` as `"completed"` in `userResponsesInfo`.
        *   If `completionMode` is `"together"`, this could be `true` once they start.
        *   This field can be managed by Cloud Functions reacting to status updates or by client-side logic.
    *   `startedAt`: (Timestamp, Optional) When the first user starts answering or when a "together" session is initiated.
    *   `lastActivityAt`: (Timestamp) Timestamp of the most recent activity (e.g., a response saved). Useful for reminders or tracking stale check-ins.
    *   `fullyCompletedAt`: (Timestamp, Optional) When the check-in is considered fully completed by the couple/system.
        *   For `"individual"` mode: when both users' statuses are `"completed"`.
        *   For `"together"` mode: when the couple explicitly marks the session as complete.

### 4. `userCheckInResponses`

Stores the actual answers provided by each user to each question in a check-in.

*   **Document ID:** `responseID` (String, auto-generated)
*   **Fields:**
    *   `coupleCheckInID`: (String, FK) References `coupleCheckIns.coupleCheckInID`. Links the response to a specific couple's check-in session.
    *   `userID`: (String, FK) References `users.userID`. Identifies the user who provided this specific answer.
    *   `questionID`: (String, FK) References `checkInQuestions.questionID`. Identifies the question being answered.
    *   `setID`: (String, FK, Denormalized) References `checkInQuestionSets.setID`. Denormalized from `checkInQuestions` or `coupleCheckIns` for potential broad queries on responses to a particular question set across all users/couples (optional, but can be useful for analytics).
    *   `responseValue`: (Number for V1) The user's answer. For scale-based questions (e.g., 1-5), this will be a number. If text-based responses are introduced later, this field type might need to become more flexible (e.g., a map `{"scaleValue": 3, "textValue": "comment"}`) or use separate fields. For V1, assuming numerical responses for scales.
    *   `respondedAt`: (Timestamp) When this specific response was saved or last updated.

## Relationships Summary

*   A `Couple` (from existing `pairs` collection) undertakes a `CoupleCheckIn`.
*   Each `CoupleCheckIn` uses one `CheckInQuestionSet`.
*   A `CheckInQuestionSet` consists of multiple `CheckInQuestions`.
*   Each `User` in the couple provides multiple `UserCheckInResponses` for the `CheckInQuestions` within their `CoupleCheckIn` session.

## Indexing Considerations (Conceptual)

*   **`checkInQuestionSets`**:
    *   `(isActive, weekIdentifier)` - For finding the active set for the current week.
    *   `(isActive, isDefault)` - For finding a default set.
*   **`checkInQuestions`**:
    *   `(setID, order)` - To fetch all questions for a set, in order.
*   **`coupleCheckIns`**:
    *   `(pairID, weekIdentifier)` - To find a specific couple's check-in for a week.
    *   `(pairID, setID)` - Alternative way to find a couple's check-in for a set.
    *   `(pairID, fullyCompletedAt)` - To find a couple's past completed check-ins.
    *   Consider indexing fields within `userResponsesInfo` if complex status queries are needed, e.g., `(userResponsesInfo.[userID_A].status)`. However, direct document reads are often sufficient.
*   **`userCheckInResponses`**:
    *   `(coupleCheckInID, userID)` - To fetch all responses from a specific user for a specific check-in session.
    *   `(coupleCheckInID, questionID)` - To fetch all responses to a specific question within a check-in session (useful if both users respond to the same question instance, though in 'individual' mode, they'd have separate `UserCheckInResponses` documents).
    *   `(userID, setID, respondedAt)` - To fetch all of a user's responses for a particular question set over time.

This revised schema aims for clarity, query efficiency, and robustness in handling the "Check-In Circles" feature.
