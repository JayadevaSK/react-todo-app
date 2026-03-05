name: TestNeo CI

on:
  push:
    branches: [ main, staging, develop, master ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 90

    env:
      TESTNEO_MAX_WORKERS: "4"
      TESTNEO_BATCH_SIZE: "10"
      TESTNEO_TEST_TIMEOUT: "600"
      SKIP_RESOURCE_CHECK: "true"
      TESTNEO_AGENT_MODE: "true"
      AGENT_MODE: "true"

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.9"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install playwright psutil python-dotenv requests httpx \
          pydantic pydantic-settings sqlalchemy aiofiles openai \
          langchain langchain-community langgraph PyYAML structlog \
          numpy Pillow groq
          playwright install chromium

      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libnss3 libnspr4 libatk1.0-0 libatk-bridge2.0-0 \
          libcups2 libdrm2 libdbus-1-3 libxkbcommon0 libxcomposite1 libxdamage1 \
          libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libasound2 \
          libatspi2.0-0 libxshmfence1

      - name: Download TestNeo Agent
        env:
          TESTNEO_SERVER: ${{ secrets.TESTNEO_SERVER }}
          TESTNEO_API_KEY: ${{ secrets.TESTNEO_API_KEY }}
        run: |
          curl -f -H "X-API-Key: $TESTNEO_API_KEY" \
          "$TESTNEO_SERVER/api/web/v1/ci/agent-files/agent.py" -o agent.py

      - name: Trigger TestNeo CI Job
        id: trigger
        env:
          TESTNEO_SERVER: ${{ secrets.TESTNEO_SERVER }}
          TESTNEO_API_KEY: ${{ secrets.TESTNEO_API_KEY }}
          TESTNEO_PROJECT_ID: ${{ secrets.TESTNEO_PROJECT_ID }}
        run: |
          RESPONSE=$(curl -s -X POST "$TESTNEO_SERVER/api/web/v1/ci/trigger" \
            -H "X-API-Key: $TESTNEO_API_KEY" \
            -H "Content-Type: application/json" \
            -d "{\"project_id\": $TESTNEO_PROJECT_ID, \"branch\": \"$GITHUB_REF_NAME\"}")

          JOB_ID=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
          echo "JOB_ID=$JOB_ID" >> $GITHUB_OUTPUT

      - name: Run TestNeo Tests
        env:
          TESTNEO_SERVER: ${{ secrets.TESTNEO_SERVER }}
          TESTNEO_API_KEY: ${{ secrets.TESTNEO_API_KEY }}
          TESTNEO_JOB_ID: ${{ steps.trigger.outputs.JOB_ID }}
        run: python3 agent.py

      - name: Check Results
        if: always()
        env:
          TESTNEO_SERVER: ${{ secrets.TESTNEO_SERVER }}
          TESTNEO_API_KEY: ${{ secrets.TESTNEO_API_KEY }}
          JOB_ID: ${{ steps.trigger.outputs.JOB_ID }}
        run: |
          curl "$TESTNEO_SERVER/api/web/v1/ci/status/$JOB_ID" \
          -H "X-API-Key: $TESTNEO_API_KEY"