---
layout: post
title: Scheduled Testing with Playwright
categories:
  - Testing
  - Playwright
  - CI/CD
  - GitLab
---
## Motivation

Recently there were a few important changes made to the project I spend most of my time on. The first was the departure of our dedicated tester, who was great at finding new issues and catching longstanding issues with performance and functionality. We have also reduced developer capacity now, which means fewer eyes on the pages we're serving and a bigger chance of things slipping through the cracks. We should have had end-to-end testing implemented earlier, but it is better late than never. We selected [Playwright](https://playwright.dev/) for this purpose. 

## Goals

- Configure Playwright for an existing project
- Create page navigation testing sample
- Create visual testing sample
- Schedule tests to run on a GitLab CI/CD pipeline

## Integrating Playwright

Installing Playwright in an existing project was straightforward with npm:

```
npm init playwright@latest
```

As outlined in the [documentation](https://playwright.dev/docs/intro), the CLI walks you through selecting JavaScript or TypeScript, the directory name for tests, and whether a GitHub Actions workflow is needed:

```
Getting started with writing end-to-end tests with Playwright:
Initializing project in '.'
✔ Where to put your end-to-end tests? · e2e
✔ Add a GitHub Actions workflow? (Y/n) · false
Installing Playwright Test (npm install --save-dev @playwright/test)…
```

With it installed, the sample tests could be run immediately:

```
npx playwright test
```

After the sample tests passed, running `npx playwright show-report` serves the test report in a browser window. Failing tests runs are served automatically and provide good detail for debugging what part of the tests might need adjustment. 

## Page Navigation Tests

I made a couple of small adjustments to the example tests to incorporate dismissing a cookies notice, clicking a button, then checking for the presence of a page title. The example tests were not difficult to expand on, and only a simple change was needed to ensure that they did apply to the site that needs to be tested: 

```
import { test, expect } from '@playwright/test';

test('has title', async ({ page }) => {
    await page.goto('https://www.page-for-testing.com/');

    // Expect a title "to contain" a substring.
    await expect(page).toHaveTitle(/The Title of This Page/);
});

test('test button', async ({ page }) => {
    await page.goto('https://www.page-for-testing.com/');

    // Dismiss cookies.
    await page.getByRole('button', { name: /decline all cookies/i }).click();

    // Click the button.
    await page.getByRole('button', { name: /button name/i }).click();

    // Expects page to have a specified heading, ex: 'Heading on This Page'
    await expect(page.getByRole('heading', { name: 'Heading on This Page' })).toBeVisible();
});
```

## Visual Tests

### Sample Test

One of the starting points which led to selecting Playwright was the need for simple implementation of visual testing, and creating a sample test case was extremely quick once the remainder was configured:

```
// visual-test-sample.spec.ts

import { test, expect } from '@playwright/test';  
  
test('sample test', async ({ page }) => {  
    await page.goto('https://www.website.com/subpage/subpage');  
    await page.getByRole('button', { name: /decline all cookies/i }).click();  
    await expect(page).toHaveScreenshot();  
})
```

The test is configured to navigate to the designated page and wait until it is fully loaded before dismissing the cookies notice and then taking a screenshot. 

### Creating Baseline Images

In order to run the test properly, it's necessary to have a set of baseline images ready for comparison. They can be created by running the following command:

```
npx playwright test --update-snapshots
```

Under the first conditions that I set for the test, running the tests failed unexpectedly. This was thanks to a small animation running on the page; because the animation was captured at different cycles each time Playwright took a screenshot, the results differed and the tests failed. Switching the test page to one without an animation in the first view resolved that issue and then the tests could pass. 

## Playwright on the Pipeline 

Playwright has GitHub Actions supported out of the box, with slightly sparser documentation about implementing it in GitLab: 

```
stages:
  - test

tests:
  stage: test
  image: mcr.microsoft.com/playwright:v1.56.0-noble
  script:
  ...
```

### Basic Configuration

There were a few options for when to perform these tests. Our pipeline does daily database dumps already during off-peak hours, so my aim was to perform tests during that same window of opportunity. Because these were to be scheduled rather than triggered within a release pipeline, the `.gitlab-ci.yml` syntax needed to be organized a little differently. After some trial and error, this was the result for the basic test setup:

```
Run Playwright Tests:  
  image: mcr.microsoft.com/playwright:v1.56.0-noble  
  stage: Scheduler  
  rules:  
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $RUN_PLAYWRIGHT_TESTS == "1"'  
      when: on_success  
    - when: never  
  cache:  
    key: "playwright-baseline-${CI_COMMIT_SHA}"  
    paths:  
      - website/e2e/test-results/  
  script:  
    - *git-credentials  
    - pushd website  
    - npm ci  
    - mkdir -p ./website/e2e/test-results  
    - npx playwright test  
    - popd  
  artifacts:  
    paths:  
      - website/e2e/test-results  
      - website/e2e/test-results/junit.xml  
    reports:  
      junit: website/e2e/test-results/junit.xml  
    expire_in: 7 days  
    when: always
```

Breaking it down section by section, this job is defined as using the public Docker image provided by Playwright and takes place in the Scheduler stage defined in the overall pipeline. 

The rules define that the job will only run if the pipeline is triggered by GitLab's pipeline scheduler, and if the variable `RUN_PLAYWRIGHT_TESTS` is set to `1`. If those conditions aren't met, then the job does not run on any other pipelines. A cache was also added to the job by providing a key and path.

The scripts the job is set to run start with providing the git credentials for access to a private repository where one of the npm dependencies is stored before changing into the directory where the tests are configured and then installing all of the dependencies. 

Next, a directory is created as the destination for the test report. With all of the setup complete, `npx playwright test` triggers the start of the test. Once complete, the script returns to the parent directory. 

Artifacts are stored in directory created in the earlier script definition. The report name and destination is also defined, as well as when the reports can be discarded from artifact storage. 

The successful test run of this job looks like this:

```
Running with gitlab-runner 18.4.0 (139a0ac0)
on xxxxxx-runner-xxxxxx xxxxxxx, system ID: xxxxxxxxxxxxx

Preparing the "docker" executor
Using Docker executor with image mcr.microsoft.com/playwright:v1.56.0-noble ...
Using effective pull policy of [always if-not-present] for container mcr.microsoft.com/playwright:v1.56.0-noble
Pulling docker image mcr.microsoft.com/playwright:v1.56.0-noble ...
Using docker image sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx for mcr.microsoft.com/playwright:v1.56.0-noble with digest mcr.microsoft.com/playwright@sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx ...

Preparing environment
Using effective pull policy of [always if-not-present] for container sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Running on runner-xxxxxxxxx-xxxxxxxxx-concurrent-0 via xxxxxxxxxxxx...

Getting source from Git repository
Gitaly correlation ID: xxxxxxxxxxxxxxxxxxxxxxxxxx
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in /builds/path/to/project/.git/
Created fresh repository.
Checking out xxxxxxxx as detached HEAD (ref is master)...
Skipping Git submodules setup

Executing "step_script" stage of the job script
Using effective pull policy of [always if-not-present] for container mcr.microsoft.com/playwright:v1.56.0-noble
Using docker image sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx for mcr.microsoft.com/playwright:v1.56.0-noble with digest mcr.microsoft.com/playwright@sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx ...
$ mkdir -p ~/.ssh # collapsed multi-line command
$ pushd website
/builds/path/to/project/website /builds/path/to/project
$ npm ci
added 563 packages, and audited 564 packages in 40s
148 packages are looking for funding
run `npm fund` for details
found 0 vulnerabilities
$ mkdir -p ./website/coverage/playwright
$ mkdir -p ./website/test-results/
$ npx playwright test
Running 9 tests using 1 worker
✓ 1 [chromium] › e2e/test-site.spec.ts:3:5 › has title (2.6s)
✓ 2 [chromium] › e2e/test-site.spec.ts:10:5 › test button (5.4s)
✓ 3 [chromium] › e2e/visual-test-sample.spec.ts:3:5 › sample test (2.7s)
✓ 4 [firefox] › e2e/test-site.spec.ts:3:5 › has title (3.5s)
✓ 5 [firefox] › e2e/test-site.spec.ts:10:5 › test button (6.0s)
✓ 6 [firefox] › e2e/visual-test-sample.spec.ts:3:5 › sample test (2.7s)
✓ 7 [webkit] › e2e/test-site.spec.ts:3:5 › has title (2.6s)
✓ 8 [webkit] › e2e/test-site.spec.ts:10:5 › test button (7.1s)
✓ 9 [webkit] › e2e/visual-test-sample.spec.ts:3:5 › sample test (4.5s)
9 passed (41.3s)
$ popd
/builds/path/to/project

Saving cache for successful job00:00
Creating cache playwright-baseline-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx...
website/e2e/test-results/: found 9 matching artifact files and directories
Uploading cache.zip to [https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx)
Created cache

Uploading artifacts for successful job
Uploading artifacts...
website/e2e/test-results: found 9 matching artifact files and directories
website/e2e/test-results/junit.xml: found 1 matching artifact files and directories
Uploading artifacts as "archive" to coordinator... 201 Created correlation_id=xxxxxxxxxxxxxxxxxxxxxxxxxx id=xxxxxx responseStatus=201 Created token=xxxxxxxxx
Uploading artifacts...
website/test-results/junit.xml: found 1 matching artifact files and directories
Uploading artifacts as "junit" to coordinator... 201 Created correlation_id=xxxxxxxxxxxxxxxxxxxxxxxxxx id=xxxxxx responseStatus=201 Created token=xxxxxxxxx
Cleaning up project directory and file based variables

Job succeeded
```


### Visual Testing on the Pipeline

Making the transition from successful local tests to successful visual tests on the pipeline was a little tricky to debug. It's important to note that the default values for `playwright.config.ts` were adequate until this point. To ensure that the snapshots taken for visual testing and for the baseline were accessible to the pipeline, I specified in `defineConfig` that they should be placed in a subdirectory of `e2e/` :

```
export default defineConfig({  
    testDir: './e2e',  
    snapshotDir: './e2e/test-results',
    ...
});
```

A recurring issue in configuring the visual tests was their failure to find baseline images for comparison. It wasn't immediately obvious what was causing the issue, as the pipeline was looking in the correct directory and the baseline images had successfully been added to version control. 

Here's a sample of the error output from an unsuccessful run:

```
  1) [chromium] › e2e/visual-test-sample.spec.ts:3:5 › sample test ──────────────
    Error: A snapshot doesn't exist at /builds/path/to/project/e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-chromium-linux.png, writing actual.
      4 |     await page.goto('https://www.page-for-visual-testing.com');
      5 |     await page.getByRole('button', { name: /decline all cookies/i }).click();
    > 6 |     await expect(page).toHaveScreenshot();
        |     ^
      7 | })
      8 |
        at /builds/path/to/project/e2e/visual-test-sample.spec.ts:6:5
    attachment #1: sample-test-1 (image/png) ───────────────────────────
    Expected: e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-chromium-linux.png
    Received: test-results/visual-test-sample-sample-test-chromium/sample-test-1-actual.png
    ────────────────────────────────────────────────────
    Error Context: test-results/visual-test-sample-sample-test-chromium/error-context.md
  2) [firefox] › e2e/visual-test-sample.spec.ts:3:5 › sample test ──────────────
    Error: A snapshot doesn't exist at /builds/path/to/project/e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-firefox-linux.png, writing actual.
      4 |     await page.goto('https://www.page-for-visual-testing.com');
      5 |     await page.getByRole('button', { name: /decline all cookies/i }).click();
    > 6 |     await expect(page).toHaveScreenshot();
        |     ^
      7 | })
      8 |
        at /builds/path/to/project/e2e/visual-test-sample.spec.ts:6:5
    attachment #1: sample-test-1 (image/png) ───────────────────────────
    Expected: e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-firefox-linux.png
    Received: test-results/visual-test-sample-sample-test-firefox/sample-test-1-actual.png
    ───────────────────────────────────────────────────
    Error Context: test-results/visual-test-sample-sample-test-firefox/error-context.md
  3) [webkit] › e2e/visual-test-sample.spec.ts:3:5 › sample test ───────────────
    Error: A snapshot doesn't exist at /builds/path/to/project/e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-webkit-linux.png, writing actual.
      4 |     await page.goto('https://www.page-for-visual-testing.com');
      5 |     await page.getByRole('button', { name: /decline all cookies/i }).click();
    > 6 |     await expect(page).toHaveScreenshot();
        |     ^
      7 | })
      8 |
        at /builds/path/to/project/e2e/visual-test-sample.spec.ts:6:5
    attachment #1: sample-test-1 (image/png) ──────────────────────────
    Expected: e2e/visual-test-sample.spec.ts-snapshots/sample-test-1-webkit-linux.png
    Received: test-results/visual-test-sample-sample-test-webkit/sample-test-1-actual.png
    ───────────────────────────────────────────────────
    Error Context: test-results/visual-test-sample-sample-test-webkit/error-context.md
  3 failed
    [chromium] › e2e/visual-test-sample.spec.ts:3:5 › sample test ──────────────
    [firefox] › e2e/visual-test-sample.spec.ts:3:5 › sample test ───────────────
    [webkit] › e2e/visual-test-sample.spec.ts:3:5 › sample test ───────────────
  6 passed (38.6s)
```

It turned out to relate to the OS difference between local and pipeline builds. My local environment is MacOS so all the baseline images from local testing (which had been added to version control for comparison) included Darwin in their names: `sample-test-1-webkit-darwin.png`. The pipeline images generated were looking for images with the same OS as the pipeline uses: Linux. 

There are differences in visual rendering between operating systems, so forcing local tests to just name their `-darwin` images `-linux` wasn't an acceptable workaround. I instead downloaded the artifacts from a previous run and renamed the images to adhere to the expected convention: `sample-test-1-webkit-linux.png`. That correction meant that the pipeline could find the baseline images it expected, and the tests passed. 

## Next Steps

Follow up steps to the configuration described above are simple - build up a set of page navigation and visual tests which can verify a consistent visitor experience and identify page-breaking bugs despite reduced scrutiny by developers and site testers. Beyond this, alerting to failed test runs is visible when manually checking GitLab, and as the owner of this pipeline schedule I also get emails when the pipeline fails. It's not ideal to have to manually check in, and alerting shouldn't go to a single person but instead to a channel that can be joined. Making the tests results more visible is another priority for this reason. Lastly, the need to manually upload baseline images when a new visual test is defined is not ideal behavior. Finding a solution which involves images from the first run of a new job which implements `npx playwright test --update-snapshots` and then adds the results to version control could be a more predictable experience. In summary:

- Write page navigation tests
- Write visual tests
- Implement channel-based alerting on failed runs (Slack? Email list?)
- Automate adding the pipeline's baseline images to version control for subsequent runs