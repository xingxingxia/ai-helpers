# Slack Plugin

A Claude Code plugin providing Slack integration and automation for OpenShift engineering workflows.

## Overview

This plugin provides commands to automate common Slack-related tasks for OpenShift engineers, including customer support, team communication, and workflow automation. The plugin helps reduce manual effort and improve response quality by leveraging AI to research, analyze, and draft responses.

## Commands

### `/slack:answer-question`

Research and draft answers to customer questions from Slack support channels.

**Purpose**: Automate the research process for answering technical questions in Slack channels like `#forum-ocp-apiserver`, `#forum-ocp-etcd`, `#forum-ocp-networking`, etc.

**Quick Start**:
```bash
# Basic usage
/slack:answer-question "How do I configure custom certificates for the API server?"

# With component specified
/slack:answer-question "What are the etcd performance tuning options?" etcd

# Cross-component question
/slack:answer-question "How does authentication work with the API server when using custom certificates?"
```

**What it does**:
1. Analyzes the customer question
2. Searches relevant OpenShift GitHub repositories for code, configs, and examples
3. Fetches relevant Red Hat documentation
4. Generates a comprehensive answer with references
5. Saves to `.work/slack/answer-{timestamp}.md`

**Key Features**:
- Automatic component detection
- Cross-component question support
- Jira issue integration
- Code and documentation references
- Version-aware responses

See the [answer-question command documentation](commands/answer-question.md) for full details.

## Prerequisites

### Required
- Claude Code CLI
- Internet access (for GitHub and Red Hat docs)

### Optional but Recommended
- **GitHub CLI (`gh`)**: For better repository searches
  - Install: https://cli.github.com/
  - Check: `which gh`

### Slack Authentication

For reading Slack thread content automatically:

#### Option 1: Manual Copy-Paste (Recommended for simplicity)
1. Open the Slack thread in your browser
2. Copy the customer's question text
3. Paste it as the command argument

```bash
/slack:answer-question "Copied question text here" --slack-url https://redhat.slack.com/archives/C123/p123456789
```

#### Option 2: Slack API Token (For automation)
1. Generate a Slack API token:
   - Go to https://api.slack.com/apps
   - Create a new app or use an existing one
   - Add OAuth scopes: `channels:history`, `channels:read`
   - Install the app to your workspace
   - Copy the Bot User OAuth Token (starts with `xoxb-`)

2. Set environment variable:
   ```bash
   export SLACK_BOT_TOKEN="xoxb-your-token-here"
   ```

3. Add to shell profile (to persist):
   ```bash
   echo 'export SLACK_BOT_TOKEN="xoxb-your-token-here"' >> ~/.bashrc
   ```

4. Use with just the URL:
   ```bash
   /slack:answer-question --slack-url https://redhat.slack.com/archives/C123/p123456789
   ```

### Jira Access (Optional)

If questions reference internal Jira issues:
- Set `JIRA_API_TOKEN` environment variable
- Verify access to the Jira instance
- Use `/jira:*` commands for deeper integration

## Component Support

The plugin understands OpenShift components and their repositories:

| Component | Slack Channels | Key Repositories |
|-----------|----------------|------------------|
| apiserver | #forum-ocp-apiserver | openshift/kubernetes, openshift/api |
| authentication | #forum-ocp-authentication | openshift/oauth-server, openshift/oauth-proxy |
| etcd | #forum-ocp-etcd | openshift/cluster-etcd-operator |
| networking | #forum-ocp-networking | openshift/ovn-kubernetes, openshift/sdn |
| storage | #forum-ocp-storage | openshift/local-storage-operator |
| installer | #forum-ocp-installer | openshift/installer |
| operators | Various | openshift/cluster-*-operator |

## Cross-Component Questions

The plugin automatically detects when questions span multiple components:

**Examples**:
- "How does authentication work with the API server?"
- "How does networking interact with storage for persistent volumes?"
- "What's the interaction between etcd and the cluster operators during upgrades?"

For cross-component questions, the plugin:
- Searches all relevant component repositories
- Identifies integration points and shared interfaces
- Explains how components work together
- Provides configuration examples for component interactions

## Answer Review Process

**⚠️ CRITICAL**: All generated answers include a prominent review warning.

**Before delivering answers to customers, you MUST**:
1. **Review** all technical details for accuracy
2. **Double-confirm** code references and documentation links are current
3. **Verify** version compatibility with the customer's environment
4. **Test** any provided commands or configurations if possible
5. **Tune** the language and examples to match the customer's context
6. **Remove** any sensitive or internal-only information

The generated answers are research drafts, not final customer responses.

## Output Location

Answers are saved to:
```
.work/slack/answer-{timestamp}.md
```

This allows you to:
- Review and edit before posting
- Keep a history of researched answers
- Share files with team members
- Track commonly asked questions

## Common Use Cases

### Customer Support Questions
```bash
# How-to questions
/slack:answer-question "How to enable audit logging for the Kubernetes API server?"

# Troubleshooting
/slack:answer-question "Why would authentication fail with 'certificate has expired' error?" authentication

# Configuration
/slack:answer-question "What are the supported values for the etcd quota-backend-bytes setting?" etcd

# Conceptual
/slack:answer-question "How does OpenShift handle DNS resolution for services?" networking
```

### Complex Scenarios
```bash
# Cross-component with Jira reference
/slack:answer-question "What's the fix for the authentication issue in OCPBUGS-12345?"

# Version-specific
/slack:answer-question "What changed in the networking stack between OCP 4.14 and 4.15?"

# Integration question
/slack:answer-question "How do I configure custom CA certificates for both the API server and OAuth server?"
```

## Future Commands

This plugin is designed to support additional Slack-related commands in the future, such as:
- Message formatting and templating
- Thread summarization
- Escalation workflows
- Metrics and analytics
- Team notifications

## Installation

### From Marketplace
```bash
# Add the ai-helpers marketplace
/plugin marketplace add openshift-eng/ai-helpers

# Install the slack plugin
/plugin install slack@ai-helpers

# Use commands
/slack:answer-question "Your question here"
```

### Manual Installation
```bash
# Clone the repository
git clone https://github.com/openshift-eng/ai-helpers.git ~/.claude-plugins/ai-helpers

# Restart Claude Code
# Commands will be available as /slack:*
```

## Tips for Best Results

1. **Be Specific**: Include context (versions, error messages, configs)
2. **Specify Components**: Helps narrow down search scope
3. **Use Quotes**: Wrap questions containing special characters
4. **Provide Context**: Include Slack URL or Jira references when relevant
5. **Review Thoroughly**: Always review and validate before customer delivery
6. **Test First**: Verify commands and configs in a test environment

## Contributing

To improve or extend this plugin:
1. Add new Slack-related commands under `commands/`
2. Enhance component-to-repository mappings
3. Improve documentation search strategies
4. Add support for new Slack workflows
5. Create skills for complex multi-step Slack tasks

See [CLAUDE.md](../../CLAUDE.md) for plugin development guidelines.

## Support

- **Issues**: https://github.com/openshift-eng/ai-helpers/issues
- **Repository**: https://github.com/openshift-eng/ai-helpers
- **Documentation**: See individual command documentation in `commands/`

## License

Same as the parent repository.
