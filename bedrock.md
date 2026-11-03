# Testing AWS Bedrock from Postman (with the prompts this codebase uses)

How the backend talks to Bedrock: `internals/aistudio/providers/bedrock/bedrock.go` calls the **Converse** API (`system` blocks, `messages`, `inferenceConfig`, `toolConfig`). Postman should call the same API, so the payloads below are Converse payloads.

Prompts below are copied from the code. Where a prompt is built at runtime, the source file is named.

---

## 1. URL, auth, model

| Thing | Value |
|---|---|
| Method | `POST` |
| URL | `https://bedrock-runtime.{{region}}.amazonaws.com/model/{{modelId}}/converse` |
| Auth header | `Authorization: Bearer {{bedrockToken}}` (Bedrock API key / access token) |
| Content-Type | `application/json` |

- **Model choosing** = the `{{modelId}}` in the URL. Use an ID or inference-profile ID your account has access to, e.g. `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (placeholder: take the exact ID from the Bedrock console, Model access / Inference profiles).
- Region must be the one where that model/profile is enabled (e.g. `us-east-1`).
- Other endpoints: `/converse-stream` (streaming), `/model/{id}/invoke` (raw per-model body, only used by the code for image generation).
- Only `bedrock-runtime.*` is for inference. `bedrock.*` (no `-runtime`) is the control plane (list models/profiles).

Postman environment variables to create: `region`, `modelId`, `bedrockToken`, and for the file payloads `slipPdfBase64` / `docBase64` (base64 of the file, no `data:` prefix).

### Converse request shape

```json
{
  "system":   [ { "text": "system prompt" } ],
  "messages": [ { "role": "user", "content": [ { "text": "..." }, { "document": {} }, { "image": {} } ] } ],
  "inferenceConfig": { "temperature": 0, "maxTokens": 4096 },
  "toolConfig": { "tools": [ { "toolSpec": { "name": "", "description": "", "inputSchema": { "json": {} } } } ],
                  "toolChoice": { "tool": { "name": "" } } }
}
```

- PDF block: `{"document":{"format":"pdf","name":"slips","source":{"bytes":"<base64>"}}}` (name: letters, digits, spaces, hyphens only).
- Image block: `{"image":{"format":"jpeg","source":{"bytes":"<base64>"}}}`.
- Tool call response: `output.message.content[]` contains `{"toolUse":{"toolUseId","name","input":{...}}}` and `stopReason` is `tool_use`. The tool arguments are in `input`.
- Sequential tool calls: append the assistant message you got back, then a user message `{"role":"user","content":[{"toolResult":{"toolUseId":"<id>","content":[{"json":{...}}]}}]}` and POST again.
- The backend's `buildToolConfig` does not set `toolChoice`. The payloads below force it so the test is deterministic. Remove it to match the backend exactly.

### How to use the scripts below

Each request body is `{{requestBody}}`. Paste the script into the request's *Scripts → Pre-request* tab; it builds the JSON, so prompts stay verbatim (no JSON escaping). The prompts sit in template literals; none contain a backtick or `${`.

---

## 2. Inference profiles and reasoning models

### Inference profiles (IDs starting `us.` / `eu.` / `global.`)
- Put the profile ID in the same `{{modelId}}` slot. No other change to any payload.
- Some models reject the bare model ID with "on-demand throughput isn't supported"; the profile ID fixes that.
- The profile's region prefix must match `{{region}}`.
- List what your account can use: `GET https://bedrock.{{region}}.amazonaws.com/inference-profiles` with the same Bearer header (host has no `-runtime`). The backend does this in `ListInferenceProfiles` (`bedrock.go`).

### Reasoning (extended thinking) models
Add to any body:

```json
"additionalModelRequestFields": { "thinking": { "type": "enabled", "budget_tokens": 2000 } }
```

- `maxTokens` must be larger than `budget_tokens`.
- Remove `temperature: 0` (Claude thinking mode rejects it).
- Drop forced `toolChoice` when thinking is on.
- The response then holds `reasoningContent` blocks next to the text.
- This is from the Bedrock API, not this codebase: the backend provider never sends `additionalModelRequestFields`. Check the field name against your model's Bedrock docs page.

---

## 3. Charts (bar / pie): `chart_builder` tool call

Source: `app/nabhikserver/tools/datavisualizer.go` (`DataVisualizationToolcallSchemas`, key `buildCharts`). The Nabhik data-visualizer agent flow is **plan → build**. The plan step is optional; the build step alone gives the PNG chart code. This is the only real sequential tool-call chain in the codebase.

### 3a. Step 1: planner (`data_visualization_planner`)

```js
const system = `You're a data visualization expert that creates structured chart plans. Follow these rules:
			1. For general requests ("show insights", "visualize this"):
			- Create 2-4 chart plans
			- Each plan has a chart type, fields, and description
			- Suggest diverse chart types fitting the data

			2. For specific requests ("make a pie chart of X"):
			- Strictly follow requested chart types/fields
			- Only suggest alternatives if they add clear value

			3. Always include:
			- Clear chart-type justifications
			- Fields that exist in the dataset
			- Meaningful titles and descriptions

			Output structure:
			- Array of chart plans, each with:
			  * Title
			  * Chart type (from enum)
			  * Fields
			  * Chart description (array of strings)
			  * Filename (optional, can be empty)
			Prioritize clarity over quantity. Adapt to user's specificity.`;

const schema = {
  type: "object",
  properties: {
    output: {
      type: "array",
      items: {
        type: "object",
        properties: {
          title: { type: "string", description: "Title of the visualization plan." },
          chartDescription: { type: "array", items: { type: "string", description: "Description(s) of the chart." }, description: "Descriptions for the chart." },
          fields: { type: "array", items: { type: "string", description: "Field names to be used in the visualization." }, description: "List of field names to be used in the visualization." },
          chartMode: { type: "string", enum: ["PLAIN_CHART", "INTERACTIVE_CHART"], description: "Mode of the chart, either interactive or plain." },
          chartType: { type: "string", enum: ["bar","line","pie","scatter","histogram","heatmap","boxplot","bubble","area","candlestick","radar","funnel","treemap","gauge","wordcloud","choropleth","network","parallel"], description: "Type of chart to be used for visualization." },
          filename: { type: "string", description: "Filename for the chart (optional, can be empty)." }
        },
        required: ["title", "chartDescription", "fields", "chartType", "chartMode", "filename"]
      },
      description: "List of visualization plans."
    }
  },
  required: ["output"],
  description: "Visualization planning response schema."
};

const body = {
  system: [{ text: system }],
  messages: [{ role: "user", content: [{ text:
    "Fields: admission_type (string), billing_amount (number). Sample rows:\n" +
    "admission_type,billing_amount\nEmergency,25000.5\nElective,12000\nUrgent,8000\nEmergency,31000\n" +
    "Make a bar chart and a pie chart of billing_amount by admission_type." }] }],
  inferenceConfig: { temperature: 0, maxTokens: 4096 },
  toolConfig: {
    tools: [{ toolSpec: { name: "data_visualization_planner",
      description: "Suggests a data visualization plan based on the provided data fields and suggests chart types with relevant fields.",
      inputSchema: { json: schema } } }],
    toolChoice: { tool: { name: "data_visualization_planner" } }
  }
};
pm.variables.set("requestBody", JSON.stringify(body));
```

### 3b. Step 2: builder (`chart_builder`), the prompt that draws the bar + pie PNG

Send as a new request (or as turn 2 with the planner's `toolUse` / `toolResult` appended).

```js
const system = `
			You are an expert Python visualization engineer. Generate production-ready chart code with these strict requirements:
			IMPORTANT: Follow these rules strictly. Only build png charts. Do not build any other format.
			Flag for parsing outputfile is "--output".

			1. SINGLE OUTPUT: Combine all requested charts into one Python script that produces a single PNG file with subplots
			2. INPUT HANDLING:
			   - Use exactly the chart types requested (bar and pie in this case)
			   - Use only the specified fields (admission_type and billing_amount)
			3. CODE REQUIREMENTS:
			   - Single output file via --output flag. Flag for parsing outputfile is "--output".
			   - Input File is mentioned in user message to accordingly hardcode it in code, only output file will be passed via output flag.
			   - Use matplotlib/seaborn for static PNG output
			   - Use only **external libraries** ("pandas", "matplotlib", "seaborn", "plotly", etc.) in the "dependencies" list — exclude "os", "argparse", etc.
			   - All charts must appear in one figure using subplots
			   - Include proper error handling and data cleaning
			   - Robust error handling (try/except for each operation)
			   - No plt.show() - only savefig()
			4. QUALITY STANDARDS:
			   - Professional styling with clear labels
			   - Memory-efficient pandas operations
			   - Proper figure layout with titles
			STRICT RULES:
			- Only make ONE tool call containing the complete solution
			- Never split charts across multiple responses
			- The output must be an executable Python script
			- Mention all external libraries in the dependencies list
			- Automatic data cleaning (handle NaNs, type conversion)
			-  MUST return valid tool call format

			### Pie Chart Specifics

			- Pie charts **must not** be created in a "make_subplots" grid with "type='xy'".
			- Either:
			- Use "plotly.graph_objects.Figure()" for standalone pie
			- Or use "make_subplots(specs=[[{"type": "domain"}]])"
			`;

const schema = {
  type: "object",
  properties: {
    justification: { type: "string", description: "Justification for the chart reading." },
    code: { type: "string", description: "Python code to build the chart." },
    description: { type: "string", description: "Description of the chart." },
    dependencies: { type: "array", items: { type: "string", description: "List of dependencies required to build the chart." }, description: "List of dependencies required to build the chart." }
  },
  required: ["justification", "code", "description", "dependencies"],
  description: "Chart builder schema."
};

const body = {
  system: [{ text: system }],
  messages: [{ role: "user", content: [{ text:
    "Input file: /data/billing.csv (columns: admission_type, billing_amount). " +
    "Build a bar chart and a pie chart of billing_amount grouped by admission_type." }] }],
  inferenceConfig: { temperature: 0, maxTokens: 8192 },
  toolConfig: {
    tools: [{ toolSpec: { name: "chart_builder", description: "Builds chart after planning",
      inputSchema: { json: schema } } }],
    toolChoice: { tool: { name: "chart_builder" } }
  }
};
pm.variables.set("requestBody", JSON.stringify(body));
```

Result: `output.message.content[0].toolUse.input.code` is the Python script (reads the CSV, writes the PNG to `--output`), plus `dependencies`.

Other chart prompts in the code (swap `system` / `schema` to use them):
- `internals/nabhik/schemas/datavisualizer.go`: `ChartBuilderSysMess` + `ChartBuilderToolSchema` (tool `ChartSuggestTool`, 4-6 suggestions each with `chartCode`), `InteractiveChartBuilderSysMess` (HTML output), `DataVisualizerPlannerSysMess` + `action_planner`.
- `app/nabhikserver/tools/datavisualizer.go`: key `interactiveChartBuilder` (tool `interactive_chart_builder`, HTML via Plotly).

---

## 4. CMS ATM slips: a bundle of 4 slips

**The CMS flow has no tool calls.** It is one call: system prompt + user text + one PDF, with JSON-only output (`OcrJSONResponse: true`; there is no `toolConfig`). Source: `app/financeapp/services/cms-routed.go` (`Analyze`, `promptFor`, `cmsBatchInput`) → `nabhikMultiPageOcrCall` in `cms-poc.go` → `internals/nabhikagents/ocr/v1beta1/helper.go` (`GeneralCallWithUsages`, `Temperature: 0`).

A "bundle of 4 slips" is one PDF **page**: a grid of up to 4 slip photos (quadrants 1-4), each stamped with a filename `PREFIX_IFSC_DOCID_PAGE.jpg`. Several pages go in one request (`DisablePdfBatching: true`) and the model returns `{"receipts":[...]}` with one object per page.

| Part | Value |
|---|---|
| system | prompt built per batch (below) |
| user text | `Extract the N receipt sets in the attached PDF as the system prompt specifies.` (N = pages) |
| file | the grid PDF |
| temperature | 0 |

The system prompt has two forms:
1. **Template route** (ATM master gives the cassette + audit type): `BuildCmsRoutedBatchPrompt` in `internals/applications/financeapp/services/poc/cms-routing-templates.go` = `cmsRoutedCoreDirectives` + the route's two dictionary lines + `cmsRoutedRules` + a `confidenceReasons` line + `CmsFamilyKeyContract(keys, false)`. The dictionary lines depend on the ATM master, so they can't be shown without a route.
2. **Fallback** (route unknown): `BuildCmsGeminiPromptForIFSC(ifscBlock)` + `"\n\n"` + `CmsFamilyKeyContract(keys, true)`. Fully derivable from the IFSC, so the example below is exactly this prompt for **IFSC `SBIN0001234`, 1 page, family key `ATM1_SBIN0001234_DOC1`** (SBIN → template `T3A`). For another bank, change the `Bank/Template`, `BANK DICTIONARY`, `AUDIT ALIASES` and `TEMPLATE MAPPING` sections using `internals/applications/financeapp/services/poc/cms-gemini.go`.

### 4a. Bundle of 4 slips (one grid page)

Env var `slipPdfBase64` = base64 of the grid PDF.

```js
const system = `Transcribe ATM cash-management slips to JSON. Copy printed digits exactly (strip RS/commas); no arithmetic, no guessed values.
RULES:
1. Cassette slip (note COUNTS per slot/type) -> beforeCassette/afterCassette. Audit slip (rupee AMOUNTS / transaction totals) -> beforeAudit/afterAudit. Never cross-assign. Exception: PRINT MEDIA / EPSBANCS slips hold note counts but map to audit.
2. Handwriting written ON a printed slip (figures, overwriting, notes; not signatures, stamps or ticks): still transcribe the printed values and add top-level "fraud":"<quadrant digits>: <what is handwritten, max 8 words>". Only pen marks count: overwritten or corrected digits, handwritten numbers, strikethroughs, sums written next to printed values. NOT fraud: printed text of another slip showing in the photo (fixed-width print on its own straight lines, or printed labels such as C2 / IC / CASH STATUS / MACHINE), a bank logo or footer over the numbers, a thumb, a fold, faded print, a cropped edge, the same slip photographed twice.
3. Cassette slips: before = the slip with DISPENSED > 0 (its LAST CLEARED is days earlier); after = the slip whose DISPENSED is 0 on every row (its LAST CLEARED is minutes before its own time). Use the printed time only to break a tie. Never photo or quadrant order.
4. cassetteType = the printed row / TYPE / slot number (1,2,3).
5. denomination: a printed face value (100, 200, 500), or the printed cassette code in the CUR/TYPE/DENO column copied as printed ("0002", "0003", "0005", "0500"): the backend maps codes. Never infer it from row position.
6. dateAndTime = the slip's own printed header date and time, copied as printed. Never LAST CLEARED, never the photo overlay ("Date: ... Lat: ... ATM ID: ..."); "NOT_PRINTED" if the slip has none or it is hidden.
7. One image may hold several slips photographed together, or every quadrant may repeat the same photo: extract each distinct slip once. The same slip photographed twice is ONE slip.
8. Several slips of one kind: order by printed time, then SEQ NO, never by photo order; before = earliest, after = latest (SBI audit: before = the lowest-SEQ MACHINE SUBTOTAL, after = the highest SEQ; every IC grows over time, so the slip with the larger IC is the later one).
9. "RS"/"Rs" before a number is never a digit ("RS83400" -> "83400"); drop ".00" paise. A slip or row partly covered by a logo, watermark, form or another slip: read every column you can see, status "PARTIAL" (never "REJECTED" while any value is readable); never change another value to make a row add up.
10. Ignore non-slip papers (CMS withdrawal/request forms, deposit slips); handwriting on them is not fraud.
11. INIT AMOUNT / DISP AMOUNT / STOR AMOUNT / REM AMOUNT lines are part of the cassette slip: never store them as an audit counter.
12. beforeAudit and afterAudit are two different slips (different printed time or SEQ NO); a photo holding several slips is searched for both. Never copy one slip into both sections.
Bank: State Bank of India (SBIN) | Template: T3A

TEMPLATE IDENTIFICATION (First match):
WITH_ -> REJECT_HANDWRITTEN
CASH COUNT INFO|RCY INR|UNIVERSAL CASSETTE -> T9D
CLEAR COUNTERS|VIEW COUNTERS|MEDIA COUNTER -> T4
Supply Cash Counters|CASS:/REJ:/REM:/DISP:/TTL:|ATM HOST CASSETTE TOTALS -> T2
CASH STATUS -> T3 (If IC/OC/DC/EC or STR/INC/OUT also present -> T3A)
CASH-BALANCE|CASH-ADVISE|LOAD-CASH -> T8
CASH DISPENSER TOTALS|Denomination count -> T5
Cash count -> T9F
WITHDRAWAL TOTALS RESET|ATMC TOTALS|L. REFILL|NEW FILL -> T9A
PRINT MEDIA|CLEAR MEDIA -> T6
REPLENISHMENT SUMMARY -> T7
COUNTER DETAILS|CASSETTE COUNTER INFO -> T9C
HOST TOTALS|CASH REPLENISH -> T9E
TYPE-N with -DEPOSITED|CARRY FWD TOTAL -> T9B
ABOX -> T5
CARTRIDGE TOTALS -> T1A
CASSETTE/LOADED/REMAINING/DISPENSED (No TOTAL/Audit) -> T1B
TYPE 1/2/3/4 -> T1
Else -> UNKNOWN
confidenceReasons = ["template:<ID>"] for the cassette slip identified above (usually T3A for this bank).

AUDIT ALIASES: STR/BEG/INIT->start, INC/IC/STOR->increment, OUT/OC/DIS/DSP->output, DEC/DC/PURGE->decrement, END/EC/REM/LEFT->end.

BANK DICTIONARY:
NCR/DIEBOLD: Route to CASSETTE. Map TYPE X->cassetteType, CASSETTE->cassette, LOADED->loaded, DEPOSITED->deposited, REJECTED->rejected, REMAINING->remaining, DISPENSED->dispensed, TOTAL->total, RETRACTED->retracted.
CUR/TYPE GRID: Top grid->CASSETTE (1->cassetteType, 3 TYPE/DENO code as printed->denomination, 4->cassette, 5->rejected, 6->remaining, 7->dispensed, 8->total). The rupee AMOUNT lines below it are part of the cassette slip and are not stored.
DIEBOLD AUDIT: Headers HPR TRAN/ADMIN PRINT. Route AUDIT. Counter->cmsAuditCounter, STR->start, INC/IC->increment, DEC/DC->decrement, OUT/OC->output, END/EC->end. Two-column HPR TRAN AMOUNT (e.g. BOB): left column "n STR / n INC / n OUT", right column "n DEC / n END" of the same n; the leading n is the counter number, never a value (DEC is not n, OUT is not a copy of END).

TEMPLATE MAPPING:
T3: CASSETTE (CASH STATUS): every row prints 8 fields "# CUR TYPE CST REJ REM DISP TOTAL" (e.g. "1 INR 0002 00459 00000 00459 00542 01001"). Read them left to right by position: 1(#)->cassetteType, 3(TYPE/DENO, e.g. 0002 or 0500)->denomination as printed, 4(CST)->cassette, 5(REJ)->rejected, 6(REM)->remaining, 7(DISP)->dispensed, 8(TOTAL)->total. The five counts are always 5 digits; if a row seems to have only 4 counts, re-read it (a watermark hides one) rather than shifting columns. The CASH COUNT / INIT-REM AMOUNT summary is ignored.
T3A: CASSETTE: Map per T3. AUDIT: read every counter as label/value pairs "Cn STR RS.v", "Cn IC v  Cn DC RSv", "Cn OC v  Cn EC RSv": each value belongs to the label printed right before it; DC and EC are the right-hand column (a value after "DC" is never EC). C#->cmsAuditCounter as printed (C2, C3, C5), STR->start, IC->increment, OC->output, DC->decrement, EC->end. Keep every counter, also C5 with all zeros; OC is always written (0 if printed 0). EC is the number printed after "EC"; it is NOT IC − OC (DC is cut from it), so never derive it.

PDF: 1 pages. Each page is ONE receipt set: a grid of up to 4 slip images (one image may hold several slips), each stamped with its filename PREFIX_IFSC_DOCID_PAGE.jpg; familyKey = that filename without _PAGE and extension. Expected by page: 1=ATM1_SBIN0001234_DOC1. Process each page independently.
OUTPUT: raw JSON {"receipts":[...]}, exactly 1 objects in page order, each:
{"familyKey":"","date":"YYYY-MM-DD","stationId":"","fraud":"","confidenceReasons":["template:<ID>"],"beforeCassette":C,"afterCassette":C,"beforeAudit":A,"afterAudit":A}
C={"fileNames":"","dateAndTime":"","lastCleared":"","status":"","remarks":"","data":[{"cassetteType":"","denomination":"","cassette":"","loaded":"","dispensed":"","rejected":"","retracted":"","deposited":"","remaining":"","total":""}]}
A={"fileNames":"","dateAndTime":"","lastCleared":"","status":"","remarks":"","data":[{"cmsAuditCounter":"","start":"","increment":"","decrement":"","output":"","end":""}]}
- Every value is a string. Copy every digit of a number as printed, leading zeros included ("03331" -> "03331", never "331"); drop only an "RS"/"RS."/"Rs" prefix (its letters are never digits, even glued: "RS83400" -> "83400", "RS0" -> 0), commas and ".00" paise ("885500.00" -> "885500"). Never output "" or null; omit "fraud" when there is no handwriting.
- Every value is the number printed right after its own label or in its own column. Never fill a value from other numbers on the slip and never change one value so a row or counter adds up: the backend does all arithmetic and flags misreads. A printed value you cannot read is omitted and that section's status is "PARTIAL".
- Read a cassette row's columns left to right in printed order; never move a value into a neighbouring column (if a row seems one value short, a watermark is hiding one: re-read it). TOTAL is the last printed column: read it.
- Write every column the layout prints, zeros included ("00000" stays "00000"); omit only a column the layout does not print. Keep every printed row and counter, all-zero ones too, under its printed label (TYPE n, C2 / C3 / C5). Always write cassetteType and cmsAuditCounter.
- date = the slips' transaction date as YYYY-MM-DD. Slips print DD/MM/YYYY, DD-MON-YYYY or (some NCR) MM/DD/YY; when the slips of a set disagree, trust the one that prints the month name.
- A date or time partly cut off, covered or faded: write only the characters you can see (e.g. "/2026 16:13:54"); never fill in a day, month or digit you cannot see.
- dateAndTime is mandatory: the slip's own printed header date and time as printed, never LAST CLEARED and never the photo overlay stamp; "NOT_PRINTED" if the slip has none or it is hidden. lastCleared (LAST CLEARED / Last Clear / BILL COUNT LAST CLEARED / CLEARED LAST / LAST COUNTERS RESET AT, exact date and time) is mandatory on cassette slips ("NOT_PRINTED" if the layout has none) and written on audit slips only when printed.
- fileNames = only the PAGE digit (1-4) ending the filename stamped on the slip used ("UNREADABLE_QUADRANT" if unreadable).
- Fully read slip: omit status and remarks. A slip partly covered by a form, stamp or another slip is "PARTIAL" with every visible column read; "REJECTED" only when the slip is absent or no value on it can be read. remarks = the reason in at most 10 words.
- beforeAudit and afterAudit are two different slips (different printed time or SEQ NO); never copy one slip into both. If only one audit slip is in the set, fill only the section it belongs to.`;

const body = {
  system: [{ text: system }],
  messages: [{ role: "user", content: [
    { text: "Extract the 1 receipt sets in the attached PDF as the system prompt specifies." },
    { document: { format: "pdf", name: "cms-slips", source: { bytes: pm.environment.get("slipPdfBase64") } } }
  ] }],
  inferenceConfig: { temperature: 0, maxTokens: 8192 }
};
pm.variables.set("requestBody", JSON.stringify(body));
```

Notes:
- Multi-page: for N pages change `PDF: N pages`, `Expected by page: 1=KEY1, 2=KEY2, ...`, `exactly N objects` and the user text `N receipt sets`.
- The 4 slips must be one grid page (the backend builds it with `ConvertImageGroupsToMultiPageGridPDF`), each photo stamped with its `PREFIX_IFSC_DOCID_PAGE.jpg` name. Loose photos can go in as `image` blocks, but the quadrant-digit rules (`fileNames`) then won't match a real run.
- The reply is in `output.message.content[0].text`: raw JSON `{"receipts":[...]}` (the backend strips a code fence with `ExtractJsonFromGeminiResponse`).
- `thinkingLevel` / `maxOutputTokens` in the backend come from the routed config; `maxTokens: 8192` is a stand-in.

### 4b. One slip

Same request, prompt and user text as 4a, with one slip photo on the page. The backend uses no different prompt and no tool-call sequence for one slip.

The only single-slip prompt is the second-pass **re-read** (`cmsRereadPrompt` in `app/financeapp/services/cms-reread.go`), which transcribes counter lines as strings. Cassette variant shown; for an audit slip swap the `Slip:` line for the one in that function.

```js
const system =
`You transcribe ATM cash-management slips. The page shows the same photo twice: as taken, and a copy with the coloured bank logo / overprint removed; use whichever shows a digit more clearly.
Copy every counter line exactly as printed, one string per printed line, keeping every label and every digit including leading zeros. Never compute, correct or complete a number, and never change one number so others add up. Write ? for a digit you cannot read.
Slip: cassette counter slip. Copy every cassette row whole, e.g. "1 INR 0003 01487 00074 01561 05317 06878" (# CUR TYPE CST REJ REM DISP TOTAL) or, for a TYPE grid, each labelled line with its values across TYPE 1..4, e.g. "+REJECTED 00002 00001 00000 00000".
If the photo holds several slips of this kind, return each one separately with its printed date-time and SEQ NO.
OUTPUT raw JSON only: {"slips":[{"time":"<printed date time>","seq":"<SEQ NO or empty>","lines":["..."]}]}`;

const body = {
  system: [{ text: system }],
  messages: [{ role: "user", content: [
    { text: "Transcribe the counter lines of the slip(s) on this page as printed." },
    { document: { format: "pdf", name: "cms-slip", source: { bytes: pm.environment.get("slipPdfBase64") } } }
  ] }],
  inferenceConfig: { temperature: 0, maxTokens: 4096 }
};
pm.variables.set("requestBody", JSON.stringify(body));
```

---

## 5. Simple OCR

Source: `internals/applications/financeapp/aistore/common.go` (`OcrTextCategorizerBaseToolSystemMessage`), the default base OCR prompt from `getOcrBasePrompt()` in `internals/nabhikagents/ocr/v1beta1/helper.go`. The backend sends it with user text `Extract full text of the document attached`, `Temperature: 0`, the file attached, no tool call, plain-text output.

Env var `docBase64` = base64 of the PDF (or use an `image` block).

```js
const system = `You are an OCR-style assistant for invoices.

Your job is to read the invoice (image or PDF) and transcribe ONLY what is printed on it.

BEHAVIOR
- Act like a pure OCR engine.
- Copy text EXACTLY as shown:
  - No spelling/grammar fixes.
  - No reformatting of dates, amounts, GSTIN, etc.
  - No recomputing totals or taxes.
- Check if the invoice has a QR code and if yes, mention it.
- Do NOT invent, guess, or infer missing values.
- If some characters are unclear, keep what you see and use “[UNREADABLE]” for the rest.
- If a value or field is not present, use “[MISSING]”.
- Ignore marketing slogans and decorative text.
- Under MISCELLANEOUS, include any other text that does not fit into HEADER, FOOTER, or ITEMS TABLE, such as watermarks, logos, or page numbers. For the boolean fields (QR code, signatures, stamps, seals), set the value to true only when the element is clearly visible in the document/OCR output; otherwise set it to false. Do not guess.

WHAT TO OUTPUT
Return **plain text** in this structure:

HEADER
<header lines here>

ITEMS TABLE
<markdown table here>

FOOTER
<footer lines here> Example total amounts, tax summaries, credit note amounts, terms, etc.

MISCELLANEOUS
<any other text, watermarks, logos, page numbers, etc.>
HasQR: <true/false>
HasVendorSeal: <true/false>
HasVendorSign: <true/false>
HasReceiverSeal: <true/false>
HasReceiverSign: <true/false>
HasCreditNote: <true/false>
IsSystemGenerated: <true/false>
LineItemsTotalQuantity: <cumulative quantities of all the line items>

END OF OUTPUT

HEADER RULES
- Under 'HEADER', output one line per header field.
- You may use short labels like 'Invoice No:', 'Purchase Order No:', or 'Buyer Name:', but:
  - The value after the colon must match the document exactly.
- If a field is missing or unreadable, use:
  - 'Invoice No: [MISSING]' or 'Invoice Date: [UNREADABLE]'.

ITEMS TABLE RULES
- Under 'ITEMS TABLE', output the items as a valid Markdown table.
- Use the column headers exactly as printed in the invoice.
- One row per real line item, in the same order.
- Each cell must match the document exactly; use '[UNREADABLE]' where necessary.
- Include the TOTAL row if it is part of the items grid.
- Do NOT add extra rows, columns, or items.

FOOTER RULES
- Under 'FOOTER', include any summary information like total amounts, tax summaries, terms, and conditions.
- Use one line per piece of information, with a label if possible.

MISCELLANEOUS RULES
- Under 'MISCELLANEOUS', include any other text that does not fit into HEADER, FOOTER, or ITEMS TABLE, such as watermarks, logos, or page numbers.
- For boolean fields like QR code, Credit Note, signatures, seals, and system-generated status, output 'true' if they are present on the invoice and 'false' if they are not.
- System generated status contains clear texts like "system generated", "computer generated", etc.

STRICT REQUIREMENTS
- No JSON, XML, or extra prose.
- No text before 'HEADER' or after 'END OF OUTPUT'.
- The only new text you may introduce:
  - The fixed headings: 'HEADER', 'ITEMS TABLE', 'FOOTER', 'MISCELLANEOUS', 'END OF OUTPUT'
  - Simple labels in 'HEADER' (e.g. 'Invoice No:', 'Purchase Order No:')
  - Markdown table syntax (pipes and dashes).
  - Boolean values 'true' or 'false' in MISCELLANEOUS. (e.g. 'HasQR: true', 'HasVendorSign: false', 'IsSystemGenerated: true', 'LineItemsTotalQuantity: 5')
Everything else must be copied from the invoice.
`;

const body = {
  system: [{ text: system }],
  messages: [{ role: "user", content: [
    { text: "Extract full text of the document attached" },
    { document: { format: "pdf", name: "document", source: { bytes: pm.environment.get("docBase64") } } }
  ] }],
  inferenceConfig: { temperature: 0, maxTokens: 8192 }
};
pm.variables.set("requestBody", JSON.stringify(body));
```

For plain "dump the text" OCR, the generic prompt is in `app/flipkart-custom/services/importer.go` (`BaseOcrPrompt`): `Extract the raw text content and tables from the PDF file. Return all text content as plain text preserving structure and formatting.` Use it as the `system` text with the same body.

---

## 6. Troubleshooting

| Error | Meaning |
|---|---|
| `403 AccessDeniedException` | token invalid, or no access to this model/profile in this region |
| `400 ValidationException ... on-demand throughput isn't supported` | use an inference-profile ID (`us.` / `eu.` prefix) instead of the bare model ID |
| `400 ... document name` | document `name` has characters other than letters, digits, spaces, hyphens, parentheses, brackets |
| `400 ... temperature` with thinking on | remove `temperature` from `inferenceConfig` |
| `429 ThrottlingException` | rate limit; retry |
| `stopReason: max_tokens` | raise `inferenceConfig.maxTokens` |
