# Demoblaze Performance Testing

A JMeter performance testing project for the [Demoblaze](https://www.demoblaze.com) e-commerce application. This project covers the full end-to-end user journey under five load scenarios, with automated CI/CD integration via GitHub Actions and Slack notifications.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Application Under Test](#application-under-test)
- [Performance Goals](#performance-goals)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Test Scenarios](#test-scenarios)
- [User Journey](#user-journey)
- [Running the Tests](#running-the-tests)
- [Generating the HTML Report](#generating-the-html-report)
- [CI/CD Integration](#cicd-integration)
- [Slack Notifications](#slack-notifications)
- [Test Results Summary](#test-results-summary)
- [Known Issues](#known-issues)

---

## Project Overview

This project uses Apache JMeter 5.6.3 to assess the performance, scalability, and reliability of the Demoblaze e-commerce platform under varying user loads. Five thread groups are defined — LightLoad, MediumLoad, PeakLoad, Stress Test, and Endurance Test — each exercising the complete six-step user journey from homepage through to checkout.

The test plan is integrated into a GitHub Actions workflow that runs automatically on pushes to `main`, on pull requests, and on a nightly schedule. After each run, a detailed statistics report is posted to a configured Slack channel.

---

## Application Under Test

| Property | Value |
|---|---|
| Application | Demoblaze E-Commerce Store |
| Web URL | https://www.demoblaze.com |
| API URL | https://api.demoblaze.com |
| Type | Single-Page Application (JavaScript) |

---

## Performance Goals

| Metric | Target |
|---|---|
| Average Response Time | ≤ 2,000 ms for all critical transactions |
| Throughput | ≥ 500 requests per second |
| Error Rate | ≤ 1% under maximum load |
| Scalability | Stable behaviour under increasing concurrent load |

---

## Prerequisites

Before running the tests, ensure the following are installed on your machine.

- **Java 17 or higher** — JMeter requires Java to run. Verify with `java -version`.
- **Apache JMeter 5.6.3** — Download from the [official Apache JMeter site](https://jmeter.apache.org/download_jmeter.cgi) and extract to a directory of your choice.
- **JMeter on PATH** — Add the JMeter `bin` directory to your system PATH so the `jmeter` command is available globally.

To verify JMeter is ready:

```bash
jmeter --version
```

---

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── jmeter-performance-test.yml   # GitHub Actions workflow
├── results/                              # Created at runtime — not committed
│   ├── results.jtl                       # Raw test results log
│   ├── jmeter.log                        # JMeter execution log
│   └── html_report/                      # Generated HTML dashboard
│           └── index.html
├── Test_Plan_Updated.jmx                 # JMeter test plan
└── README.md
```

> The `results/` directory is generated at runtime and should be added to `.gitignore`.

---

## Test Scenarios

Five thread groups are defined in the test plan. Only one should be enabled at a time before running.

| Thread Group | Users | Ramp-Up | Loops | Total Requests | Purpose |
|---|---|---|---|---|---|
| LightLoad | 50 | 50s | 1 | 300 | Baseline test |
| MediumLoad | 150 | 150s | 2 | 1,800 | Load test |
| PeakLoad | 300 | 300s | 3 | 5,400 | Load test |
| Stress Test | 500 | 500s | 2 | 6,000 | Break-point test |
| Endurance Test | 150 | 150s | 12 | 10,800 | Soak test |

All thread groups except LightLoad are disabled by default in the JMX file. Enable the desired group in JMeter's GUI before running, or see the [Running the Tests](#running-the-tests) section for CLI instructions.

---

## User Journey

Each thread group exercises the same six-step flow in sequence.

1. **GET Homepage** — Loads `https://www.demoblaze.com/index.html`
2. **POST Login** — Authenticates against `https://api.demoblaze.com/login` and extracts the session token via Regex Extractor
3. **POST Product Search** — Fetches the phones category via `https://api.demoblaze.com/bycat`
4. **GET Product Details** — Loads the product page at `https://www.demoblaze.com/prod.html?idp_=1`
5. **POST Add to Cart** — Adds the product to cart via `https://api.demoblaze.com/addtocart` using the extracted session token
6. **POST Checkout** — Submits the order via `https://api.demoblaze.com/send` using the extracted session token

Each step has the following assertions and timers applied:

- **Response Assertion** — checks HTTP status code is 200
- **Text Assertion** — checks the response body contains expected content
- **Duration Assertion** — fails if the response exceeds 2,000 ms
- **Gaussian Random Timer** — introduces a 1,000 ms base delay plus up to 3,000 ms random offset to simulate realistic user think time

---

## Running the Tests

### Step 1 — Enable the desired thread group

Open `Test_Plan_Updated.jmx` in JMeter's GUI. Enable only the thread group you want to run by right-clicking it and selecting Enable. Ensure all other thread groups are disabled. Save the file.

Run the thread groups in this recommended order:

```
LightLoad → MediumLoad → PeakLoad → Stress Test → Endurance Test
```

Always start with LightLoad to establish a baseline before progressing to heavier loads.

### Step 2 — Prepare the results directory

```bash
mkdir -p results && rm -rf results/html_report
```

### Step 3 — Run the test

```bash
jmeter -n -t Test_Plan_Updated.jmx -l results/results.jtl -e -o results/html_report
```

| Flag | Meaning |
|---|---|
| `-n` | Non-GUI (headless) mode |
| `-t` | Path to the test plan file |
| `-l` | Path to save the raw results log (.jtl) |
| `-e` | Generate HTML report after the test completes |
| `-o` | Output directory for the HTML report |

---

## Generating the HTML Report

### Option A — Run and generate in one command

Use this when running the test for the first time or after clearing old results.

```bash
mkdir -p results && rm -rf results/html_report && \
jmeter -n -t Test_Plan_Updated.jmx -l results/results.jtl -e -o results/html_report
```

### Option B — Generate from an existing .jtl file

Use this to regenerate the report from a previous test run without re-running the test.

```bash
rm -rf results/html_report && \
jmeter -g results/results.jtl -o results/html_report
```

Once generated, open `results/html_report/index.html` in a browser to view the full dashboard including APDEX scores, response time percentiles, throughput over time, and error breakdowns.

> **Note:** The `results/html_report` directory must not already exist when the report generator runs. JMeter will throw an error if it does. Always delete it first.

---

## CI/CD Integration

The project includes a GitHub Actions workflow at `.github/workflows/jmeter-performance-test.yml` that automates test execution.

### Triggers

The workflow runs automatically on:

- Every push to the `main` branch
- Every pull request targeting `main`
- A nightly schedule at 2:00 AM UTC
- Manual trigger from the GitHub Actions UI (workflow_dispatch)

### What the workflow does

1. Checks out the repository
2. Installs Java 17
3. Downloads and caches Apache JMeter 5.6.3
4. Prepares the results directory
5. Runs the JMeter test plan in non-GUI mode
6. Parses the `.jtl` results file and computes statistics
7. Sends a Slack notification with a full statistics table
8. Uploads the HTML report, JTL file, and JMeter log as downloadable artifacts
9. Fails the pipeline if the test run produced errors

### Setup

**1. Add the test plan to your repository**

Commit `Test_Plan_Updated.jmx` to the root of your repository.

**2. Add the workflow file**

Place `jmeter-performance-test.yml` at `.github/workflows/jmeter-performance-test.yml` in your repository.

**3. Add the Slack webhook secret**

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret** and add:

| Secret Name | Value |
|---|---|
| `SLACK_WEBHOOK_URL` | Your Slack Incoming Webhook URL |

To obtain a Slack Incoming Webhook URL, go to [api.slack.com/apps](https://api.slack.com/apps), create a new app, enable Incoming Webhooks, and add a webhook to your chosen workspace and channel.

**4. Push to main**

The workflow will trigger automatically on the next push to `main`. You can also trigger it manually from the **Actions** tab in your GitHub repository by selecting the workflow and clicking **Run workflow**.

### Artifacts

After each run, the following artifacts are available for download from the GitHub Actions UI for 30 days:

| Artifact | Contents |
|---|---|
| `jmeter-html-report-{run_number}` | Full HTML dashboard report |
| `jmeter-jtl-{run_number}` | Raw `.jtl` results file |
| `jmeter-log-{run_number}` | JMeter execution log (kept 14 days) |

---

## Slack Notifications

After every workflow run — whether it passes or fails — a notification is sent to the configured Slack channel. The message includes:

- Run status (passed or failed) with colour indicator
- Repository, branch, commit SHA, and trigger source
- Total requests, total errors, and error rate
- Average, minimum, and maximum response times
- Throughput in requests per second
- P90, P95, and P99 response time percentiles
- SLA compliance check for each of the three performance goals
- Per-request breakdown with individual counts, error rates, and response times
- A direct link to the GitHub Actions run for the full report

---

## Test Results Summary

The following results were recorded during the initial test run on 19 March 2026. Note that the 50% error rate across all runs is attributable to test-script defects, not application failures. See the [Known Issues](#known-issues) section for details.

| Thread Group | Users | Requests | Error Rate | Avg Response | Throughput | APDEX |
|---|---|---|---|---|---|---|
| LightLoad | 50 | 300 | 50.00% | 358 ms | 4.71 req/s | 0.446 |
| MediumLoad | 150 | 1,800 | 50.00% | 309 ms | 9.57 req/s | 0.474 |
| PeakLoad | 300 | 5,400 | 50.02% | 310 ms | 15.36 req/s | 0.473 |
| Stress Test | 500 | 6,000 | 50.02% | 310 ms | 11.31 req/s | 0.471 |
| Endurance | 150 | 10,800 | 50.03% | 280 ms | 30.62 req/s | 0.480 |

---

## Known Issues

Two test-script defects were identified during the initial run. These affect Login, Product Details, and Checkout across all thread groups and account for the entire 50% error rate. The application itself is not at fault.

**Issue 1 — Login text assertion mismatch**

The Login text assertion checks for the string `Auth_token` but the Demoblaze API returns the session token under the JSON key `token`. The assertion fails on every request. The Regex Extractor is also configured with the pattern `Auth_token: (.+?)"` which does not match the actual response, causing it to fall back to `NOT_FOUND`. This fallback value is then passed as the session cookie in the Checkout request, causing all Checkout requests to return HTTP 404.

*Fix:* Update the text assertion to check for `token`. Update the Regex Extractor pattern to `"token":"(.+?)"`.

**Issue 2 — Product Details text assertion targets JavaScript-rendered content**

The Product Details text assertion checks for the string `Add to cart`. Demoblaze is a Single-Page Application — JMeter receives a static HTML skeleton from the server and the Add to cart button is only rendered after JavaScript executes in a browser. The string is never present in the raw HTTP response, so the assertion fails on every request despite the server returning HTTP 200.

*Fix:* Replace the assertion string with one present in the static HTML (such as the page title), or switch the request to the REST API endpoint `GET /prod?id=1` and assert on a JSON field.

---

> For a full analysis of test results, root cause breakdowns, and recommendations, refer to `Performance_Test_Executive_Summary.docx`.