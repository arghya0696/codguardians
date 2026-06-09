# codguardians: 🤖 AI Self-Healing Pipeline

Welcome to **codguardians**! This project implements a fully autonomous, language-agnostic AI agent powered by Anthropic's Claude. It monitors your CI/CD pipelines and production endpoints, automatically diagnosing failures, generating code fixes, and opening GitHub Pull Requests to resolve issues without human intervention.

### 🌟 Live Repository Examples
You can see this pipeline actively running in the following reference repositories:
* **Python (Pytest):** [arghya0696/sample-project](https://github.com/arghya0696/sample-project)
* **Java (Spring Boot / Maven):** [arghya0696/codguardians-sample-project-java](https://github.com/arghya0696/codguardians-sample-project-java)

---

## 🚀 Key Features

* **CI/CD Test Healing:** Automatically catches failing tests or compilation errors, extracts logs, and rewrites broken application logic.
* **SonarQube Healing:** Integrates with SonarCloud to automatically resolve code smells and security vulnerabilities.
* **100% Language Agnostic:** The execution scripts contain zero language-specific logic. You can switch from Java to Python simply by swapping out the JSON configuration file.
* **Automated GitOps:** Automatically creates feature branches (e.g., `health-fix-...`), stages files, commits changes as `github-actions[bot]`, and opens a detailed PR using the GitHub CLI.

---

## 📂 Architecture & File Structure

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