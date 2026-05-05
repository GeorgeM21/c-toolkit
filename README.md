# GenAI-Dev-Toolkit

A curated collection of **custom AI agents**, **instructions**, and **prompts** to accelerate development with GitHub Copilot.

---

## 📂 Repository Structure

### **Agents/** - Custom AI Personas
Specialized AI agents for specific development tasks. Each agent has a unique role, expertise, and behavior.

**Why use them?** Agents provide context-aware assistance tailored to your task (code review, debugging, testing, etc.) and automatically apply relevant rules and constraints.

**Available agents:**
- `@bug-detective` - Find and fix bugs with systematic debugging
- `@code-documentation` - Generate comprehensive code documentation
- `@code-familiarization` - Understand unfamiliar codebases quickly
- `@code-generation` - Generate production-ready code following project patterns
- `@code-reviewer` - Perform thorough code reviews
- `@feature-scaffolder` - Build new features with best practices
- `@unit-testing-generator` - Create comprehensive unit tests

### **Instructions/** - Rules & Guidelines
Development guidelines and best practices that shape AI behavior.

- **`copilot-instructions.md`** - Core global instructions (merge with your `.github/copilot-instructions.md`)
- **`Tech/`** - Technology-specific instructions (e.g., `dotnet-copilot.instructions.md`)
- **`UseCases/`** - Agent-specific instructions that define rules and constraints for each custom agent

**How they work:** Instructions automatically apply to matching contexts via `appliesTo` metadata. You can also manually enforce them using `#file:instruction-name.md`.

### **Prompts/** - Ready-to-Use Templates
Pre-built prompts for common development scenarios. Many prompts auto-select a custom agent when available locally.

**Examples (from `Prompts/dotnet/`):** `add-unit-tests.prompt.md`, `create-readme.prompt.md`, `new-onion-api.prompt.md`

**How to use:** Reference with `#prompt-name` in Copilot Chat (e.g., `#add-unit-tests`)

### **Instructions/UseCases/** - Use-Case Instructions
Agent-specific instruction files that define rules and constraints for each workflow in this toolkit.

**Examples:** Feature Scaffolding, Unit Testing, Code Refactoring, Bug Fixing

---

## 🚀 Getting Started

### **Option 1: Manual Copy (Simple)**

Copy the files you need into your project's `.github/` folder:

```
your-project/
  .github/
    agents/               ← Copy agents here
      code-reviewer.agent.md
    instructions/         ← Copy instructions here
      dotnet-copilot.instructions.md
    prompts/              ← Copy prompts here
      add-unit-tests.prompt.md
    copilot-instructions.md  ← Merge main instructions here
```

**Then use them:**
- **Agents**: Select from Chat panel (e.g., `@code-reviewer`)
- **Instructions**: Auto-applied via `appliesTo` or enforce with `#file:instruction-name.md`
- **Prompts**: Reference with `#prompt-name` in Copilot Chat

---

### **Option 2: MCP Server (Automated - Recommended)**

Use our **Model Context Protocol (MCP) server** to search and fetch toolkit files directly from VS Code.

#### **Quick MCP Setup:**

**1. Clone the MCP Server:**
```powershell
git clone https://github.com/CognizantSoftVision/GenAI-Dev-MCP-Server.git
cd GenAI-Dev-MCP-Server
```

**2. Create GitHub Personal Access Token:**
- Go to [GitHub Settings → Tokens](https://github.com/settings/tokens)
- Generate new token with `repo` scope
- **Authorize for SSO** (cognizant organization)

**3. Set Environment Variable:**
```powershell
$env:GITHUB_PAT = "ghp_your_token_here"
```

**4. Build Docker Image:**
```powershell
docker build -t cognizant-mcp-server .
```

**5. Configure VS Code (`mcp.json`):**

Create `C:\Users\<YourUsername>\AppData\Roaming\Code\User\mcp.json`:

```json
{
  "mcpServers": {
    "cognizant": {
      "type": "stdio",
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "GitHub__PersonalAccessToken=${env:GITHUB_PAT}",
        "cognizant-mcp-server"
      ]
    }
  }
}
```

**6. Restart VS Code**

#### **MCP Usage Examples:**

**Search for agents:**
```
#get_cognizant_agents keyword="bug"
```

**Fetch agent to your workspace:**
```
#fetch_cognizant_agent name="code-reviewer"
```
→ Saves to `.github/agents/code-reviewer.agent.md`

**Fetch instruction:**
```
#fetch_cognizant_instruction name="dotnet"
```
→ Saves to `.github/instructions/dotnet-copilot.instructions.md`

**Fetch prompt:**
```
#fetch_cognizant_prompt name="add-unit-tests"
```
→ Saves to `.github/prompts/add-unit-tests.prompt.md`

**📖 Full MCP Documentation:** [GenAI-Dev-MCP-Server README](https://github.com/CognizantSoftVision/GenAI-Dev-MCP-Server)

---

## 💡 Usage

Once files are copied (manually or via MCP), use them in GitHub Copilot Chat:

### **Using Custom Agents:**
1. Open GitHub Copilot Chat panel
2. Select agent from dropdown (e.g., `@code-reviewer`)
3. Enter your prompt - the agent's behavior and rules apply automatically

### **Using Instructions:**
- **Automatic**: Instructions with `appliesTo` metadata apply contextually
- **Manual**: Enforce specific instruction with `#file:instruction-name.md`

### **Using Prompts:**
Type `#prompt-name` in Copilot Chat (e.g., `#add-unit-tests` or `#create-readme`)

---

## 🔗 Resources

- **MCP Server Repository**: [GenAI-Dev-MCP-Server](https://github.com/CognizantSoftVision/GenAI-Dev-MCP-Server)
- **Questions?** Check the MCP Server README for troubleshooting and advanced usage

---

**Internal Cognizant use only.**
