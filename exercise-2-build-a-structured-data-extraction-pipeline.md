## Exercise 3: Build a Structured Data Extraction Pipeline

**Objective:** Practice using tool use with JSON schemas, validation-retry loops, few-shot examples, nullable fields, and human review routing.

### Scenario

You are building a document extraction system that processes vendor invoices, contracts, and purchase orders. The system must extract structured data, avoid fabricating missing values, validate outputs, and route uncertain cases to human reviewers.

### Steps

1. **Define an extraction tool with a JSON schema.**

   Create a tool such as `extract_document_fields`.

   The schema should include:

   * Required fields, such as `document_type`

   * Optional or nullable fields, such as `payment_terms`

   * Enum fields, such as:

     ```json
     {
       "document_type": ["invoice", "contract", "purchase_order", "other"]
     }
     ```

   * An `other_document_type_detail` field when `document_type` is `"other"`

   * Field-level confidence scores

   * Source evidence snippets for important fields

2. **Use tool choice to enforce structured output.**

   Test three configurations:

   * `tool_choice: "auto"`
   * `tool_choice: "any"`
   * Forced tool choice for `extract_document_fields`

   Observe when Claude may return conversational text versus guaranteed tool use.

3. **Test nullable fields.**

   Use sample documents where some fields are absent.

   Verify that the model returns:

   ```json
   "payment_terms": null
   ```

   instead of inventing a value.

4. **Implement semantic validation.**

   Add validation rules such as:

   * Invoice line items must sum to the subtotal.
   * Subtotal plus tax must equal total.
   * Contract effective date must be before expiration date.
   * Required fields must have supporting evidence.

5. **Build a retry-with-error-feedback loop.**

   When validation fails, send Claude:

   * The original document
   * The failed extraction
   * The exact validation error
   * Instructions to correct only the invalid fields

   Track which failures are fixed by retry and which are not fixable because the source document lacks the information.

6. **Add few-shot examples.**

   Include examples for:

   * An invoice with a table
   * A contract with narrative clauses
   * A purchase order with missing payment terms
   * A document with ambiguous type requiring `"other"` plus detail

   Use the examples to improve consistency on varied document formats.

7. **Design batch processing.**

   Create a plan for processing 100 documents using the Message Batches API.

   Include:

   * `custom_id` for each document
   * How to correlate responses
   * How to resubmit failed documents only
   * How to chunk oversized documents
   * Why batch processing is suitable only when latency is acceptable

8. **Add human review routing.**

   Route documents to human review when:

   * Any critical field has low confidence
   * Source evidence is missing
   * The document contains conflicting values
   * Validation still fails after retry

   Track accuracy by document type and by field, not only aggregate accuracy.

### Success Criteria

* Structured output is enforced through tool use.
* Missing information is represented as `null`, not hallucinated.
* Validation errors are fed back into retries.
* Batch failures are handled by `custom_id`.
* Human review is based on field-level confidence and validation outcomes.

### Domains Reinforced

* Domain 4: Prompt Engineering & Structured Output
* Domain 5: Context Management & Reliability
