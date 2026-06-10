# codguardians: 🤖 AI Self-Healing Pipeline

Welcome to **codguardians**! This project implements a fully autonomous, language-agnostic AI agent powered by Anthropic's Claude. It monitors your CI/CD pipelines and production endpoints, automatically diagnosing failures, generating code fixes, and opening GitHub Pull Requests to resolve issues without human intervention.

### 🌟 Live Repository Examples
You can see this pipeline actively running in the following reference repositories:
* **Python (Pytest):** [codguardians-sample-project-python](https://github.com/arghya0696/codguardians-sample-project-python)
* **Java (Spring Boot / Maven):** [codguardians-sample-project-java](https://github.com/arghya0696/codguardians-sample-project-java)
* **Terraform (HCL):** [codeguardians-sample-tf-project](https://github.com/codguardians/codeguardians-tf-project)

---

## 🚀 Key Features

* **CI/CD Test Healing:** Automatically catches failing tests or compilation errors, extracts logs, and rewrites broken application logic.
* **SonarQube Healing:** Integrates with SonarCloud to automatically resolve code smells and security vulnerabilities.
* **100% Language Agnostic:** The execution scripts contain zero language-specific logic. You can switch from Java to Python to Terraform simply by swapping out the JSON configuration file.
* **Automated GitOps:** Automatically creates feature branches (e.g., `AI-fix-...`), stages files, commits changes as `github-actions[bot]`, and opens a detailed PR using the GitHub CLI.

---

## 📂 Architecture & File Structure

<img width="1472" height="1160" alt="image" src="https://github.com/user-attachments/assets/273a3bb5-dcd1-43cc-82cf-42d85c98346e" />


All automation scripts and configurations live inside the `.github/scripts/` directory.

| Component | Description |
| :--- | :--- |
| `self_healer.py` | The core, language-agnostic agent that intercepts CI/CD failures, reads test logs, parses Claude's XML tags (`<files>` and `<code>`), and applies fixes locally. |
| `sonar_healer.py` | Connects to the Sonar API to automatically resolve static analysis issues and security vulnerabilities. |
| `git_manager.py` | Git wrapper that handles dynamic branch creation, commits, and uses `gh pr create` to generate Pull Requests. |
| `ai-skills.json` | The "brain" configuration. Defines `test_command`, `test_reports_glob`, and language-specific rules (like Spring DI rules or Python exception handling). |
| `coding-standards.md` | Team standards strictly enforced by the AI, ensuring fixes use modern features and adhere to your architectural guidelines. |

---

## ⚙️ Configuration

### 1. The `ai-skills.json`
This file makes the pipeline language-agnostic. It controls how the AI reads reports and executes tests.
* **`model_version`**: The Anthropic model ID to use (e.g., `claude-3-5-sonnet-20241022`).
* **`test_command`**: The command the AI uses to validate its own fixes (e.g., `["mvn", "test"]` or `["pytest", "-v"]`).
* **`test_reports_glob`**: Where the AI should look for error logs (e.g., `target/surefire-reports/*.txt` or `reports/test-results.xml`).
* **Dynamic Rule Injection (Optional)**: You can add custom array blocks to teach the AI framework-specific debugging tricks (e.g., `"python_common_errors": [...]`, `"spring_di_rules": [...]`, or `"my_internal_framework_rules": [...]`). The Python script dynamically finds *any* list in the JSON and automatically injects it into Claude's context!
* **Prompt Overrides (Optional)**: The pipeline has highly optimized default prompts baked directly into the Python code. However, power users can completely override the AI's behavior by explicitly defining `file_extraction_prompt`, `fix_generation_system_prompt`, and `fix_generation_user_prompt`.

### 2. The `coding-standards.md`
Define your strict human code review rules here. The AI is instructed to never violate these. For example, you can enforce the use of specific libraries, mandate immutability, and strictly forbid renaming classes or modifying test assertions illegally.

---

## 🛠️ GitHub Actions Setup

To enable the Self-Healing pipeline in your repository, you need to configure your GitHub Actions YAML files and add the necessary secrets.

### Required Secrets
Go to **Settings > Secrets and variables > Actions** and add:
* `ANTHROPIC_API_KEY`: Your Claude API key.
* `SONAR_TOKEN`: (Optional) Your SonarCloud token if using the Sonar healer script.

### Permissions Requirement
Your GitHub Actions workflow file must explicitly grant write permissions to allow the bot to create branches and PRs:
```yaml
permissions:
  contents: write
  pull-requests: write

```
### 💻 CI/CD Workflow Examples

**For Python (`python-build.yml`):**
Executes `pytest` and triggers the AI script if the step fails.

```yaml
      - name: Stage 2 - Run Tests
        run: |
          mkdir -p reports
          pytest --junitxml=reports/test-results.xml -v

      - name: Stage 3 - AI Self-Healing Agent (Test Failures)
        if: failure()
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          echo "Tests failed. Invoking AI self-healing script..."
          python .github/scripts/self_healer.py

```
**For Java (`maven-build.yml`):**
Executes `mvn test` and triggers the AI script if the step fails.

```yaml
    - name: Stage 3 - Run Maven Test
      run: mvn test

    - name: Stage 5 - AI Self-Healing Agent (Test Failures)
      if: failure()
      env:
         ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      run: |
         echo "Tests failed. Invoking AI self-healing script..."
         python .github/scripts/self_healer.py
```
**For Terraform (`terraform-ci.yml`):**

```yaml
jobs:
terraform-plan:
name: "Stage 1 · Terraform Validate"
runs-on: ubuntu-latest
outputs:
plan_output: ${{ steps.capture.outputs.plan_output }}

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init (local backend, no AWS creds needed)
        run: terraform init -input=false -backend=false

      # Run validate but don't fail yet — capture output first
      - name: Terraform Validate
        id: validate
        run: |
          set +e
          terraform validate 2>&1 | tee /tmp/validate_output.txt
          echo "exitcode=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT

      # Always runs so the error text is available to the self-heal job
      - name: Capture error output
        id: capture
        if: always()
        run: |
          OUTPUT=$(cat /tmp/validate_output.txt | tail -80)
          echo "plan_output<<EOF" >> $GITHUB_OUTPUT
          echo "$OUTPUT"          >> $GITHUB_OUTPUT
          echo "EOF"              >> $GITHUB_OUTPUT

      - name: Fail if validate errored
        if: steps.validate.outputs.exitcode != '0'
        run: exit 1

self-heal:
name: "Stage 2 · AI Self-Heal"
runs-on: ubuntu-latest
needs: terraform-plan
if: failure()

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      # terraform binary is required for the healer's validate retry loop
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init (needed for validate retry)
        run: terraform init -input=false -backend=false

      - name: Install Python dependencies
        run: pip install anthropic requests

      # Bridge: write the captured error into the path self_healer.py already knows
      - name: Write Terraform error into surefire-reports path
        env:
          PLAN_OUTPUT: ${{ needs.terraform-plan.outputs.plan_output }}
        run: |
          mkdir -p target/surefire-reports
          echo "$PLAN_OUTPUT" > target/surefire-reports/terraform-validate.txt

      - name: Run AI Self-Healer
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN:      ${{ secrets.GITHUB_TOKEN }}
        run: python .github/scripts/self_healer.py
```
