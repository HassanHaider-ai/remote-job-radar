# Job Radar: Remote Job Screening Pipeline

**Built with Make.com, Google Gemini, and the Arbeitnow API**

Job Radar is an automation project that collects job postings from the free Arbeitnow API, filters remote jobs related to AI and automation, scores new jobs with Google Gemini, saves the results in a Make Data Store, and sends an email with the best matches.

I built this project after completing the Make Academy Advanced course. My goal was to apply pagination, Data Stores, regex, AI integration, and error handling in one practical workflow.

**Project status:** Built and tested through multiple test cases. The Make scenario is kept inactive after testing to save credits.

**Privacy:** Screenshots use practice data and practice email accounts. No API keys or credentials should be included in this repository.

---

## 1. What It Does

1. Fetches job postings from the free Arbeitnow API using page-based pagination.
2. Checks the API response before processing the jobs.
3. Filters for remote jobs with AI and automation-related keywords.
4. Checks the Data Store to skip jobs that have already been processed.
5. Sends each new job to Google Gemini for a fit score from 0 to 10 and a short explanation.
6. Saves the scored job in the Data Store.
7. Collects jobs with a score of 5 or higher into one email.
8. Sends alerts when configured HTTP or Gemini operations fail.

## 2. Architecture

The main workflow is:

**Repeater → HTTP → Response Validation → Iterator → Remote and Keyword Filter → Data Store Existence Check → Sleep → Gemini Scoring → Save Job → Score Filter → Text Aggregator → Gmail Digest**

Separate error-handling routes are used for HTTP and Gemini failures. An additional fallback route is intended to handle unexpected API responses.

### Full Scenario

![Scenario overview](docs/screenshots/01-scenario-overview.png)

## 3. Version History

| Version | Main feature |
|---|---|
| V1 | Fetch jobs, filter remote jobs, and save them using the job slug as the key |
| V2 | Check whether a job already exists before processing it again |
| V3 | Add Gemini scoring with structured JSON output |
| V4 | Add a digest email, error alerts, Retry handling, and response validation |

---

## 4. Technical Decisions

### Page-Based Pagination

The Arbeitnow API uses page-based pagination. The workflow requests pages using `?page=i`.

The API response includes a `links.next` value, and the last page can return an empty `data` array. The workflow uses a limited number of pages to control execution time and credit usage.

### Filter Before Using AI

Remote and keyword filters run before the Gemini module. The Data Store existence check also comes before AI scoring.

This reduces unnecessary Gemini requests by excluding irrelevant jobs and jobs that have already been processed.

### Use the Job Slug as the Key

Each job has a slug that is used as its Data Store key.

The existence check uses this key to identify previously processed jobs. This helps prevent the same job from being scored repeatedly.

### Keyword Regex

The filter uses a case-insensitive regex to identify AI and automation-related job titles.

Word boundaries (`\b`) help reduce false matches for short keywords such as `AI` and `KI`. German terms such as `Automatisierung` can also be included because Arbeitnow has many German-language listings.

### Structured AI Output

Gemini receives the candidate profile, scoring rules, and job information.

The expected output contains two fields:

- `score`: a number from 0 to 10.
- `reason`: a short explanation of the score.

Using structured JSON makes it easier to map the results into separate Make fields.

### Clean and Short Job Descriptions

Job descriptions may contain HTML. The workflow removes HTML using `stripHTML` and keeps the first 2,000 characters.

This gives Gemini readable text while limiting the amount of information sent to the model.

### Save Jobs After Successful Scoring

A job is saved after Gemini successfully returns a score.

If Gemini fails, the job is not saved as a completed result. This allows the workflow to try scoring it again on a later run.

### One Digest Email

The Text Aggregator combines qualifying jobs into one digest rather than sending a separate email for every job.

The score filter is set to accept jobs scoring 5 or higher. During testing, the threshold was temporarily lowered to demonstrate the digest workflow.

### Rate Limiting

During testing, the Gemini free tier showed a limit of five requests per minute.

A 20-second Sleep module was added before the Gemini call to leave more time between requests. Provider limits can change, so this timing may need adjustment.

### API Response Validation

During testing, an incorrect API URL returned an HTML page with HTTP status 200. A successful status code alone was not enough to confirm that the response contained valid job data.

The response guard was designed to check the expected job array and the `Content-Type` header. Unexpected responses should go to a fallback route.

**Final verification note:** The router must be tested with both a valid JSON response and an unexpected HTML response before claiming that the final guard works correctly.

### Error Handling

HTTP and Gemini operations have separate error-handling routes.

The configured Retry settings are:

- HTTP: up to 3 retries, 15 minutes apart.
- Gemini: up to 2 retries, 15 minutes apart.

The error routes send alert emails. Repeated failures can generate multiple alerts because a separate single-alert-after-all-retries mechanism has not been added.

---

## 5. Testing and Results

The tests below cover different parts of the workflow. Their results should not be treated as if they all came from one execution.

### Test A: Initial Run

**Setup:** The Data Store was emptied, three pages were requested, and the score threshold was temporarily lowered to 0 to demonstrate the digest email.

The original test notes record the following:

- 7 jobs reached Gemini.
- 6 jobs were scored and saved.
- 1 Gemini request failed temporarily.
- The Text Aggregator combined the saved jobs into one bundle.
- A digest email was generated.

**Important:** The project notes contain a different count from another run: 8 jobs reached Gemini, 2 failed, and 6 were saved. Confirm the exact counts against the Make execution history before publishing them as final results.

**Evidence**

Canvas and operation counts:

![Run A canvas](docs/screenshots/02-run-a-canvas-counts.png)

Aggregator output:

![Aggregator output](docs/screenshots/05-aggregator-output.png)

Digest email:

![Digest email](docs/screenshots/06-digest-email.png)

Data Store records:

![Data Store records](docs/screenshots/04-data-store-records.png)

**Known issue:** The `reason` text appeared twice in some saved records because of a duplicate mapping in the Save Job module. This should be corrected before describing the saved records as clean.

### Test B: Duplicate Job Check

**Goal:** Previously saved jobs should not be sent to Gemini again.

The recorded test showed:

- 4 jobs passed the remote and keyword filter.
- All 4 were identified as already present in the Data Store.
- The Sleep, Gemini, and Save Job modules had 0 operations for those jobs.

This shows how the existence check can reduce repeated processing.

**Evidence**

Canvas and operation counts:

![Run B canvas](docs/screenshots/07-run-b-dedupe-canvas.png)

Data Store existence check:

![Existence check output](docs/screenshots/07a-check-existence-output.png)

### Test C: Unexpected API Response

**Setup:** The API path was intentionally changed to an incorrect path. The server returned an HTML page with HTTP status 200.

This test showed why checking the status code alone is not enough.

A response guard was designed to check the expected job array and JSON content type. During testing, a filter incorrectly compared the whole `Headers` collection instead of the individual `Content-Type` value. This caused valid responses to be sent to the fallback route.

The filter was identified as the source of the problem. The final configuration must be verified to ensure that valid JSON follows the normal route and unexpected HTML follows the fallback route.

**Evidence**

Router filter:

![Router filter rules](docs/screenshots/09-router-filter-rules.png)

Fallback alert:

![Fallback alert email](docs/screenshots/11-fallback-alert-email.png)

### Test D: Error Handling

| Test case | Recorded result |
|---|---|
| HTTP 500 | The error-handling route sent an alert and used the configured Retry mechanism. |
| Gemini 429 rate limit | Some requests failed before the Sleep module was added. |
| Gemini 503 temporary overload | A request failed with a temporary server-side error. The error was handled and an alert was sent. |

The Sleep module was added to reduce rate-limit failures. It cannot prevent every temporary server-side error, which is why error handling is still important.

**Evidence**

HTTP 500:

![HTTP 500 test](docs/screenshots/12-http-500-error.png)

Gemini rate-limit alert:

![Gemini rate-limit alert](docs/screenshots/06b-gemini-error-alert-email.png)

Gemini 503 error:

![Gemini 503 error](docs/screenshots/03b-gemini-503-handled-error.png)

### Score Filter and Gemini Prompt

The following screenshots show the scoring threshold and the instructions sent to Gemini.

| Score filter | Gemini prompt |
|---|---|
| ![Score filter](docs/screenshots/08-score-filter-final.png) | ![Gemini prompt](docs/screenshots/20-gemini-prompt.png) |

---

## 6. Evaluating the AI Scores

I checked Gemini's scores against job details instead of assuming every result was correct.

### Example: Fixing an Incorrect Score

A job titled **Test Automation Engineer** initially received a score of 7.

The job required more than five years of experience, was restricted to candidates within the EU, and focused on software QA testing rather than workflow automation.

Initially, Gemini received only the title, company, location, and tags. It did not have enough information to judge the match properly.

After the job description and stricter scoring rules were added, the same job received a score of 1.

| Version | Information sent to Gemini | Score |
|---|---|---:|
| Before | Title, company, location, and tags | 7 |
| After | The same information plus the job description and stricter rules | 1 |

This example shows that AI scores depend on the quality of the input data and the scoring instructions.

### Sample Scored Jobs

The following examples are from the recorded demo results.

| Job | Score | Short explanation |
|---|---:|---|
| GTM Manager - AI Startup | 1 | Sales-focused role with a specific language requirement |
| Customs AI Integration & Automation Engineer (France) | 3 | Relevant topic, but a location restriction was present |
| ServiceNow AI Architect | 3 | Relevant work, but a senior-level role |

This is a small sample, not a formal accuracy measurement. AI scores can still be wrong, so borderline jobs should be reviewed manually.

---

## 7. Cost and Limits

- Each module operation uses a Make credit. Filters and routers were not counted as module operations in the tests.
- The Gemini module uses credits when it processes a job, so filtering and deduplication before Gemini help control usage.
- The free-plan limits recorded during development were 1,000 credits per month, two active scenarios, and a five-minute maximum run time. These limits may change.
- A 20-second Sleep module limits how quickly new jobs can reach Gemini.
- Error alerts may be sent for each failed attempt or retry.
- Saved incomplete executions are not automatically retried while the scenario is inactive.

Credit usage depends on the number of pages, jobs, module operations, and retries. Actual usage should be checked in Make.

---

## 8. Known Limitations

1. **Job source:** Arbeitnow is German-focused, so relevant remote junior AI automation jobs may be limited.
2. **Candidate profile:** The candidate profile, keyword regex, and score threshold are configured inside the scenario.
3. **Limited job information:** Gemini receives metadata and only the first 2,000 characters of the cleaned description.
4. **Single source:** The workflow currently uses only one job board.
5. **Duplicate reason mapping:** Some saved records may contain duplicated reason text until the Save Job mapping is corrected.
6. **AI scoring:** Gemini can make mistakes. Scores should be treated as recommendations, not final decisions.
7. **Provider limits:** Gemini request limits and availability can change.

---

## 9. How to Adapt the Workflow

1. **Candidate profile:** Update Gemini's System Instructions with the target role, skills, location, and scoring rules.
2. **Keywords:** Change the regex in the filter after the Iterator.
3. **Minimum score:** Change the score filter before the Text Aggregator.
4. **Number of pages:** Adjust the Repeater count. More pages may find more jobs but will use more credits.
5. **Email settings:** Select your own Gmail connection and configure the recipient.
6. **Data Store:** Make sure the required fields and key are configured correctly before running the workflow.

### Setup

1. Create a Make Data Store named `DS_job_radar` with these fields:
   - `slug` (key)
   - `title`
   - `company_name`
   - `location`
   - `remote` (boolean)
   - `url`
   - `tags` (text)
   - `created_at` (number)
   - `score` (number)
   - `reason` (text)

2. Recreate the workflow in Make.com using the architecture and screenshots in this repository. **The Make.com blueprint is not included in this project.**

3. Create your own Gmail and Google Gemini connections and select them in the relevant modules. Never publish API keys, credentials, or private connection details.

4. Run the scenario manually and check the execution history and Data Store results before enabling any schedule.

---

## 10. Possible Improvements

- Add another job source and combine the results using a common field structure.
- Move the candidate profile, keywords, and score threshold into a configuration Data Store.
- Stop pagination when a page contains only jobs that have already been seen.
- Send one final alert after all retries fail instead of sending an alert for every attempt.
- Add a field to evaluate whether a job accepts applicants from the candidate's country.
- Fix the duplicate `reason` mapping in the Save Job module.
- Complete and verify the response guard tests for valid JSON and unexpected HTML.

---

## 11. What I Practiced

This project helped me apply:

- Page-based pagination with a Repeater.
- Data Stores and existence checks.
- Duplicate prevention using a unique key.
- Regex filters and word boundaries.
- Structured AI output with Google Gemini.
- Job-fit scoring and prompt improvement.
- Text aggregation and email digests.
- Rate limiting.
- Router fallback routes.
- Error handlers and Retry.
- Testing, debugging, and documenting real workflow issues.

---

## 12. Credits and Attribution

- Job data comes from the free [Arbeitnow API](https://www.arbeitnow.com/blog/job-board-api).
- The workflow was built using [Make.com](https://www.make.com) and Google Gemini.
- Job links point to their original Arbeitnow pages.

## Author

**Hassan Haider**

GitHub: [HassanHaider-ai](https://github.com/HassanHaider-ai/)
