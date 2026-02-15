# Design Document: AI Decision Engine

## Overview

The AI Decision Engine is a serverless, microservice-based system that transforms career and startup decision-making from subjective advice into quantitative, explainable analysis. The system processes natural language descriptions of user situations, structures them into decision profiles, evaluates multiple paths using configurable scoring rules, simulates future outcomes with risk assessment, and generates actionable weekly roadmaps.

The architecture follows a pipeline pattern: Input → Extraction → Scoring → Simulation → Generation → Output, with each stage implemented as an independent AWS Lambda function. This design prioritizes modularity, scalability, and explainability while maintaining sub-3-second response times for core operations.

### Key Design Principles

1. **Explainability First**: Every score, recommendation, and risk assessment includes transparent reasoning
2. **Modular Services**: Each component operates independently with well-defined interfaces
3. **Serverless Scalability**: Auto-scaling Lambda functions handle variable load without infrastructure management
4. **Structured Data Flow**: Natural language is converted to structured data early, then processed deterministically
5. **Fail-Safe Operations**: Graceful degradation and clear error messages when services fail

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                           │
│                    (Next.js / React SPA)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS API Gateway                             │
│                  (REST API + Authentication)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
┌───────────────────────────┐   ┌──────────────────────────┐
│   Context Extractor       │   │   Decision Orchestrator  │
│   (Lambda + Bedrock)      │   │   (Lambda)               │
└───────────┬───────────────┘   └────────┬─────────────────┘
            │                             │
            │ Decision_Profile            │
            ▼                             │
┌───────────────────────────┐            │
│      DynamoDB             │◄───────────┤
│  (Decision Profiles)      │            │
└───────────────────────────┘            │
                                         │
                ┌────────────────────────┼────────────────────┐
                │                        │                    │
                ▼                        ▼                    ▼
┌───────────────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│  Decision Engine      │  │ Simulation Engine│  │  Roadmap Generator  │
│  (Lambda)             │  │ (Lambda)         │  │  (Lambda + Bedrock) │
└───────────┬───────────┘  └────────┬─────────┘  └──────────┬──────────┘
            │                       │                        │
            │ Scores                │ Simulations            │ Roadmap
            └───────────────────────┴────────────────────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │   Response Aggregator │
                        │   (Lambda)            │
                        └───────────┬───────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │   S3 Bucket           │
                        │   (Reports Storage)   │
                        └───────────────────────┘
```

### Component Responsibilities

**Frontend (Next.js/React)**
- Renders decision input forms and comparison dashboards
- Manages user authentication state
- Displays interactive visualizations of scores and simulations
- Handles real-time weight adjustments for scoring

**API Gateway**
- Routes requests to appropriate Lambda functions
- Handles authentication and authorization
- Implements rate limiting and request validation
- Provides CORS support for frontend

**Context Extractor Lambda**
- Receives natural language user input
- Calls Amazon Bedrock LLM to parse text into structured data
- Validates extracted data against Decision_Profile schema
- Identifies missing critical fields
- Stores Decision_Profile in DynamoDB

**Decision Engine Lambda**
- Retrieves Decision_Profile from DynamoDB
- Applies configurable scoring rules to each Decision_Path
- Calculates weighted scores across multiple criteria
- Normalizes scores to 0-100 scale
- Generates explainability data for each score
- Ranks paths by total score

**Simulation Engine Lambda**
- Models future outcomes for each Decision_Path
- Calculates success probabilities based on user context
- Identifies key milestones and estimates timelines
- Quantifies risk indicators
- Performs skill gap analysis

**Roadmap Generator Lambda**
- Takes selected Decision_Path and user constraints
- Calls Amazon Bedrock LLM to generate weekly action plans
- Sequences tasks based on dependencies
- Inserts milestone checkpoints
- Formats roadmap for display and export

**Decision Orchestrator Lambda**
- Coordinates multi-step workflows
- Manages parallel execution of scoring and simulation
- Aggregates results from multiple services
- Handles error recovery and retries

**Response Aggregator Lambda**
- Combines outputs from all analysis services
- Generates Decision_Report documents
- Stores reports in S3
- Returns unified response to frontend

**DynamoDB**
- Stores Decision_Profiles with user association
- Maintains version history for profiles
- Supports fast retrieval by user ID
- Stores scoring configurations and weights

**S3**
- Stores generated Decision_Reports
- Provides pre-signed URLs for secure downloads
- Archives historical reports

**Amazon Bedrock**
- Provides LLM capabilities for natural language understanding
- Generates human-readable roadmap text
- Supports prompt-based extraction and generation

## Components and Interfaces

### Context Extractor

**Purpose**: Convert natural language user input into structured Decision_Profile

**Input Interface**:
```typescript
{
  userId: string,
  inputText: string,
  sessionId: string
}
```

**Output Interface**:
```typescript
{
  profileId: string,
  decisionProfile: Decision_Profile,
  missingFields: string[],
  confidence: number
}
```

**Processing Logic**:
1. Construct LLM prompt with extraction instructions and schema
2. Call Bedrock API with user input text
3. Parse LLM response into Decision_Profile structure
4. Validate required fields (skills, goals, constraints, paths)
5. Identify missing critical information
6. Store profile in DynamoDB with timestamp
7. Return profile ID and extraction results

**Error Handling**:
- LLM timeout: Return partial extraction with missing fields flagged
- Invalid JSON from LLM: Retry with clarified prompt
- Schema validation failure: Request user clarification

### Decision Engine

**Purpose**: Score and rank Decision_Paths using rule-based weighted criteria

**Input Interface**:
```typescript
{
  profileId: string,
  scoringWeights?: ScoringWeights
}
```

**Output Interface**:
```typescript
{
  rankedPaths: Array<{
    pathId: string,
    pathName: string,
    totalScore: number,
    scoreBreakdown: {
      skillMatch: number,
      resourceFit: number,
      timelineFeasibility: number,
      goalAlignment: number
    },
    explainability: string[]
  }>
}
```

**Scoring Criteria**:

1. **Skill Match Score (0-100)**
   - Calculate percentage of required skills user already possesses
   - Weight by skill importance for the path
   - Formula: `(matchedSkills / requiredSkills) * 100`

2. **Resource Fit Score (0-100)**
   - Compare required resources (money, time, network) against user's available resources
   - Penalize paths requiring unavailable resources
   - Formula: `100 - (resourceGap * penaltyWeight)`

3. **Timeline Feasibility Score (0-100)**
   - Evaluate if path timeline aligns with user's time constraints
   - Consider user's available hours per week
   - Formula: `min(100, (availableTime / requiredTime) * 100)`

4. **Goal Alignment Score (0-100)**
   - Measure how well path outcomes match user's stated goals
   - Use semantic similarity between path outcomes and goals
   - Formula: `semanticSimilarity(pathOutcomes, userGoals) * 100`

**Total Score Calculation**:
```
totalScore = (skillMatch * w1) + (resourceFit * w2) + 
             (timelineFeasibility * w3) + (goalAlignment * w4)

where w1 + w2 + w3 + w4 = 1 (normalized weights)
```

**Default Weights**:
- Skill Match: 0.30
- Resource Fit: 0.25
- Timeline Feasibility: 0.20
- Goal Alignment: 0.25

### Simulation Engine

**Purpose**: Model future outcomes, risks, and timelines for Decision_Paths

**Input Interface**:
```typescript
{
  profileId: string,
  pathId: string
}
```

**Output Interface**:
```typescript
{
  pathId: string,
  successProbability: number,
  timeline: {
    totalMonths: number,
    milestones: Array<{
      name: string,
      monthOffset: number,
      completionProbability: number
    }>
  },
  riskIndicators: Array<{
    riskType: string,
    severity: 'low' | 'medium' | 'high',
    description: string,
    mitigationSuggestions: string[]
  }>,
  skillGaps: Array<{
    skillName: string,
    currentLevel: number,
    requiredLevel: number,
    impactOnSuccess: number,
    learningTimeEstimate: number
  }>
}
```

**Simulation Logic**:

1. **Success Probability Calculation**:
   - Base probability from historical data for similar paths
   - Adjust for user's skill match: `+10% per 20% skill match above 50%`
   - Adjust for resource availability: `-15% if critical resources missing`
   - Adjust for timeline pressure: `-10% if timeline is aggressive`
   - Clamp final probability between 10% and 90%

2. **Timeline Estimation**:
   - Retrieve standard milestones for path type
   - Adjust milestone timing based on user's available time
   - Add buffer time for skill gaps: `+1 month per major skill gap`
   - Sequence milestones with dependencies

3. **Risk Identification**:
   - Financial risk: Flag if required capital > 50% of user's available funds
   - Skill risk: Flag if critical skills are missing
   - Market risk: Flag if path is in highly competitive or declining market
   - Time risk: Flag if timeline conflicts with user constraints

4. **Skill Gap Analysis**:
   - Compare user's current skills against path requirements
   - Quantify gap severity (1-10 scale)
   - Estimate learning time based on skill complexity
   - Calculate impact on success probability

### Roadmap Generator

**Purpose**: Create weekly execution plans for selected Decision_Path

**Input Interface**:
```typescript
{
  profileId: string,
  pathId: string,
  startDate: string,
  weeklyHoursAvailable: number
}
```

**Output Interface**:
```typescript
{
  pathId: string,
  roadmap: Array<{
    weekNumber: number,
    weekStartDate: string,
    tasks: Array<{
      taskId: string,
      description: string,
      estimatedHours: number,
      priority: 'high' | 'medium' | 'low',
      category: string,
      dependencies: string[]
    }>,
    milestone?: {
      name: string,
      successCriteria: string[]
    }
  }>
}
```

**Generation Logic**:
1. Retrieve Decision_Path and simulation results
2. Identify all required tasks from path requirements and skill gaps
3. Construct LLM prompt with:
   - User context and constraints
   - Path objectives and milestones
   - Skill gaps to address
   - Weekly time availability
4. Call Bedrock to generate task breakdown
5. Parse LLM output into structured tasks
6. Sequence tasks based on:
   - Dependencies (prerequisite tasks)
   - Priority (critical path items first)
   - Time constraints (fit within weekly hours)
7. Insert milestone checkpoints at logical intervals
8. Validate that total estimated hours fit within user's availability

### Decision Orchestrator

**Purpose**: Coordinate multi-service workflows and aggregate results

**Workflow for Complete Analysis**:
1. Receive analysis request with profileId
2. Invoke Decision Engine (scoring)
3. Invoke Simulation Engine in parallel for top 3 paths
4. Wait for all results
5. Aggregate scores, simulations, and rankings
6. Return unified response

**Error Handling**:
- If any service fails, return partial results with error flags
- Implement exponential backoff for retries
- Timeout individual services after 10 seconds

### Data Models

**Decision_Profile**:
```typescript
{
  profileId: string,
  userId: string,
  createdAt: timestamp,
  updatedAt: timestamp,
  userContext: {
    background: string,
    currentSituation: string,
    skills: Array<{
      name: string,
      level: number, // 1-10
      yearsExperience: number
    }>,
    constraints: {
      timeAvailablePerWeek: number,
      financialResources: number,
      geographicConstraints: string[],
      personalConstraints: string[]
    },
    goals: {
      shortTerm: string[],
      longTerm: string[],
      priorities: string[]
    }
  },
  decisionPaths: Array<{
    pathId: string,
    pathName: string,
    pathType: 'career' | 'startup' | 'education',
    description: string,
    requiredSkills: Array<{
      name: string,
      level: number
    }>,
    requiredResources: {
      financialInvestment: number,
      timeCommitment: number,
      networkRequirements: string[]
    },
    expectedOutcomes: string[],
    estimatedTimeline: number // months
  }>
}
```

**ScoringWeights**:
```typescript
{
  skillMatch: number,      // 0-1
  resourceFit: number,     // 0-1
  timelineFeasibility: number, // 0-1
  goalAlignment: number    // 0-1
  // Sum must equal 1.0
}
```

**Decision_Report**:
```typescript
{
  reportId: string,
  profileId: string,
  generatedAt: timestamp,
  summary: {
    topRecommendedPath: string,
    confidenceLevel: number,
    keyInsights: string[]
  },
  pathAnalysis: Array<{
    pathName: string,
    score: number,
    scoreBreakdown: object,
    simulation: object,
    pros: string[],
    cons: string[]
  }>,
  recommendedRoadmap: object,
  s3Url: string
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Context Extraction Properties

**Property 1: Extraction produces valid profiles**
*For any* natural language input describing a user situation, the Context_Extractor should produce a Decision_Profile that conforms to the defined schema.
**Validates: Requirements 1.1, 1.5**

**Property 2: Required fields are extracted or flagged**
*For any* user input, the Context_Extractor should either extract all required fields (skills, goals, constraints, paths) or explicitly identify which critical fields are missing.
**Validates: Requirements 1.2, 1.3**

**Property 3: Missing fields trigger prompts**
*For any* extraction result with missing critical fields, the system should generate specific prompts requesting the missing information.
**Validates: Requirements 1.4**

### Scoring Properties

**Property 4: All paths receive scores**
*For any* Decision_Profile containing Decision_Paths, the Decision_Engine should calculate a numerical score for every path.
**Validates: Requirements 2.1**

**Property 5: Scores include all criteria**
*For any* scored Decision_Path, the score breakdown should include values for skill match, resource fit, timeline feasibility, and goal alignment.
**Validates: Requirements 2.2**

**Property 6: Weight changes affect scores**
*For any* Decision_Profile and two different sets of scoring weights, changing the weights should produce different total scores (unless all paths score identically on all criteria).
**Validates: Requirements 2.3**

**Property 7: Scores are bounded**
*For any* scored Decision_Path, the total score should be in the range [0, 100].
**Validates: Requirements 2.4**

**Property 8: Paths are ranked by score**
*For any* set of scored Decision_Paths, the ranking should be in descending order by total score.
**Validates: Requirements 2.5**

**Property 9: Scores include explainability**
*For any* scored Decision_Path, explainability data showing the calculation breakdown should be present in the output.
**Validates: Requirements 2.6**

### Dashboard and UI Properties

**Property 10: Dashboard displays all paths**
*For any* set of scored Decision_Paths, the dashboard rendering should include all paths with their scores and key metrics (timeline, resources, risk level).
**Validates: Requirements 3.1, 3.2**

**Property 11: Selection reveals details**
*For any* Decision_Path selection in the dashboard, the detailed scoring breakdown should be displayed.
**Validates: Requirements 3.3**

**Property 12: Top path is highlighted**
*For any* set of ranked Decision_Paths, the path with the highest score should be marked as the top recommendation in the dashboard.
**Validates: Requirements 3.4**

**Property 13: Weight adjustment updates rankings**
*For any* dashboard state with scoring weights, adjusting the weights should trigger recalculation and update the displayed rankings.
**Validates: Requirements 3.5**

### Simulation Properties

**Property 14: Simulations include required components**
*For any* Decision_Path simulation, the output should include success probability, timeline with milestones, and risk indicators.
**Validates: Requirements 4.1, 4.2, 4.3, 4.5**

**Property 15: Simulations respond to context changes**
*For any* Decision_Path, changing user constraints or skill levels in the Decision_Profile should affect the simulation results (probability, timeline, or risks).
**Validates: Requirements 4.4**

### Skill Gap Properties

**Property 16: Skill gaps are identified and quantified**
*For any* Decision_Path with required skills, the system should identify skills the user lacks and assign severity levels to each gap.
**Validates: Requirements 5.1, 5.2, 5.3**

**Property 17: Skill gaps are prioritized**
*For any* set of identified Skill_Gaps, they should be ordered by their impact on success probability (highest impact first).
**Validates: Requirements 5.4**

**Property 18: Skill gaps appear in comparison**
*For any* path comparison view, skill gap analysis should be included in the displayed data.
**Validates: Requirements 5.5**

### Roadmap Properties

**Property 19: Roadmaps contain structured tasks**
*For any* selected Decision_Path, the generated roadmap should contain a list of specific, actionable tasks organized by week.
**Validates: Requirements 6.1, 6.2**

**Property 20: Task dependencies are respected**
*For any* roadmap with task dependencies, tasks should be sequenced such that prerequisite tasks appear in earlier weeks than dependent tasks.
**Validates: Requirements 6.3**

**Property 21: Roadmaps include milestones**
*For any* generated roadmap, milestone checkpoints should be present at appropriate intervals.
**Validates: Requirements 6.4**

**Property 22: Roadmaps respect time constraints**
*For any* generated roadmap, the total estimated hours for tasks in each week should not exceed the user's available weekly hours.
**Validates: Requirements 6.5**

**Property 23: Roadmaps are chronologically ordered**
*For any* generated roadmap, weeks should be in ascending chronological order.
**Validates: Requirements 6.6**

### Report Properties

**Property 24: Reports contain complete analysis**
*For any* generated Decision_Report, it should include the Decision_Profile, all evaluated paths with scores, simulation results, recommended roadmap, and explainability sections.
**Validates: Requirements 7.1, 7.2, 7.5**

**Property 25: Reports include visualizations**
*For any* generated Decision_Report, visual representations of comparisons and timelines should be present.
**Validates: Requirements 7.3**

**Property 26: Reports are downloadable**
*For any* generated Decision_Report, it should be available in a downloadable format (PDF or structured document).
**Validates: Requirements 7.4**

### Persistence Properties

**Property 27: Profile persistence round-trip**
*For any* Decision_Profile created and stored, retrieving it by profileId should return an equivalent profile.
**Validates: Requirements 8.1**

**Property 28: User profiles are retrievable**
*For any* user with stored Decision_Profiles, querying by userId should return all profiles associated with that user.
**Validates: Requirements 8.2, 8.4**

**Property 29: Profile updates are persisted**
*For any* Decision_Profile that is updated, subsequent retrieval should reflect the updated values.
**Validates: Requirements 8.3**

**Property 30: Version history is maintained**
*For any* Decision_Profile that is updated, previous versions should remain accessible in the version history.
**Validates: Requirements 8.5**

### UI Responsiveness Properties

**Property 31: Loading indicators appear during processing**
*For any* long-running operation, loading indicators should be displayed in the UI while processing is active.
**Validates: Requirements 9.4**

**Property 32: Delays trigger notifications**
*For any* operation that exceeds expected processing time, a notification should be displayed to the user.
**Validates: Requirements 9.5**

### Explainability Properties

**Property 33: Scores include factor breakdown**
*For any* displayed score, the breakdown of scoring factors and their weights should be shown.
**Validates: Requirements 10.1**

**Property 34: Risks include explanations**
*For any* displayed Risk_Indicator, an explanation of the specific risk should be present.
**Validates: Requirements 10.2**

**Property 35: Skill gaps include importance explanations**
*For any* displayed Skill_Gap, an explanation of why the skill is important for the path should be present.
**Validates: Requirements 10.3**

**Property 36: Timelines include reasoning**
*For any* displayed timeline estimate, reasoning for the estimate and milestone sequence should be provided.
**Validates: Requirements 10.4**

**Property 37: Drill-down reveals underlying logic**
*For any* recommendation in the dashboard, drill-down interaction should reveal the underlying calculation logic and data.
**Validates: Requirements 10.5**

### LLM Integration Properties

**Property 38: Rate limits are handled gracefully**
*For any* LLM API call that encounters a rate limit error, the system should handle it gracefully (retry with backoff or return informative error).
**Validates: Requirements 13.3**

**Property 39: LLM outputs are validated**
*For any* output received from the LLM, it should be validated against expected schema before being used in downstream processing.
**Validates: Requirements 13.5**

**Property 40: Transient failures trigger retries**
*For any* transient LLM service failure, the system should attempt retry with exponential backoff before returning an error.
**Validates: Requirements 13.6**

### Security Properties

**Property 41: Stored data is encrypted**
*For any* Decision_Profile stored in the database, sensitive user data fields should be encrypted.
**Validates: Requirements 14.1**

**Property 42: Users access only their own data**
*For any* user request to retrieve Decision_Profiles, only profiles associated with that user's account should be returned.
**Validates: Requirements 14.2**

**Property 43: Endpoints enforce authorization**
*For any* API endpoint request, authorization checks should be performed before processing the request.
**Validates: Requirements 14.3**

**Property 44: Connections use HTTPS**
*For any* data transmission between client and server, the connection should use HTTPS encryption.
**Validates: Requirements 14.4**

**Property 45: Logs exclude sensitive data**
*For any* error or operation logged by the system, sensitive personal information should not appear in plain text.
**Validates: Requirements 14.5**

### Error Handling Properties

**Property 46: Errors return user-friendly messages**
*For any* error that occurs during processing, the system should return a user-friendly error message (not technical stack traces).
**Validates: Requirements 15.1**

**Property 47: Parse failures explain issues**
*For any* Context_Extractor parsing failure, the error message should explain what information was unclear or missing.
**Validates: Requirements 15.2**

**Property 48: Service failures provide guidance**
*For any* external service failure, the system should provide either fallback behavior or clear guidance to the user.
**Validates: Requirements 15.3**

**Property 49: Error logs are debuggable but secure**
*For any* logged error, the log entry should contain sufficient detail for debugging but should not expose sensitive user data.
**Validates: Requirements 15.4**

**Property 50: Recoverable errors allow retry**
*For any* recoverable error, the system should provide a mechanism for the user to retry the operation.
**Validates: Requirements 15.5**

## Error Handling

### Error Categories

**1. Input Validation Errors**
- Invalid or incomplete user input
- Malformed Decision_Profile data
- Missing required fields

**Handling Strategy**:
- Return 400 Bad Request with specific field errors
- Provide clear guidance on what needs to be corrected
- Preserve partial input to avoid user frustration

**2. LLM Service Errors**
- API rate limiting
- Service timeouts
- Invalid or unparseable LLM responses

**Handling Strategy**:
- Implement exponential backoff retry (3 attempts)
- Timeout after 10 seconds per attempt
- Validate LLM JSON output with schema before use
- Return partial results if extraction partially succeeds
- Provide fallback error message if all retries fail

**3. Database Errors**
- Connection failures
- Write conflicts
- Query timeouts

**Handling Strategy**:
- Retry transient errors (connection issues) up to 3 times
- Return 503 Service Unavailable for persistent database issues
- Implement optimistic locking for concurrent updates
- Log errors with request ID for debugging

**4. Authorization Errors**
- Unauthenticated requests
- Unauthorized access attempts
- Expired tokens

**Handling Strategy**:
- Return 401 Unauthorized for authentication failures
- Return 403 Forbidden for authorization failures
- Do not leak information about resource existence
- Log security events for monitoring

**5. Resource Errors**
- S3 upload failures
- Report generation failures
- Memory or timeout limits exceeded

**Handling Strategy**:
- Retry S3 operations with exponential backoff
- Implement streaming for large reports
- Set Lambda timeout limits and handle gracefully
- Return 500 Internal Server Error with request ID

### Error Response Format

All errors follow consistent JSON structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "User-friendly error message",
    "details": {
      "field": "specific field if applicable",
      "suggestion": "what the user should do"
    },
    "requestId": "unique-request-id-for-support"
  }
}
```

### Logging Strategy

- **Info Level**: Successful operations, key milestones
- **Warn Level**: Retryable errors, degraded performance
- **Error Level**: Failed operations, unrecoverable errors
- **Never Log**: Passwords, tokens, sensitive personal data

All logs include:
- Request ID for tracing
- User ID (hashed for privacy)
- Timestamp
- Service name
- Operation name

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage. These approaches are complementary:

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties across all inputs

Together, they provide comprehensive coverage where unit tests catch concrete bugs and property tests verify general correctness.

### Property-Based Testing

**Framework**: Use `fast-check` for TypeScript/JavaScript components

**Configuration**:
- Minimum 100 iterations per property test
- Each test must reference its design document property
- Tag format: `// Feature: ai-decision-engine, Property {number}: {property_text}`

**Example Property Test Structure**:

```typescript
// Feature: ai-decision-engine, Property 7: Scores are bounded
test('all path scores are in range [0, 100]', () => {
  fc.assert(
    fc.property(
      decisionProfileArbitrary(),
      scoringWeightsArbitrary(),
      (profile, weights) => {
        const scores = decisionEngine.scoreAllPaths(profile, weights);
        return scores.every(s => s.totalScore >= 0 && s.totalScore <= 100);
      }
    ),
    { numRuns: 100 }
  );
});
```

**Generators Needed**:
- `decisionProfileArbitrary()`: Generate valid Decision_Profiles with random skills, constraints, goals, and paths
- `scoringWeightsArbitrary()`: Generate valid weight configurations (sum to 1.0)
- `userInputTextArbitrary()`: Generate realistic natural language descriptions
- `pathArbitrary()`: Generate valid Decision_Path objects

### Unit Testing

**Framework**: Jest for TypeScript/JavaScript

**Focus Areas**:

1. **Specific Examples**:
   - Test known input/output pairs
   - Verify correct behavior for typical cases
   - Document expected behavior through examples

2. **Edge Cases**:
   - Empty inputs
   - Maximum/minimum values
   - Boundary conditions
   - Single-item collections

3. **Error Conditions**:
   - Invalid inputs
   - Service failures
   - Timeout scenarios
   - Malformed data

4. **Integration Points**:
   - API Gateway to Lambda integration
   - Lambda to DynamoDB operations
   - Lambda to Bedrock API calls
   - S3 upload and retrieval

**Example Unit Test**:

```typescript
describe('Decision Engine', () => {
  test('scores path with perfect skill match at 100 for skill component', () => {
    const profile = createProfileWithSkills(['Python', 'AWS', 'React']);
    const path = createPathRequiringSkills(['Python', 'AWS', 'React']);
    
    const result = decisionEngine.scorePath(profile, path);
    
    expect(result.scoreBreakdown.skillMatch).toBe(100);
  });

  test('returns empty array when profile has no paths', () => {
    const profile = createProfileWithNoPaths();
    
    const result = decisionEngine.scoreAllPaths(profile);
    
    expect(result).toEqual([]);
  });
});
```

### Integration Testing

**Scope**: Test complete workflows across multiple services

**Key Workflows to Test**:
1. End-to-end analysis: Input → Extraction → Scoring → Simulation → Report
2. Profile persistence: Create → Store → Retrieve → Update
3. Error recovery: Service failure → Retry → Success
4. Weight adjustment: Change weights → Recalculate → Update UI

**Tools**: 
- AWS SAM Local for local Lambda testing
- DynamoDB Local for database testing
- Mock Bedrock API for LLM testing

### Test Coverage Goals

- **Line Coverage**: Minimum 80%
- **Branch Coverage**: Minimum 75%
- **Property Coverage**: 100% of correctness properties implemented as tests
- **Critical Path Coverage**: 100% of user-facing workflows

### Continuous Testing

- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests before deployment
- Monitor production errors and add regression tests
