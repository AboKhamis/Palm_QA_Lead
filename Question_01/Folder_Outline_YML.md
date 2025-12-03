
# Folder/Architecture Outline 

```
proxy-api-keys-automation/
├── app_helpers/                     # Business logic helpers
│   ├── api_key_helper.py            # API key CRUD workflows
│   ├── ui_helper.py                 # Dashboard helper (UI)
│
├── base/                         # Base classes for common functionality
│   ├── base_api.py               # Base API client with auth
│   ├── base_page.py              # Base page object
│
├── configs/                       # Configuration files
│   ├── settings.py                # API endpoints, test data
│   ├── paths.py                   # File paths for reports, logs
│   └── test_data/                 # Static test data
│       ├── api_keys.json          # Sample key configurations
│       ├── proxy_pools.json       # Pool configurations
│       └── rate_limits.json       # Rate limit test scenarios
│
├── pom/                            # Page Object Model
│   └── web/
│       └── page_objects/
│           ├── login_page.py       # Login actions
│           ├── dashboard_page.py   # Dashboard navigation
│           ├── api_keys_page.py    # Create/view/revoke keys
│           ├── usage_log_page.py   # View usage metrics
│           └── settings_page.py    # Configure rate limits, IP whitelist
│
├── tests/                          # Test cases
│   ├── api/                        # API tests (using requests)
│   │   ├── test_api_keys_crud.py         # Create, read, update, delete
│   │   ├── test_rate_limiting.py         # Rate limit enforcement
│   │   ├── test_proxy_routing.py         # Pool routing logic
│   │   ├── test_ip_whitelisting.py       # IP whitelist validation
│   │   ├── test_environment_isolation.py # Dev/stage/prod separation
│   │   ├── test_revocation.py            # Key revocation propagation
│   │   └── test_high_scale.py            # 100k+ keys performance
│   │
│   ├── web/                        # E2E tests (using Selenium)
│       ├── test_dashboard_e2e.py         # Full user journey
│       ├── test_key_creation_ui.py       # UI key creation flow
│       ├── test_key_revocation_ui.py     # UI revocation flow
│       ├── test_usage_logs_ui.py         # Usage dashboard
│       └── test_settings_ui.py           # Settings configuration
│
├── utils/                          # Utility modules
│   ├── backend_helper.py           # Direct DB access for test data setup
│   ├── custom_logger.py            # Logging utilities
│   ├── driver_registry.py          # Selenium WebDriver management
│   ├── environment_manager.py      # Manage dev/stage/prod configs
│   
│
├── reports/                        # Test execution reports
│   ├── html/                      # HTML reports
│   ├── xml/                       # JUnit XML for GitLab
│   └── logs/                      # Detailed logs
│
├── conftest.py                     # pytest fixtures and hooks
├── pytest.ini                      # pytest configuration
├── requirements.txt                # Python dependencies
├── Pipfile                        # pipenv dependencies (if using pipenv)
└── .gitlab-ci.yml                 # GitLab CI pipeline
```



# GitLab CI/CD Pipeline - Proxy API Keys Test Automation

```yaml
# GitLab CI/CD Pipeline - Proxy API Keys Test Automation
# Framework: Selenium + pytest + requests
# Focus: UI/API automated tests (no performance tests)

image: python:3.13.5-slim

workflow:
  rules:
    - if: $CI_MERGE_REQUEST_IID
    - if: $CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH == "develop"

variables:
  # Test execution paths
  API_TESTS_PATH: "tests/api"
  WEB_TESTS_PATH: "tests/web"

  # Report paths (matching your folder structure)
  REPORTS_HTML_PATH: "reports/html"
  REPORTS_XML_PATH: "reports/xml"
  REPORTS_LOGS_PATH: "reports/logs"

  # Environment configs
  TEST_ENV: "dev"  # Override per job: dev, stage, prod

  # Browser config for Selenium
  BROWSER: "chrome"
  HEADLESS: "true"

cache:
  key: ${CI_PROJECT_ID}
  paths:
    - .venv/
    - .pytest_cache/

before_script:
  - apt-get update -qy
  - apt-get install -y curl jq git chromium chromium-driver
  - pip install --upgrade pip
  - pip install pipenv
  - pipenv install --dev  # Install from Pipfile
  - mkdir -p $REPORTS_HTML_PATH $REPORTS_XML_PATH $REPORTS_LOGS_PATH

stages:
  - lint
  - test
  - report

# ========================================
# STAGE: LINT
# ========================================

Lint Python Code:
# This is will be dedicated for lint stage to analyze the code statically using for example ((flake8, pylint))

# ========================================
# STAGE: TEST - API TESTS
# ========================================

Test API - Key Lifecycle (CRUD):
  stage: test
  interruptible: true
  allow_failure: false  # Critical path
  script:
    - pipenv run pytest $API_TESTS_PATH/test_api_keys_crud.py 
        --junitxml=$REPORTS_XML_PATH/api_crud.xml 
        --html=$REPORTS_HTML_PATH/api_crud.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_crud.html
      - $REPORTS_XML_PATH/api_crud.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_crud.xml
    expire_in: 1 week

Test API - Revocation Propagation:
  stage: test
  interruptible: true
  allow_failure: false  # Priority 1 - Critical
  script:
    - pipenv run pytest $API_TESTS_PATH/test_revocation.py 
        --junitxml=$REPORTS_XML_PATH/api_revocation.xml 
        --html=$REPORTS_HTML_PATH/api_revocation.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_revocation.html
      - $REPORTS_XML_PATH/api_revocation.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_revocation.xml
    expire_in: 1 week

Test API - Rate Limiting:
  stage: test
  interruptible: true
  allow_failure: false  # Priority 2 - Critical
  script:
    - pipenv run pytest $API_TESTS_PATH/test_rate_limiting.py 
        --junitxml=$REPORTS_XML_PATH/api_rate_limiting.xml 
        --html=$REPORTS_HTML_PATH/api_rate_limiting.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_rate_limiting.html
      - $REPORTS_XML_PATH/api_rate_limiting.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_rate_limiting.xml
    expire_in: 1 week

Test API - Environment Isolation:
  stage: test
  interruptible: true
  allow_failure: false  # Priority 3 - Critical
  parallel:
    matrix:
      - TEST_ENV: "dev"
      - TEST_ENV: "stage"
      - TEST_ENV: "prod"
  script:
    - pipenv run pytest $API_TESTS_PATH/test_environment_isolation.py 
        --junitxml=$REPORTS_XML_PATH/api_env_isolation_${TEST_ENV}.xml 
        --html=$REPORTS_HTML_PATH/api_env_isolation_${TEST_ENV}.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_env_isolation_*.html
      - $REPORTS_XML_PATH/api_env_isolation_*.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_env_isolation_*.xml
    expire_in: 1 week

Test API - IP Whitelisting:
  stage: test
  interruptible: true
  allow_failure: false  # Priority 4 - High
  script:
    - pipenv run pytest $API_TESTS_PATH/test_ip_whitelisting.py 
        --junitxml=$REPORTS_XML_PATH/api_ip_whitelist.xml 
        --html=$REPORTS_HTML_PATH/api_ip_whitelist.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_ip_whitelist.html
      - $REPORTS_XML_PATH/api_ip_whitelist.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_ip_whitelist.xml
    expire_in: 1 week

Test API - Proxy Routing:
  stage: test
  interruptible: true
  allow_failure: false  # Priority 5 - Medium
  script:
    - pipenv run pytest $API_TESTS_PATH/test_proxy_routing.py 
        --junitxml=$REPORTS_XML_PATH/api_proxy_routing.xml 
        --html=$REPORTS_HTML_PATH/api_proxy_routing.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/api_proxy_routing.html
      - $REPORTS_XML_PATH/api_proxy_routing.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/api_proxy_routing.xml
    expire_in: 1 week

# ========================================
# STAGE: TEST - WEB/UI TESTS
# ========================================

Test Web - Dashboard E2E:
  stage: test
  interruptible: true
  allow_failure: true  # UI tests can be flaky
  script:
    - pipenv run pytest $WEB_TESTS_PATH/test_dashboard_e2e.py 
        --junitxml=$REPORTS_XML_PATH/web_dashboard.xml 
        --html=$REPORTS_HTML_PATH/web_dashboard.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/web_dashboard.html
      - $REPORTS_XML_PATH/web_dashboard.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/web_dashboard.xml
    expire_in: 1 week

Test Web - Key Creation UI:
  stage: test
  interruptible: true
  allow_failure: true
  script:
    - pipenv run pytest $WEB_TESTS_PATH/test_key_creation_ui.py 
        --junitxml=$REPORTS_XML_PATH/web_key_creation.xml 
        --html=$REPORTS_HTML_PATH/web_key_creation.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/web_key_creation.html
      - $REPORTS_XML_PATH/web_key_creation.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/web_key_creation.xml
    expire_in: 1 week

Test Web - Key Revocation UI:
  stage: test
  interruptible: true
  allow_failure: true
  script:
    - pipenv run pytest $WEB_TESTS_PATH/test_key_revocation_ui.py 
        --junitxml=$REPORTS_XML_PATH/web_key_revocation.xml 
        --html=$REPORTS_HTML_PATH/web_key_revocation.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/web_key_revocation.html
      - $REPORTS_XML_PATH/web_key_revocation.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/web_key_revocation.xml
    expire_in: 1 week

Test Web - Settings UI:
  stage: test
  interruptible: true
  allow_failure: true
  script:
    - pipenv run pytest $WEB_TESTS_PATH/test_settings_ui.py 
        --junitxml=$REPORTS_XML_PATH/web_settings.xml 
        --html=$REPORTS_HTML_PATH/web_settings.html 
        --self-contained-html
        -v
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/web_settings.html
      - $REPORTS_XML_PATH/web_settings.xml
      - $REPORTS_LOGS_PATH/
    reports:
      junit: $REPORTS_XML_PATH/web_settings.xml
    expire_in: 1 week

# ========================================
# STAGE: REPORT - Aggregate Results
# ========================================

Generate Test Report:
  stage: report
  when: always
  script:
    - echo "All test results collected in artifacts"
    - ls -lah $REPORTS_HTML_PATH/
    - ls -lah $REPORTS_XML_PATH/
  artifacts:
    when: always
    paths:
      - $REPORTS_HTML_PATH/
      - $REPORTS_XML_PATH/
      - $REPORTS_LOGS_PATH/
    expire_in: 1 month
```