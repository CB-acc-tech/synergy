# Design Document: Multilingual AI Assistant for Government Services

## Overview

The Multilingual AI Assistant is a web-based conversational application that helps Indian citizens discover government welfare schemes they're eligible for. The system uses a hybrid architecture that combines deterministic rule-based eligibility matching with AI-powered natural language understanding to ensure accuracy while maintaining conversational accessibility.

The core design principle is separation of concerns: structured government data and eligibility logic remain deterministic and verifiable, while AI handles only language understanding, translation, and conversational flow. This prevents hallucination of schemes or eligibility criteria while enabling natural, multilingual interaction.

## Architecture

The system follows a three-tier architecture:

### Presentation Layer
- Mobile-first responsive web interface
- Language selector component
- Conversational UI with message bubbles
- Voice input/output interface (optional)

### Application Layer
- Conversation Manager: Orchestrates dialogue flow
- AI Language Model Integration: Handles NLU and translation
- Eligibility Engine: Applies deterministic rules
- Session Manager: Maintains conversation state

### Data Layer
- Scheme Dataset: Structured government scheme information
- Session Store: Temporary user conversation data
- Configuration Store: Language mappings and system settings

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   Presentation Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Web UI     │  │   Language   │  │    Voice     │  │
│  │  Component   │  │   Selector   │  │  Interface   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │Conversation  │  │  AI Language │  │  Eligibility │  │
│  │   Manager    │◄─┤     Model    │  │    Engine    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                                      │         │
│  ┌──────────────┐                             │         │
│  │   Session    │                             │         │
│  │   Manager    │                             │         │
│  └──────────────┘                             │         │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                      Data Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Scheme     │  │   Session    │  │Configuration │  │
│  │   Dataset    │  │    Store     │  │    Store     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Components and Interfaces

### 1. Conversation Manager

**Responsibility**: Orchestrates the dialogue flow, manages question sequencing, and coordinates between components.

**Interface**:
```typescript
interface ConversationManager {
  startConversation(sessionId: string, language: string): ConversationState
  processUserInput(sessionId: string, input: string): ConversationResponse
  getCurrentQuestion(sessionId: string): Question
  collectUserInfo(sessionId: string, answer: Answer): void
  getEligibleSchemes(sessionId: string): Scheme[]
}

interface ConversationState {
  sessionId: string
  language: string
  currentStep: ConversationStep
  collectedInfo: UserInfo
}

interface ConversationResponse {
  message: string
  options?: string[]
  requiresInput: boolean
  isComplete: boolean
}

enum ConversationStep {
  LANGUAGE_SELECTION,
  AGE_COLLECTION,
  OCCUPATION_COLLECTION,
  INCOME_COLLECTION,
  LOCATION_COLLECTION,
  SPECIAL_CATEGORY_COLLECTION,
  RESULTS_PRESENTATION
}
```

**Behavior**:
- Maintains a state machine for conversation flow
- Validates answers before proceeding
- Delegates language understanding to AI Language Model
- Delegates eligibility checking to Eligibility Engine

### 2. AI Language Model Integration

**Responsibility**: Handles natural language understanding, intent extraction, and translation without generating scheme information.

**Interface**:
```typescript
interface AILanguageModel {
  parseUserInput(input: string, language: string, expectedType: InputType): ParsedInput
  translateText(text: string, fromLanguage: string, toLanguage: string): string
  simplifyText(text: string, language: string): string
  extractIntent(input: string, language: string): Intent
}

interface ParsedInput {
  value: any
  confidence: number
  needsClarification: boolean
}

enum InputType {
  AGE_RANGE,
  OCCUPATION,
  INCOME_BRACKET,
  LOCATION,
  YES_NO,
  FREE_TEXT
}

interface Intent {
  type: string
  entities: Map<string, any>
}
```

**Behavior**:
- Uses LLM API (e.g., OpenAI, Gemini) for language understanding
- Constrains LLM output to parsing and translation only
- Does not allow LLM to generate eligibility rules or scheme information
- Implements prompt engineering to prevent hallucination

### 3. Eligibility Engine

**Responsibility**: Applies deterministic rules to determine which schemes a user qualifies for.

**Interface**:
```typescript
interface EligibilityEngine {
  evaluateEligibility(userInfo: UserInfo, schemes: Scheme[]): EligibleScheme[]
  checkCriteria(userInfo: UserInfo, criteria: EligibilityCriteria): boolean
}

interface UserInfo {
  ageRange: AgeRange
  occupation: string
  incomeBracket: IncomeBracket
  state: string
  district: string
  specialCategories: string[]
}

interface EligibilityCriteria {
  ageMin?: number
  ageMax?: number
  occupations?: string[]
  incomeMax?: number
  states?: string[]
  requiredCategories?: string[]
  excludedCategories?: string[]
}

interface EligibleScheme {
  scheme: Scheme
  matchedCriteria: string[]
}
```

**Behavior**:
- Evaluates each scheme's criteria against user information
- Uses boolean logic (AND/OR conditions)
- Returns only schemes where all required criteria are met
- Provides explanation of which criteria matched

### 4. Scheme Dataset

**Responsibility**: Stores verified government scheme information in a structured format.

**Data Model**:
```typescript
interface Scheme {
  id: string
  name: Record<string, string> // language -> name
  description: Record<string, string> // language -> description
  benefits: Record<string, string[]> // language -> benefit list
  eligibilityCriteria: EligibilityCriteria
  requiredDocuments: Record<string, string[]> // language -> document list
  applicationSteps: Record<string, string[]> // language -> step list
  officialLink: string
  lastUpdated: Date
}
```

**Storage**:
- JSON files for initial implementation (easy to update)
- Can be migrated to database for production
- Supports versioning for scheme updates

### 5. Session Manager

**Responsibility**: Manages user sessions and temporary data storage.

**Interface**:
```typescript
interface SessionManager {
  createSession(language: string): string
  getSession(sessionId: string): Session | null
  updateSession(sessionId: string, data: Partial<Session>): void
  deleteSession(sessionId: string): void
  cleanupExpiredSessions(): void
}

interface Session {
  id: string
  language: string
  conversationState: ConversationState
  createdAt: Date
  lastAccessedAt: Date
  expiresAt: Date
}
```

**Behavior**:
- In-memory storage for development (Redis for production)
- 30-minute session timeout
- Automatic cleanup of expired sessions
- No persistent storage of user data

### 6. Translation Service

**Responsibility**: Handles translation between Indian languages.

**Interface**:
```typescript
interface TranslationService {
  translate(text: string, fromLang: string, toLang: string): Promise<string>
  translateBatch(texts: string[], fromLang: string, toLang: string): Promise<string[]>
  getSupportedLanguages(): string[]
}
```

**Supported Languages** (Initial):
- Hindi (hi)
- English (en)
- Tamil (ta)
- Telugu (te)
- Bengali (bn)
- Marathi (mr)
- Gujarati (gu)
- Kannada (kn)
- Malayalam (ml)
- Punjabi (pa)

### 7. Voice Interface (Optional)

**Responsibility**: Handles voice input and output.

**Interface**:
```typescript
interface VoiceInterface {
  speechToText(audioBlob: Blob, language: string): Promise<string>
  textToSpeech(text: string, language: string): Promise<AudioBuffer>
  isVoiceSupported(language: string): boolean
}
```

**Implementation**:
- Web Speech API for browser-based voice
- Fallback to cloud services (Google Cloud Speech-to-Text)

## Data Models

### User Information Model

```typescript
enum AgeRange {
  BELOW_18 = "below_18",
  AGE_18_25 = "18_25",
  AGE_26_35 = "26_35",
  AGE_36_50 = "36_50",
  AGE_51_60 = "51_60",
  ABOVE_60 = "above_60"
}

enum IncomeBracket {
  BELOW_1L = "below_1_lakh",
  RANGE_1L_3L = "1_3_lakh",
  RANGE_3L_5L = "3_5_lakh",
  RANGE_5L_8L = "5_8_lakh",
  ABOVE_8L = "above_8_lakh"
}

interface UserInfo {
  ageRange: AgeRange
  occupation: string
  incomeBracket: IncomeBracket
  state: string
  district: string
  specialCategories: SpecialCategory[]
}

enum SpecialCategory {
  FARMER = "farmer",
  STUDENT = "student",
  SENIOR_CITIZEN = "senior_citizen",
  WOMAN = "woman",
  DISABLED = "disabled",
  SC_ST = "sc_st",
  OBC = "obc",
  MINORITY = "minority",
  BPL = "bpl"
}
```

### Conversation Flow Model

```typescript
interface Question {
  id: string
  text: Record<string, string> // language -> question text
  type: InputType
  options?: Record<string, string[]> // language -> option list
  validation?: ValidationRule
}

interface ValidationRule {
  required: boolean
  pattern?: string
  customValidator?: (value: any) => boolean
}

interface Answer {
  questionId: string
  value: any
  timestamp: Date
}
```

## Conversation Flow

### Question Sequence

1. **Language Selection**
   - Display: Language options with native script
   - Input: Button selection
   - Validation: Must select one language

2. **Age Range**
   - Display: "What is your age range?"
   - Options: Below 18, 18-25, 26-35, 36-50, 51-60, Above 60
   - Input: Button selection or natural language

3. **Occupation**
   - Display: "What is your occupation?"
   - Options: Farmer, Student, Employed, Self-employed, Unemployed, Retired, Other
   - Input: Button selection or natural language

4. **Income Bracket**
   - Display: "What is your annual household income?"
   - Options: Below 1 lakh, 1-3 lakhs, 3-5 lakhs, 5-8 lakhs, Above 8 lakhs
   - Input: Button selection or natural language

5. **Location**
   - Display: "Which state and district do you live in?"
   - Input: Dropdown for state, then district
   - Validation: Must be valid Indian state and district

6. **Special Categories**
   - Display: "Do any of these apply to you?"
   - Options: Farmer, Student, Senior Citizen, Woman, Person with Disability, SC/ST, OBC, Minority, BPL
   - Input: Multi-select checkboxes
   - Validation: Can select multiple or none

7. **Results Presentation**
   - Display: List of eligible schemes
   - For each scheme: Name, Benefits, Documents, Steps, Official Link

### Conversation State Machine

```
START → LANGUAGE_SELECTION → AGE_COLLECTION → OCCUPATION_COLLECTION 
  → INCOME_COLLECTION → LOCATION_COLLECTION → SPECIAL_CATEGORY_COLLECTION 
  → ELIGIBILITY_CHECK → RESULTS_PRESENTATION → END
```

## Error Handling

### Error Categories

1. **User Input Errors**
   - Invalid format
   - Out of range values
   - Missing required information
   - **Handling**: Ask user to rephrase or provide valid input

2. **AI Model Errors**
   - API timeout
   - Parsing failure
   - Low confidence in understanding
   - **Handling**: Fall back to structured options, ask clarifying questions

3. **Translation Errors**
   - Unsupported language pair
   - Translation API failure
   - **Handling**: Display in default language (English) with error message

4. **Data Errors**
   - Scheme dataset unavailable
   - Missing scheme information
   - **Handling**: Display error message, log for admin review

5. **Session Errors**
   - Session expired
   - Session not found
   - **Handling**: Start new conversation, inform user

### Error Response Format

```typescript
interface ErrorResponse {
  type: ErrorType
  message: Record<string, string> // language -> error message
  recoveryAction: RecoveryAction
  userFriendlyMessage: string
}

enum ErrorType {
  INPUT_VALIDATION,
  AI_PROCESSING,
  TRANSLATION,
  DATA_ACCESS,
  SESSION
}

enum RecoveryAction {
  RETRY,
  REPHRASE,
  USE_STRUCTURED_INPUT,
  START_NEW_SESSION,
  CONTACT_SUPPORT
}
```

## Testing Strategy

The system will be tested using a dual approach combining unit tests for specific scenarios and property-based tests for universal correctness properties.

### Unit Testing

Unit tests will focus on:
- **Specific examples**: Verify correct behavior for known inputs (e.g., a farmer aged 30 with income below 1 lakh qualifies for PM-KISAN)
- **Edge cases**: Empty inputs, boundary values, special characters in text
- **Error conditions**: Invalid language codes, malformed scheme data, expired sessions
- **Integration points**: API calls to AI models, translation services, data access

### Property-Based Testing

Property-based tests will verify universal properties across many randomly generated inputs. Each test will run a minimum of 100 iterations to ensure comprehensive coverage.

**Testing Framework**: We will use `fast-check` (for TypeScript/JavaScript) to implement property-based tests.

**Test Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with: **Feature: multilingual-govt-assistant, Property {N}: {property_text}**
- Tests reference design document properties

### Test Coverage Goals

- Unit test coverage: 80%+ for core logic
- Property test coverage: All correctness properties implemented
- Integration test coverage: All API endpoints and component interactions
- End-to-end test coverage: Complete user flows for each language


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Language Consistency

*For any* user session and selected language, all system responses, questions, and displayed content should be in the user's selected language throughout the entire conversation.

**Validates: Requirements 1.2, 6.2, 7.1**

### Property 2: Single Question Display

*For any* conversation state, the system should present exactly one question at a time, never multiple simultaneous questions.

**Validates: Requirements 1.3**

### Property 3: Question Sequence Ordering

*For any* conversation flow, questions should follow the defined sequence: language selection → age → occupation → income → location → special categories → results.

**Validates: Requirements 2.1**

### Property 4: Answer Validation Before Progression

*For any* user answer, the system should validate the response before advancing to the next question, rejecting invalid inputs.

**Validates: Requirements 2.7**

### Property 5: Complete Scheme Evaluation

*For any* user information and scheme dataset, the eligibility engine should evaluate the user against every scheme in the dataset.

**Validates: Requirements 3.1**

### Property 6: Eligibility Determinism

*For any* user information, running the eligibility check multiple times should produce identical results (same schemes in same order).

**Validates: Requirements 3.2**

### Property 7: Correct Eligibility Filtering

*For any* user information and scheme dataset, the results should include all and only those schemes where the user meets all eligibility criteria.

**Validates: Requirements 3.3, 3.4**

### Property 8: Complete Scheme Structure

*For any* scheme in the dataset, it should contain all required fields: eligibility criteria, benefits, required documents, application steps, and official link.

**Validates: Requirements 4.1, 4.2, 4.3, 4.4, 4.5**

### Property 9: No Scheme Hallucination

*For any* displayed scheme, its ID should exist in the verified scheme dataset (no AI-generated or hallucinated schemes).

**Validates: Requirements 4.6, 9.4**

### Property 10: Structured Parse Output

*For any* user input and expected input type, the AI language model's parse function should return a structured ParsedInput object with value, confidence, and clarification flag.

**Validates: Requirements 5.1**

### Property 11: Low Confidence Clarification

*For any* parsed input with confidence below threshold, the system should request clarification from the user.

**Validates: Requirements 5.2**

### Property 12: Valid Entity Extraction

*For any* user input, extracted entities should match the expected entity types for the current conversation step.

**Validates: Requirements 5.3**

### Property 13: AI Scope Constraint

*For any* AI-generated response, it should not contain scheme information (names, benefits, eligibility criteria) that doesn't exist in the scheme dataset.

**Validates: Requirements 5.4**

### Property 14: Complete Results Display

*For any* user who qualifies for N schemes, the results should display all N eligible schemes.

**Validates: Requirements 6.1**

### Property 15: Complete Scheme Information Display

*For any* displayed scheme, it should include the scheme name, benefits, required documents, application steps, and official link in the user's language.

**Validates: Requirements 6.2, 6.4, 6.5, 6.6**

### Property 16: Voice-to-Text Conversion

*For any* voice input (when voice is enabled), the speech-to-text function should return a non-empty text string.

**Validates: Requirements 8.2**

### Property 17: Text-to-Speech Conversion

*For any* text response (when voice is enabled), the text-to-speech function should return audio data.

**Validates: Requirements 8.3**

### Property 18: Voice Language Support

*For any* user's selected language (when voice is enabled), the voice interface should accept that language as a parameter for both input and output.

**Validates: Requirements 8.4**

### Property 19: Session Data Deletion

*For any* session, after the session expires or ends, all user data associated with that session should be deleted and not retrievable.

**Validates: Requirements 9.3, 11.4**

### Property 20: Session Creation

*For any* conversation start, the system should create a new session and return a unique session ID.

**Validates: Requirements 11.1**

### Property 21: Session Data Persistence

*For any* active session, conversation history and collected user information should be retrievable throughout the session lifetime.

**Validates: Requirements 11.2, 11.3**

### Property 22: Fresh Conversation After Timeout

*For any* expired session ID, attempting to continue the conversation should trigger a new conversation flow starting from language selection.

**Validates: Requirements 11.5**

### Property 23: Parse Failure Recovery

*For any* AI parsing failure, the system should respond with a request for the user to rephrase their input.

**Validates: Requirements 12.1**

### Property 24: Translation Failure Fallback

*For any* translation failure, the system should display content in the default language (English) along with an error message.

**Validates: Requirements 12.2**

### Property 25: Dataset Unavailability Error

*For any* scheme dataset access failure, the system should display an appropriate error message to the user.

**Validates: Requirements 12.3**

### Property 26: Voice Failure Fallback

*For any* voice input failure, the system should prompt the user to use text input instead.

**Validates: Requirements 12.4**

### Property 27: Error Logging Without Exposure

*For any* system error, the error should be logged with technical details, but the user-facing message should not contain stack traces or technical implementation details.

**Validates: Requirements 12.5**

### Property 28: Responsive Layout Adaptation

*For any* viewport width, the UI should adapt its layout appropriately (mobile layout for widths < 768px, desktop layout for widths ≥ 768px).

**Validates: Requirements 10.1**
