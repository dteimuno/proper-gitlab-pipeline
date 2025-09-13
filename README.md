# Your GitLab CI/CD file — what it does & how to tighten it

Below is a clear, job-by-job explanation of your `.gitlab-ci.yml`, then a short verdict with the **exact fixes** I recommend.

---

## Pipeline creation

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

* **Effect:** Only creates pipelines for **MRs** or for commits to the **default branch** (e.g., `main`).
* **Why it’s good:** Cuts noise from feature-branch pushes that aren’t in MRs.

---

## Global variables

```yaml
variables:
  NETLIFY_SITE_ID: '107c9c00-2367-4d9c-80bb-3c0cda594056'
  VITE_APP_VERSION: $CI_COMMIT_SHORT_SHA
```

* **NETLIFY\_SITE\_ID**: Targets the correct Netlify site.
* **VITE\_APP\_VERSION**: Embeds the short SHA in your app (great for traceability).

> Keep `NETLIFY_AUTH_TOKEN` as a masked CI/CD variable.

---

## Stages (order matters)

```yaml
stages:
  - build
  - deploy
  - test
```

* **Effect:** GitLab runs stages in this order → **build → deploy → test**.
* **Issue:** You generally want **tests before deploys**. (See “Quick fixes” below.)

---

## Pre-stage sanity check

```yaml
test_docker:
  stage: .pre
  image: docker
  script:
    - docker --version
    - docker version
```

* **What it does:** Verifies the Docker client is available.
* **OK as-is.** (You’re not building images here, so no `dind` needed.)

---

## Build the app

```yaml
build_website:
  image: node:22-alpine
  stage: build
  script:
    - node --version
    - npm --version
    - npm ci
    - npm run build
  artifacts:
    paths:
      - build/
```

* **What it does:** Installs deps and produces the production bundle in `build/`, then **shares it** with later jobs via artifacts.
* **Nice:** Uses a slim Node image and artifacts correctly.

---

## Artifact smoke test

```yaml
test_artifact: 
  stage: test
  script:
    - test -f build/index.html
```

* **What it does:** Ensures the bundle actually exists.
* **Note:** Because `test` currently runs **after** `deploy`, this check happens **too late**.

---

## Unit tests

```yaml
unit_tests:
  image: node:22-alpine
  stage: deploy
  script:
    - npm ci
    - npm test
  artifacts:
    when: always
    reports:
      junit: reports/junit.xml
```

* **What it does:** Runs your test suite and publishes a **JUnit report** to GitLab UI.
* **Issue:** It’s in the **deploy** stage, so tests run **after** build but **before** `test_artifact`, and **in the same stage** as deploy jobs. Move to a `test` stage.

---

## Review (preview) deploys for non-default branches

```yaml
netlify_review:
  image: node:22-alpine
  stage: .pre
  rules:
    - if: $CI_COMMIT_REF_NAME != $CI_DEFAULT_BRANCH
  environment:
    name: preview/$CI_COMMIT_REF_SLUG
    url: $REVIEW_URL
  before_script:
    - npm install -g netlify-cli
    - apk add curl jq
    - mkdir build
    - echo "Test" > build/index.html
  script:
    - netlify --version
    - netlify status
    - echo "Deploying to Site ID ${NETLIFY_SITE_ID}"
    - netlify deploy --dir=build --json --auth $NETLIFY_AUTH_TOKEN | tee deploy-result.json
    - REVIEW_URL=$(jq -r '.deploy_url' deploy-result.json)
    - echo $REVIEW_URL
    - curl $REVIEW_URL | grep 'GitLab'
    - echo "REVIEW_URL=$REVIEW_URL" > deploy.env
    - cat deploy.env
  artifacts:
    reports:
      dotenv: deploy.env
```

* **What it does:** For non-default branches, deploys a **preview** and writes the **preview URL** to a dotenv artifact (`REVIEW_URL`) for downstream jobs.
* **Nuance:** `environment:url` is evaluated **when the job starts**. Since `$REVIEW_URL` is only created **during** the job, this job’s environment card won’t get the URL. (Downstream jobs can use the value just fine.)
* **Also:** You’re deploying a **dummy** `build/` created with `echo`. If you want a real preview of your app, run this **after** `build_website` and use its artifact.

---

## Playwright end-to-end tests against the preview

```yaml
e2e:
  stage: deploy
  image: mcr.microsoft.com/playwright:v1.94.1-noble
  rules:
    - if: $CI_COMMIT_REF_NAME != $CI_DEFAULT_BRANCH
  variables:
    APP_BASE_URL: $REVIEW_URL
  script:
    - npm ci
    - ls -la
    - npm run e2e
    - ls -la
  artifacts:
    when: always
    paths:
      - reports/
    reports:
      junit: reports/playwright-junit.xml
```

* **What it does:** Runs browser tests against the **preview URL** (from the dotenv).
* **Robustness:** Add `needs: ["netlify_review"]` so the dotenv is guaranteed to be present and the job can start as soon as the preview is ready.
* **Stage:** Put it under `test` so tests block deploys.

---

## Staging deploy on default branch

```yaml
netlify_staging:
  image: node:22-alpine
  stage: deploy
  rules:
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'
  environment:
    name: staging/$CI_COMMIT_REF_SLUG
    url: 'https://staging--dteimuno-learn-gitlab.netlify.app/'
  before_script:
    - npm install -g netlify-cli
    - apk add curl
  script:
    - echo "Deploying to Site ID ${NETLIFY_SITE_ID}" 
    - netlify deploy --dir=build --alias=staging --site="$NETLIFY_SITE_ID" --auth="$NETLIFY_AUTH_TOKEN"
    - echo $CI_ENVIRONMENT_URL
    - curl $CI_ENVIRONMENT_URL | grep 'GitLab'
```

* **What it does:** On the default branch, deploys the artifact to the **staging alias**.
* **Tweak:** For a single, stable staging card in Environments, set `environment.name: staging` (not per-commit).
* **Reliability:** Consider grepping for a **known marker** (e.g., the `$VITE_APP_VERSION`) instead of `'GitLab'`.

---

## Manual production deploy

```yaml
netlify_prod:
  image: node:22-alpine
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'
  environment: 
    name: production
    url: 'https://dteimuno-learn-gitlab.netlify.app/'
  before_script:
    - npm install -g netlify-cli
    - apk add curl
  script:
    - echo "Deploying to Site ID ${NETLIFY_SITE_ID}" 
    - netlify deploy --dir=build --prod --site="$NETLIFY_SITE_ID" --auth="$NETLIFY_AUTH_TOKEN"
    - curl $CI_ENVIRONMENT_URL | grep 'GitLab'
```

* **What it does:** Manual “Play” gate to publish to **production**.
* **Good:** Keeps production safe behind explicit approval.

---

## Is your code good?

**Short answer:** Yes—solid foundation. A few adjustments will make it production-grade:

### Quick fixes (do these)

1. **Reorder stages** to run tests before deploys, and move `unit_tests` + `e2e` into the `test` stage:

   ```yaml
   stages: [build, test, deploy]
   ```

   ```yaml
   unit_tests:
     stage: test
   e2e:
     stage: test
     needs: ["netlify_review"]
   ```

2. **Make preview deploy use the real build** (optional but recommended):

   * Change `netlify_review.stage` to `test` (or `deploy` after tests), and add:

     ```yaml
     needs: ["build_website"]
     ```
   * Remove the `echo "Test"` placeholder and deploy the artifacted `build/`.

3. **Guarantee artifact availability** for any job that needs the bundle:

   ```yaml
   test_artifact:
     stage: test
     needs: ["build_website"]
   netlify_staging:
     stage: deploy
     needs: ["build_website"]
   netlify_prod:
     stage: deploy
     needs: ["build_website"]
   ```

4. **Pass `--site` in the review deploy** too (you already do in staging/prod):

   ```bash
   netlify deploy --dir=build --json --site="$NETLIFY_SITE_ID" --auth "$NETLIFY_AUTH_TOKEN"
   ```

### Nice-to-haves (speed & safety)

* **Cache Node deps** to speed `npm ci`:

  ```yaml
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths: [.npm/]
  ```

  and use `npm ci --cache .npm --prefer-offline`.

* **Stabilize environment cards**:

  * `netlify_staging.environment.name: staging` (single card), and add
    `resource_group: netlify-staging` to serialize staging deploys.

* **More reliable smoke checks**:

  * Grep for `$VITE_APP_VERSION` (render it somewhere in your app) instead of `'GitLab'`.

* **Interrupt old pipelines** to save minutes:

  ```yaml
  default:
    interruptible: true
  ```

If you want, I can hand you a compact, patched YAML with the above changes already applied.
