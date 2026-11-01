# AI Invoice Automation with n8n

Automatically extract invoice data from text-based PDFs uploaded to Google Drive, check for an existing invoice, and record new invoices in Google Sheets.

## Workflow

1. **Google Drive Trigger** checks a selected folder every minute for newly created files.
2. **Download file** downloads the file using its trigger ID.
3. **Extract from File** extracts the PDF text.
4. **Information Extractor**, connected to an **OpenRouter Chat Model**, extracts seven structured fields.
5. **Get row(s) in sheet1** searches Google Sheets using invoice number AND supplier name.
6. **If1** checks whether the returned item contains `Numero_Facture`: true stops; false appends a row.

The lookup has **Always Output Data** enabled so an empty result can reach the IF node.

## Extracted fields

| Column | Meaning |
| --- | --- |
| Fournisseur | Supplier name |
| Numero_Facture | Invoice number |
| Date | Invoice date in YYYY-MM-DD format |
| Montant_HT | Total excluding tax |
| TVA | Monetary tax amount, not the percentage |
| Total_TTC | Total including tax |
| Devise | Currency stated on the invoice |
| Lien_Facture | Link to the source file in Drive |
| Statut | `À vérifier`, for human review |

## Setup

1. Import `workflow.json` into n8n. It is intentionally inactive.
2. Configure your Google Drive OAuth2, Google Sheets OAuth2 and OpenRouter credentials in the relevant nodes.
3. Create a Drive folder for incoming invoice PDFs. In **Google Drive Trigger**, replace `YOUR_GOOGLE_DRIVE_FOLDER_ID` with its ID.
4. Create a Google spreadsheet and import `sheet-template.csv` into a tab named **Feuille 1**. Keep the headers exactly as provided.
5. In BOTH Google Sheets nodes, replace `YOUR_GOOGLE_SHEETS_DOCUMENT_ID` with your document ID and select your tab if necessary.
6. Select an available model in **OpenRouter Chat Model**. The export does not explicitly specify a model. Its maximum output token setting is 1000; costs and access depend on your OpenRouter account.
7. Confirm the lookup combines the two filters with **AND**, and that **Always Output Data** remains enabled.
8. Test the full workflow, then publish or activate it in your n8n instance.

Use a dedicated folder containing supported PDFs. This workflow does not include a file-type filter.

## Demo and verification

`examples/facture-test-006.pdf` is a fictional invoice with no payment due.

| Field | Expected value |
| --- | --- |
| Fournisseur | Horizon Digital Demo SARL (fictif) |
| Numero_Facture | TEST-2026-006 |
| Date | 2026-11-01 |
| Montant_HT | 2700 |
| TVA | 540 |
| Total_TTC | 3240 |
| Devise | MAD |

During the original configured workflow's manual tests, a new invoice was inserted with these values. A copied invoice then took the true branch without appending a row. Automatic duplicate detection was also confirmed after publication.

To reproduce: upload the example PDF, check that one correct row appears, then create a copy in the watched folder and verify that the row count stays unchanged. Check the execution details for the matching invoice number.

The public template requires your credentials and resource IDs. It has not been run against a new n8n account as part of packaging.

## Scope and limitations

- Designed for PDFs with extractable text. OCR for scanned images is not included.
- AI extraction can be inaccurate. Review every row marked `À vérifier`.
- Duplicate detection uses exact invoice number and supplier matching; supplier whitespace is trimmed on input and insertion. Variations in supplier spelling can bypass matching.
- The schema permits missing values. Supplier `.trim()` requires a string; missing supplier data needs manual handling or an added validation branch.
- Concurrent executions can both pass the lookup before either inserts a row. This is not an atomic uniqueness constraint.
- There is no error notification workflow, retry strategy, or automated accounting validation.
- Invoice text is sent to the model provider configured through OpenRouter.

## Repository files

- `workflow.json`: sanitized n8n template, preserving the original node connections and mappings.
- `sheet-template.csv`: header-only spreadsheet template.
- `examples/facture-test-006.pdf`: fictional test invoice.
- `.gitignore`: excludes local secrets, original exports and private invoice folders.

Credential references, personal Drive/Sheets IDs, cached private URLs and instance metadata have been removed. A stray tab before the amount expression was also removed in this public copy; the running cloud workflow was not modified.
