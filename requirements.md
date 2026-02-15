# Requirements Document: WordTrace

## Introduction

WordTrace is an AI-assisted writing transparency platform designed for Indian higher-education environments. The system addresses the challenge of evaluating genuine student learning in the age of generative AI tools by observing and documenting how students create their work, rather than just evaluating final outputs. WordTrace captures writing activity, revision behavior, research usage, and citation validation to generate evidence-based "Proof of Effort" reports that demonstrate authentic learning processes.

## Relevance to AI for Bharat

India's higher-education ecosystem serves millions of students, where scalable and fair evaluation is critical. With rapid adoption of generative AI tools, institutions face challenges in maintaining academic integrity without unfairly penalizing genuine learners. WordTrace addresses this by enabling transparent, process-based evaluation rather than detection-based judgment, aligning with India's need for responsible AI adoption in education.

## Hackathon MVP Scope

For the AI for Bharat Hackathon, WordTrace will be implemented as a functional prototype demonstrating:

- Writing event capture and session tracking
- Timeline replay of document evolution
- Contribution heatmap visualization
- Basic citation validation using public APIs
- Proof-of-Effort report generation
- Lightweight similarity analysis using pre-trained models

This MVP validates the concept without requiring large-scale infrastructure or model training.

## Glossary

- **System**: The WordTrace platform
- **Student**: A user who creates written content within the platform
- **Educator**: A user who reviews Proof-of-Effort reports and evaluates student work
- **Writing_Session**: A continuous period of writing activity by a student
- **Writing_Event**: A discrete action taken by a student (edit, revision, paste, delete)
- **Observer_Agent**: AI component that tracks and logs writing events
- **Citation_Agent**: AI component that validates referenced sources
- **Reasoning_Agent**: AI component that aggregates signals and produces confidence summaries
- **Evidence_Agent**: AI component that generates visualizations and reports
- **Proof_of_Effort_Report**: A document summarizing writing activity, revisions, and authenticity signals
- **Contribution_Heatmap**: A visual representation of writing effort over time
- **Writing_Timeline**: A chronological replay of content evolution
- **Citation_Record**: Metadata about a referenced source including validation status
- **Activity_Metadata**: Structured data about writing events (timestamps, edit types, session duration)
- **Similarity_Score**: A numerical measure of text similarity using NLP models
- **Scholarly_API**: External service for validating academic citations

## Requirements

### Requirement 1: Writing Workspace

**User Story:** As a student, I want a distraction-free writing workspace, so that I can focus on creating my content while the system transparently tracks my activity.

#### Acceptance Criteria

1. THE System SHALL provide a rich text editor interface for content creation
2. WHEN a student opens the workspace, THE System SHALL initialize a new Writing_Session with a unique identifier
3. WHEN a student types, edits, or formats content, THE System SHALL capture these actions as Writing_Events
4. THE System SHALL support standard text formatting operations (bold, italic, headings, lists, links)
5. WHEN a Writing_Session exceeds 30 minutes of inactivity, THE System SHALL automatically close the session

### Requirement 2: Event Logging

**User Story:** As the system, I want to log writing activity as structured metadata, so that I can build an evidence trail without invasive keystroke surveillance.

#### Acceptance Criteria

1. WHEN a Writing_Event occurs, THE Observer_Agent SHALL record the event type, timestamp, and affected content range
2. THE Observer_Agent SHALL log paste operations with content length and timestamp
3. THE Observer_Agent SHALL log deletion operations with deleted content length and timestamp
4. THE Observer_Agent SHALL log revision operations with before and after content snapshots
5. THE Observer_Agent SHALL NOT record individual keystrokes or raw keystroke data
6. WHEN a Writing_Session ends, THE Observer_Agent SHALL persist all Activity_Metadata to the database
7. THE System SHALL store Activity_Metadata in a structured format (JSON) for efficient querying

### Requirement 3: Writing Timeline Replay

**User Story:** As an educator, I want to replay how a document evolved over time, so that I can understand the student's writing process and identify genuine effort.

#### Acceptance Criteria

1. WHEN an educator requests a Writing_Timeline, THE Evidence_Agent SHALL retrieve all Writing_Events for the document
2. THE Evidence_Agent SHALL order Writing_Events chronologically by timestamp
3. THE System SHALL display content snapshots at key revision points
4. WHEN an educator plays the timeline, THE System SHALL animate content changes in sequence
5. THE System SHALL allow educators to pause, rewind, and fast-forward through the timeline
6. THE System SHALL highlight added content in one color and deleted content in another color

### Requirement 4: Contribution Heatmap

**User Story:** As an educator, I want a visual summary of writing effort over time, so that I can quickly assess activity patterns and identify potential anomalies.

#### Acceptance Criteria

1. WHEN an educator requests a Contribution_Heatmap, THE Evidence_Agent SHALL aggregate Writing_Events by date and time
2. THE Evidence_Agent SHALL calculate effort intensity based on event frequency and content volume
3. THE System SHALL display the heatmap as a calendar grid with color-coded intensity levels
4. THE System SHALL show session duration and word count for each day
5. WHEN an educator clicks on a heatmap cell, THE System SHALL display detailed activity for that time period

### Requirement 5: Citation Verification

**User Story:** As an educator, I want to verify that cited sources actually exist and are accessible, so that I can confirm students are using legitimate references.

#### Acceptance Criteria

1. WHEN a student adds a citation to their document, THE System SHALL extract the citation metadata (title, author, year, DOI/URL)
2. THE Citation_Agent SHALL query Scholarly_APIs to validate the citation exists
3. WHEN a citation is validated, THE Citation_Agent SHALL store the validation result and timestamp
4. WHEN a citation cannot be validated, THE Citation_Agent SHALL mark it as unverified and include the reason
5. THE System SHALL display citation validation status inline within the document
6. THE Citation_Agent SHALL support validation for academic papers, books, and web sources
7. WHEN a Scholarly_API is unavailable, THE Citation_Agent SHALL retry with exponential backoff up to 3 attempts

### Requirement 6: Proof-of-Effort Report Generation

**User Story:** As a student, I want to generate a comprehensive report of my writing process, so that I can submit evidence of authentic effort alongside my final document.

#### Acceptance Criteria

1. WHEN a student requests a Proof_of_Effort_Report, THE Evidence_Agent SHALL compile all Activity_Metadata for the document
2. THE Evidence_Agent SHALL include the Writing_Timeline visualization in the report
3. THE Evidence_Agent SHALL include the Contribution_Heatmap in the report
4. THE Evidence_Agent SHALL include citation validation results in the report
5. THE Evidence_Agent SHALL calculate and display total writing time, session count, and revision count
6. THE Evidence_Agent SHALL generate the report in PDF format
7. THE System SHALL allow students to download the report for submission

### Requirement 7: AI-Assisted Authenticity Insights

**User Story:** As an educator, I want AI-generated insights about writing authenticity, so that I can make informed evaluations without relying on punitive AI detection.

#### Acceptance Criteria

1. WHEN generating a Proof_of_Effort_Report, THE Reasoning_Agent SHALL analyze writing patterns for authenticity signals
2. THE Reasoning_Agent SHALL calculate Similarity_Scores between consecutive revisions using sentence-transformers
3. WHEN large content blocks appear suddenly (>200 words in one paste), THE Reasoning_Agent SHALL flag this as a potential external source
4. THE Reasoning_Agent SHALL identify gradual content evolution as a positive authenticity signal
5. THE Reasoning_Agent SHALL aggregate signals into a confidence summary with explanations
6. THE Reasoning_Agent SHALL NOT make binary "AI-generated" or "human-written" determinations
7. THE System SHALL present insights as supporting evidence, not definitive judgments

## Non-Functional Requirements (MVP Considerations)

To ensure feasibility within a hackathon environment, the prototype will follow these principles:

1. Privacy-first design: The system will store only structured activity metadata and avoid recording raw keystrokes.
2. Lightweight AI usage: Pre-trained NLP models will be used without custom training to maintain speed and simplicity.
3. Modular architecture: Agents will communicate through an event-driven workflow to allow future extensibility.
4. Secure data handling: Basic authentication and role-based access will ensure students can access only their own data.
5. Deployability: The prototype will be container-ready for easy local deployment and demonstration.
6. Performance scope: The MVP will support small classroom-scale usage suitable for validation and testing.
7. Export capability: Students will be able to download their document and Proof-of-Effort report for submission purposes.
