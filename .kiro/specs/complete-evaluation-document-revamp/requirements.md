# Requirements Document

## Introduction

This feature revamps the Complete Evaluation form (`AS_GSS_FM_completeEvaluation`) to include a right-hand panel with tabbed navigation containing "Highlights" and "Documents" tabs. The Documents tab displays evaluation documents using a new card-based design. This follows the established right-panel pattern from the Consensus Form and reuses existing document display components where possible.

## Glossary

- **Complete_Evaluation_Form**: The Appian SAIL task interface (`AS_GSS_FM_completeEvaluation`) used by evaluators to complete their evaluation, including rating justifications and document review.
- **Right_Panel**: A new right-hand section (~35% width) added to the Complete Evaluation Form containing tabbed content for highlights and documents.
- **Tab_Navigation**: A horizontal tab bar within the Right Panel allowing users to switch between "Highlights" and "Documents" views.
- **Highlights_Tab**: The tab displaying evaluation highlights, findings, and contextual information relevant to the current evaluation factor and vendor.
- **Documents_Tab**: The tab displaying evaluation-related documents organized by document type sections (Factor, Reference, Vendor, Evaluator).
- **Document_Card**: A styled card component displaying a single document's metadata (name, size, type) with a download action, following the new design mockup.
- **Document_Section**: A grouping of documents by type category (e.g., Factor Documents, Reference Documents) within the Documents Tab.
- **Evaluator**: A user assigned to complete an evaluation task in the Source Selection (GAM) system.
- **Evaluation_Document**: A record from `AS_GSS_EvaluationDocument_SYNCEDRECORD` representing a file associated with an evaluation.
- **Document_Type**: A classification constant identifying the category of a document (Factor, Reference, Vendor, Evaluator, Consensus, Recommendation).

## Requirements

### Requirement 1: Right-Hand Panel with Tabbed Navigation

**User Story:** As an evaluator, I want to see a right-hand section with "Highlights" and "Documents" tabs when completing an evaluation, so that I can quickly access relevant information and documents while working.

#### Acceptance Criteria

1. WHEN an Evaluator opens the Complete_Evaluation_Form, THE Right_Panel SHALL be visible on the right side of the form occupying approximately 35% of the form width.
2. WHEN the Right_Panel is displayed, THE Tab_Navigation SHALL show two tabs labeled "Highlights" and "Documents".
3. WHEN the Complete_Evaluation_Form loads, THE Tab_Navigation SHALL default to the "Highlights" tab as the active tab.
4. WHEN an Evaluator clicks on a tab in the Tab_Navigation, THE Right_Panel SHALL display the content corresponding to the selected tab and visually indicate the active tab.
5. WHEN the Right_Panel is displayed, THE Complete_Evaluation_Form SHALL continue to display the existing form content (header, evaluation responses, final ratings, rating justification) in the left section occupying approximately 65% of the form width.

### Requirement 2: Documents Tab Content Display

**User Story:** As an evaluator, I want to view all relevant evaluation documents organized by type when I select the Documents tab, so that I can reference supporting materials during my evaluation.

#### Acceptance Criteria

1. WHEN an Evaluator selects the Documents_Tab, THE Documents_Tab SHALL query and display all Evaluation_Documents associated with the current evaluation.
2. WHEN Evaluation_Documents are loaded, THE Documents_Tab SHALL organize documents into Document_Sections grouped by Document_Type (Factor, Reference, Vendor, Evaluator).
3. WHEN a Document_Section contains documents, THE Documents_Tab SHALL display the Document_Type label as a section header followed by the Document_Cards for that section.
4. WHEN a Document_Section contains zero documents, THE Documents_Tab SHALL hide that Document_Section entirely.
5. WHEN the Documents_Tab contains zero Evaluation_Documents across all types, THE Documents_Tab SHALL display an empty state message indicating no documents are available.

### Requirement 3: Document Card Redesign

**User Story:** As an evaluator, I want documents displayed in a clean, modern card layout, so that I can quickly identify and access the documents I need.

#### Acceptance Criteria

1. THE Document_Card SHALL display the document name, document size, and document type for each Evaluation_Document.
2. WHEN an Evaluator clicks the download action on a Document_Card, THE Document_Card SHALL initiate a download of the associated Evaluation_Document.
3. THE Document_Card SHALL use the styling defined in the new design mockup, including card borders, spacing, and typography consistent with the application branding via `AS_GSS_BrandingValueByKey`.
4. WHEN the document name exceeds the available display width, THE Document_Card SHALL truncate the name with an ellipsis and display the full name on hover.

### Requirement 4: Layout Integration and Existing Content Preservation

**User Story:** As an evaluator, I want the new right-hand panel to integrate seamlessly with the existing evaluation form, so that my current workflow is not disrupted.

#### Acceptance Criteria

1. WHEN the Complete_Evaluation_Form is restructured to a two-column layout, THE Complete_Evaluation_Form SHALL preserve all existing functionality of the header, evaluation responses, final ratings, and rating justification sections in the left column.
2. WHEN the Right_Panel is added, THE Complete_Evaluation_Form SHALL pass the evaluation context (evaluation ID, vendor ID, criteria ID) to the Right_Panel for data retrieval.
3. THE Complete_Evaluation_Form SHALL load internationalization labels for the Right_Panel using `AS_GSS_CO_UT_loadBundleFromFolder` consistent with the existing i18n pattern.
4. IF the Right_Panel encounters a data loading error, THEN THE Right_Panel SHALL display a user-friendly error message and the left column form content SHALL remain fully functional.

### Requirement 5: Highlights Tab Content

**User Story:** As an evaluator, I want to see evaluation highlights and contextual information in the Highlights tab, so that I have quick access to key findings while completing my evaluation.

#### Acceptance Criteria

1. WHEN an Evaluator selects the Highlights_Tab, THE Highlights_Tab SHALL display evaluation highlights relevant to the current evaluation context (factor and vendor).
2. WHEN the Highlights_Tab contains zero highlights, THE Highlights_Tab SHALL display an empty state message indicating no highlights are available.
