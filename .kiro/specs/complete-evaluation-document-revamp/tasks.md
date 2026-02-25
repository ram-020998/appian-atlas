# Implementation Plan: Complete Evaluation Document Section Revamp

## Overview

Revamp the Complete Evaluation form to add a right-hand panel with "Highlights" and "Documents" tabs. Implementation follows a bottom-up approach: create leaf components first (document card), then container sections (documents tab, highlights tab), then the right panel with tab navigation, and finally wire into the existing form.

## Tasks

- [ ] 1. Create the new document card component
  - [ ] 1.1 Create `AS_GSS_CRD_completeEvalDocumentCard` interface
    - Implement `a!cardLayout` with three-column inner layout: file icon, document info (name + download/size), and download action button
    - Use `AS_GSS_UT_displayDocumentName` for name formatting, `AS_GSS_UT_displayDocumentSize` for size, and `AS_GSS_CO_documentDownloadLink` for the download link
    - Set `preventWrapping: true` and `tooltip` on the document name field for truncation with ellipsis
    - Apply branding via `AS_GSS_BrandingValueByKey` for icon color
    - Accept inputs: `evaluationDocument` (CDT), `i18nData` (Map), `marginAbove` (Text, optional)
    - Handle null `appianDocId` by hiding the download link; handle null `documentName` by falling back to the utility's default
    - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [ ] 2. Create the Documents tab section
  - [ ] 2.1 Create `AS_GSS_SCT_completeEvalDocumentsTab` interface
    - Accept inputs: `evaluationDocuments` (CDT Array), `factorDocumentMappings` (CDT Array), `criteriaId` (Integer), `i18nData` (Map)
    - Use `a!localVariables` to filter documents into four groups by `docTypeId`: Factor (`cons!AS_GSS_REF_ID_DOC_TYPE_FACTOR`), Reference (`cons!AS_GSS_REF_ID_DOC_TYPE_REFERENCE`), Vendor (`cons!AS_GSS_REF_ID_DOC_TYPE_VENDOR`), Evaluator (`cons!AS_GSS_REF_ID_DOC_TYPE_EVALUATOR`)
    - For each group, conditionally render a section header (i18n label via `AS_CO_I18N_UT_displayLabel`) and `a!forEach` of `AS_GSS_CRD_completeEvalDocumentCard` only when `length > 0`
    - When total documents is zero, render `AS_GSS_CPS_emptyStateForDocuments`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_


- [ ] 3. Create the Highlights tab section
  - [ ] 3.1 Create `AS_GSS_SCT_completeEvalHighlightsTab` interface
    - Accept inputs: `evaluationId` (Integer), `vendorId` (Integer), `criteriaId` (Integer), `i18nData` (Map)
    - Implement placeholder content displaying evaluation highlights for the current context
    - Show empty state message (i18n key `lbl_NoHighlightsAvailable`) when no highlights are available
    - _Requirements: 5.1, 5.2_

- [ ] 4. Create the right panel container with tab navigation
  - [ ] 4.1 Create `AS_GSS_CPS_completeEvaluationRightPanel` interface
    - Accept inputs: `evaluationId` (Integer), `vendorId` (Integer), `criteriaId` (Integer), `evaluationDocuments` (CDT Array), `factorDocumentMappings` (CDT Array), `i18nData` (Map)
    - Use `a!localVariables` with `local!selectedTab: 1` (1 = Highlights, 2 = Documents)
    - Build tab navigation bar using `a!columnsLayout` with two `a!columnLayout` columns, each containing `a!richTextDisplayField` with `a!dynamicLink` that saves the tab index into `local!selectedTab`
    - Style active tab with `style: "STRONG"` and `color: "ACCENT"`, inactive with `style: "PLAIN"` and `color: "SECONDARY"`
    - Use i18n labels (`lbl_Highlights`, `lbl_Documents`) for tab text
    - Conditionally render tab content using `if(local!selectedTab = 1, highlightsTab, documentsTab)`
    - Wire to `AS_GSS_SCT_completeEvalHighlightsTab` and `AS_GSS_SCT_completeEvalDocumentsTab`
    - _Requirements: 1.2, 1.3, 1.4_

- [ ] 5. Checkpoint - Verify new components in isolation
  - Ensure all new interfaces (`AS_GSS_CRD_completeEvalDocumentCard`, `AS_GSS_SCT_completeEvalDocumentsTab`, `AS_GSS_SCT_completeEvalHighlightsTab`, `AS_GSS_CPS_completeEvaluationRightPanel`) render correctly with test data. Ask the user if questions arise.

- [ ] 6. Integrate right panel into the Complete Evaluation form
  - [ ] 6.1 Modify `AS_GSS_FM_completeEvaluation` to use two-column layout
    - Wrap existing form content in `a!columnsLayout` with two columns
    - Left column (`width: "WIDE"`) contains all existing child interfaces: `AS_GSS_CPS_completeEvaluationHeader`, `AS_GSS_CPS_evaluationResponses`, `AS_GSS_CPS_finalRatingsForCompleteEvaluation`, `AS_GSS_CPS_ratingJustification`, `AS_GSS_CPS_evalDocsForCompleteEvaluation`
    - Right column (`width: "NARROW"`) contains `AS_GSS_CPS_completeEvaluationRightPanel`
    - Pass evaluation context (evaluationId, vendorId, criteriaId, evaluationDocuments, factorDocumentMappings, i18nData) to the right panel
    - Ensure existing form functionality is fully preserved in the left column
    - _Requirements: 1.1, 1.5, 4.1, 4.2, 4.3_

  - [ ] 6.2 Add error handling for right panel data loading
    - Wrap right panel rule call in error handling so that if the right panel fails to load, the left column remains fully functional
    - Display a user-friendly error message in the right panel area on failure
    - _Requirements: 4.4_

- [ ] 7. Add internationalization keys
  - [ ] 7.1 Add translation keys to the i18n bundle
    - Add keys: `lbl_Highlights`, `lbl_Documents`, `lbl_FactorDocuments`, `lbl_ReferenceDocuments`, `lbl_VendorDocuments`, `lbl_EvaluatorDocuments`, `lbl_NoHighlightsAvailable`
    - Ensure `AS_GSS_CO_UT_loadBundleFromFolder` loads the updated bundle in the form
    - _Requirements: 4.3_

- [ ] 8. Final checkpoint - Regression and integration verification
  - Verify tab navigation switches correctly between Highlights and Documents. Verify default tab is Highlights on form load. Verify document cards display name, size, and download link. Verify empty state renders when no documents exist. Verify all existing left-column form functionality (header, responses, ratings, justification, upload) works after the layout change. Ensure all tests pass, ask the user if questions arise.

## Notes

- This is an Appian SAIL project — all components are Appian interface objects, not code files
- The existing `AS_GSS_CRD_displayEvaluationDocument` is not modified; a new card component is created to avoid impacting other forms
- The document upload section (`AS_GSS_CPS_evalDocsForCompleteEvaluation`) stays in the left column — the right panel Documents tab is read-only
- The Highlights tab is a placeholder for future content; the primary focus of this story is the Documents tab and card redesign
- Follow the Consensus Form pattern (`AS_GSS_CPS_consensusFormRighPanel`) as the reference implementation for tab navigation
