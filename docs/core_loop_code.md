```
FUNCTION FactCheckSystem_ProcessParagraph(text_paragraph):
    // Stores the error candidate that has the highest confidence of being incorrect found so far.
    // The final selection of the "most fatal" error and the definitive stopping strategy are deferred.
    most_confident_error_candidate = NULL 
    // Example structure for most_confident_error_candidate:
    // {
    //   claim_text: "The capital of France is Berlin.",
    //   status: "INCORRECT",
    //   confidence_of_incorrectness: 0.95,
    //   supporting_evidence: [ {source: "...", snippet: "..."} ],
    //   verification_details: "Evidence clearly states Paris is the capital."
    // }

    // A list to store all claims that have been identified as factually incorrect, along with their details.
    // This could be used later if the strategy involves selecting from multiple found errors.
    all_identified_errors = [] 

    // A set to keep track of claims that have already been extracted and processed to avoid redundancy.
    processed_claims_log = new Set()

    // --- MODULE 1: Claim Extraction ---
    // This module is responsible for extracting individual, verifiable claims from the text_paragraph.
    // It should ideally yield unique claims one by one.
    claim_extractor = InitializeClaimExtractor(text_paragraph)

    WHILE claim_extractor.hasMoreClaims():
        current_claim_text = claim_extractor.getNextClaim()

        IF current_claim_text IS IN processed_claims_log:
            CONTINUE // Skip if already processed
        END IF
        
        processed_claims_log.add(current_claim_text)
        current_claim_processed_successfully_by_all_modules = TRUE // Flag

        // --- MODULE 2: Evidence Retrieval ---
        // For the current_claim_text, find supporting or refuting evidence.
        // Returns a list of evidence items (e.g., documents, snippets, URLs).
        retrieved_evidence_list = Module_RetrieveEvidence(current_claim_text)

        IF retrieved_evidence_list IS EMPTY_OR_NULL:
            // Cannot proceed for this claim without evidence.
            // Log this claim as "Evidence Not Found" or similar if needed.
            CONTINUE 
        END IF

        // --- MODULE 3: Evidence-Claim Relevance Analysis ---
        // From retrieved_evidence_list, determine which pieces are actually relevant to current_claim_text.
        // Returns a collection of relevant_evidence_items and a boolean indicating if any were found.
        (is_any_evidence_relevant, relevant_evidence_collection) = Module_AssessEvidenceRelevance(current_claim_text, retrieved_evidence_list)

        IF is_any_evidence_relevant IS FALSE:
            // No relevant evidence found to verify this claim.
            // Log this claim as "No Relevant Evidence Found" or similar if needed.
            CONTINUE
        END IF

        // --- MODULE 4: Claim Verification & Confidence Scoring ---
        // Based on relevant_evidence_collection, verify the current_claim_text.
        // This module will output:
        //   - verification_status: "CORRECT", "INCORRECT", "UNVERIFIABLE" / "NEEDS_MORE_INFO"
        //   - confidence_score: A numeric value (e.g., 0.0 to 1.0) indicating the system's certainty in the verification_status.
        //   - verification_details: Explanation, links to specific evidence used for judgment.
        (verification_status, confidence_score, verification_details_for_claim) = Module_VerifyClaimAndScoreConfidence(current_claim_text, relevant_evidence_collection)

        IF verification_status IS "INCORRECT":
            // An error has been identified.
            current_error_details = {
                claim_text: current_claim_text,
                status: "INCORRECT",
                confidence_of_incorrectness: confidence_score,
                supporting_evidence: relevant_evidence_collection, // Or specific evidence from verification_details
                verification_details: verification_details_for_claim
            }
            all_identified_errors.add(current_error_details)

            // Update the most_confident_error_candidate if this error is more confidently incorrect.
            IF most_confident_error_candidate IS NULL OR confidence_score > most_confident_error_candidate.confidence_of_incorrectness:
                most_confident_error_candidate = current_error_details
            END IF
        ELSE IF verification_status IS "CORRECT":
            // The claim is verified as correct.
            // Log or handle as per system requirements (e.g., for overall passage assessment).
            PRINT "Claim: '" + current_claim_text + "' verified as CORRECT with confidence: " + confidence_score
        ELSE IF verification_status IS "UNVERIFIABLE":
            // The claim could not be definitively verified as correct or incorrect with the available evidence.
            PRINT "Claim: '" + current_claim_text + "' is UNVERIFIABLE with confidence: " + confidence_score
        END IF

        // The undefined stopping strategy would be implemented around here or after the loop.
        // For now, the loop continues as long as claims are available.

    END WHILE // End of processing claims from claim_extractor

    // --- Output Generation (Simplified) ---
    // The exact output depends on the (deferred) stopping strategy and reporting requirements.
    // For now, if any confident error was found, we can indicate it.
    // Otherwise, indicate that no definitive errors were found based on the processing.

    IF most_confident_error_candidate IS NOT NULL:
        // This is the error with the highest confidence of incorrectness found during the run.
        // Further logic (deferred) would decide if this is the one to report,
        // or if another from 'all_identified_errors' is more "fatal" based on other criteria.
        RETURN most_confident_error_candidate 
    ELSE IF all_identified_errors IS NOT EMPTY:
        // This case might occur if errors were found but none met a certain threshold
        // to become the 'most_confident_error_candidate' under specific logic not yet defined.
        // Or, if we just return the list for later processing.
        RETURN "Multiple errors identified. Highest confidence error details: [Details of the one with highest confidence from all_identified_errors]" // Or return all_identified_errors
    ELSE:
        RETURN "No definitive factual errors identified in the processed claims."
    END IF

END FUNCTION


// --- Placeholder definitions for the Modules (to be implemented) ---

FUNCTION InitializeClaimExtractor(text_paragraph):
    // INPUT: Raw text paragraph.
    // ACTION: Parses the text, identifies potential factual claims.
    //         Manages an internal state to yield claims one by one via hasMoreClaims() and getNextClaim().
    //         Should ideally handle basic claim normalization and near-duplicate detection.
    // OUTPUT: An iterator or generator object for claims.
END FUNCTION

FUNCTION Module_RetrieveEvidence(claim_text_string):
    // INPUT: A single string representing the claim.
    // ACTION: Constructs search queries from the claim.
    //         Interacts with external knowledge sources (search engine APIs, databases, etc.).
    // OUTPUT: A list of evidence items (e.g., URLs, text snippets, document IDs).
END FUNCTION

FUNCTION Module_AssessEvidenceRelevance(claim_text_string, list_of_raw_evidence):
    // INPUT: The claim string and the list of raw evidence items.
    // ACTION: For each evidence item, determines if it is semantically relevant to the claim.
    //         May use NLP techniques like semantic similarity, NLI pre-screening.
    // OUTPUT: A tuple: (boolean_is_any_relevant, collection_of_relevant_evidence_items).
END FUNCTION

FUNCTION Module_VerifyClaimAndScoreConfidence(claim_text_string, collection_of_relevant_evidence):
    // INPUT: The claim string and a collection of evidence confirmed to be relevant.
    // ACTION: Performs the core fact-checking logic.
    //         Compares the claim against the relevant evidence.
    //         Determines if the claim is supported, refuted, or if evidence is insufficient.
    //         Crucially, calculates a confidence score for this determination.
    //         This is a key place where LLMs (like GPT or Gemini) would be heavily utilized for their
    //         reasoning, NLI, and summarization capabilities to interpret evidence against the claim.
    // OUTPUT: A tuple: (string_verification_status ["CORRECT", "INCORRECT", "UNVERIFIABLE"], 
    //                  float_confidence_score, 
    //                  object_or_string_verification_details).
END FUNCTION
```