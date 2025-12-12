---
description: Answer customer questions from Slack support channels using OpenShift code and documentation
argument-hint: "<question> [component] [--slack-url URL]"
---

## Name
slack:answer-question

## Synopsis
```
/slack:answer-question <question> [component] [--slack-url URL]
```

## Description
The `slack:answer-question` command helps engineers answer customer questions from Slack support channels (e.g., #forum-ocp-apiserver) by analyzing the question, searching relevant OpenShift GitHub repositories and Red Hat documentation, and generating a comprehensive, well-researched answer with supporting references.

This command is designed to reduce the time engineers spend researching answers to customer questions by automating the process of:
- Searching OpenShift code repositories under https://github.com/openshift/
- Fetching relevant documentation from https://docs.redhat.com/en/documentation/openshift_container_platform/
- Synthesizing information into a clear, actionable answer
- Providing references for further reading

## Prerequisites

Before using this command, ensure you have the following:

1. **Slack Authentication** (if using `--slack-url` to read thread content):
   - The company Slack workspace requires authentication to read messages
   - **Option 1: Manual Copy-Paste** (Recommended for simplicity):
     - Open the Slack thread in your browser
     - Copy the question text manually
     - Pass it directly as the question argument
   - **Option 2: Slack API Token** (For automated reading):
     - Generate a Slack API token with `channels:history` and `channels:read` scopes
     - Set environment variable: `export SLACK_BOT_TOKEN="xoxb-your-token-here"`
     - The command will use this token to fetch thread content when `--slack-url` is provided
     - See: https://api.slack.com/authentication/token-types

2. **GitHub CLI** (Optional but recommended):
   - Install: `gh` CLI tool for better repository searching
   - Check if installed: `which gh`
   - Installation guide: https://cli.github.com/

3. **Internet Access**:
   - Required to access GitHub repositories
   - Required to fetch Red Hat documentation

4. **Jira Access** (if question includes internal Jira links):
   - If the question references Jira issues, especially those marked as internal/confidential
   - Ensure you have proper authentication to access Jira
   - Check environment variable `JIRA_API_TOKEN` is set if needed
   - Verify you can access the Jira instance before running the command
   - Use `/jira:solve` or other jira commands if deeper Jira integration is needed

**Note**: If you don't have Slack API access, you can still use this command effectively by copying the question text from Slack and pasting it as the argument. The `--slack-url` parameter is optional and only used for reference/documentation purposes.

## Implementation

### 0. Slack Thread Reading (if --slack-url provided)
- If `--slack-url` is provided and `SLACK_BOT_TOKEN` is set:
  - Parse the Slack URL to extract workspace, channel, and thread timestamp
  - Use Slack API to fetch thread messages: `https://slack.com/api/conversations.replies`
  - Extract the original question and any relevant context from the thread
  - Use this as the question text for analysis
- If `--slack-url` is provided but no token is available:
  - Display a message instructing the user to either:
    - Set `SLACK_BOT_TOKEN` environment variable, or
    - Copy the question text manually from Slack
  - Proceed with the question text from the argument

### 1. Question Analysis
- Parse the customer question to understand:
  - The technical problem or feature being asked about
  - The OpenShift component(s) involved (e.g., apiserver, networking, storage)
  - Any specific versions or configurations mentioned
  - Whether it's a "how-to", troubleshooting, or conceptual question
  - Whether the question spans multiple components (cross-component question)
- If component is not provided as an argument, infer it from the question
- Extract key technical terms and concepts
- **Cross-Component Detection**:
  - Identify if the question involves multiple components working together
  - Examples: networking + storage, authentication + apiserver, etcd + operators
  - If detected, search across all relevant component repositories
  - Note component interactions and integration points in the answer

### 2. Repository Search
- Identify relevant OpenShift repositories based on the component:
  - **apiserver/kube-apiserver**: openshift/kubernetes, openshift/api, openshift/apiserver-library-go
  - **authentication**: openshift/oauth-server, openshift/oauth-proxy
  - **networking/ovn**: openshift/ovn-kubernetes, openshift/sdn
  - **storage**: openshift/local-storage-operator, openshift/csi-*
  - **etcd**: openshift/cluster-etcd-operator
  - **installer**: openshift/installer
  - **operators**: openshift/cluster-*-operator
  - **general**: openshift/origin, openshift/openshift-docs
- **Cross-Component Repository Search**:
  - For questions spanning multiple components, search all relevant repositories
  - Look for integration points between components (e.g., how networking interacts with storage)
  - Check operator code that coordinates multiple components
  - Search for end-to-end tests that exercise component interactions
- Use GitHub code search or local cloning to search for:
  - Relevant code implementations
  - Configuration examples
  - API definitions
  - Test cases that demonstrate usage
  - Comments explaining behavior
- Search strategies:
  - Use Grep/Glob tools to search for keywords from the question
  - Look for relevant types, structs, and functions
  - Find configuration files (YAML, JSON)
  - Locate relevant test files
  - For cross-component questions, search for shared interfaces and integration code

### 3. Documentation Search
- Search Red Hat OpenShift documentation:
  - Base URL: https://docs.redhat.com/en/documentation/openshift_container_platform/
  - Target the latest stable version (4.17 or current)
  - Search for pages related to the component and question
- Use WebFetch to retrieve relevant documentation pages:
  - Configuration guides
  - API reference documentation
  - Troubleshooting guides
  - Architecture documentation
- Extract relevant sections that address the question

### 4. Answer Generation
Generate a comprehensive answer with the following structure:

**Answer Format:**
```markdown
## Summary
[One-paragraph direct answer to the question]

## Detailed Explanation
[2-3 paragraphs explaining how it works, referencing code and documentation]
[For cross-component questions: explain how components interact and integrate]

## Configuration/Implementation
[If applicable, provide example configuration or steps]
- Configuration examples from code
- YAML/JSON snippets
- Commands to run
- For cross-component: show how to configure component interactions

## References
- **Code**: [Links to relevant GitHub files with line numbers]
- **Documentation**: [Links to Red Hat docs]
- **Related Issues**: [If found, link to relevant GitHub issues/PRs]
- **Jira Issues**: [If referenced in the question and accessible]

## Additional Notes
[Any caveats, version-specific information, or edge cases]
[For cross-component: note dependencies and integration considerations]

---
**⚠️ IMPORTANT - REVIEW BEFORE DELIVERY**
This answer was generated automatically. Please:
1. **Review** all technical details for accuracy
2. **Double-confirm** code references and documentation links are current
3. **Verify** version compatibility with the customer's environment
4. **Test** any provided commands or configurations if possible
5. **Tune** the language and examples to match the customer's context
6. **Remove** any sensitive or internal-only information

Do not deliver this answer to customers without thorough review and validation.
---
```

### 5. Output Delivery
- Save the generated answer to `.work/slack/answer-{timestamp}.md`
- Display the answer to the user with the review reminder prominently shown
- **CRITICAL**: Always display the review warning at the end of the output
- Optionally, if `--slack-url` is provided, include a note about copying to Slack
- Suggest follow-up questions or related topics
- Remind the user to review, verify, and tune the answer before delivering to customers

## Process Flow

1. **Parse Arguments**:
   - Extract question text (required)
   - Extract component name (optional)
   - Extract Slack URL flag (optional)

2. **Analyze Question**:
   - Identify the component(s) if not provided
   - Detect if this is a cross-component question
   - Extract key technical terms
   - Determine question type (how-to, troubleshooting, concept)
   - Check for Jira references and verify accessibility

3. **Research Phase**:
   - **Repository Search**:
     - Clone or search relevant OpenShift repositories
     - For cross-component questions, search all related repositories
     - Use Grep to find relevant code sections
     - Identify configuration examples and integration points
     - Find test cases demonstrating usage
   - **Documentation Search**:
     - Use WebFetch to retrieve relevant Red Hat docs
     - For cross-component questions, search docs for integration guides
     - Extract sections addressing the question
     - Look for official examples and guidance
   - **Jira Search** (if applicable):
     - Fetch referenced Jira issues if accessible
     - Extract relevant technical details from Jira comments
     - Link to related bugs or features

4. **Synthesis Phase**:
   - Combine code insights with documentation
   - For cross-component questions, synthesize how components work together
   - Verify consistency between code and docs
   - Create clear examples
   - Organize information logically
   - Identify integration patterns and dependencies

5. **Answer Generation**:
   - Write summary with direct answer
   - Provide detailed explanation with references
   - Include code/configuration examples
   - List all references

6. **Save and Display**:
   - Save to `.work/slack/answer-{timestamp}.md`
   - Display formatted answer to user
   - **CRITICAL**: Display the review warning prominently
   - Provide Slack-friendly formatting tips
   - Remind user to review before delivery

## Return Value

- **File**: `.work/slack/answer-{timestamp}.md` containing the complete answer
- **Display**: Formatted answer with markdown displayed to the user, including the review warning
- **Summary**: Key points and references for quick review
- **Warning**: Prominent reminder to review before customer delivery

## Examples

1. **Basic usage with question only**:
   ```
   /slack:answer-question "How do I configure custom certificates for the API server?"
   ```

2. **With component specified**:
   ```
   /slack:answer-question "What are the etcd performance tuning options?" etcd
   ```

3. **With Slack URL for context**:
   ```
   /slack:answer-question "How does the CNI plugin handle pod network isolation?" networking --slack-url https://redhat.slack.com/archives/C123/p123456789
   ```

4. **Troubleshooting question**:
   ```
   /slack:answer-question "Why would authentication fail with 'certificate has expired' error?" authentication
   ```

5. **Configuration question**:
   ```
   /slack:answer-question "How to enable audit logging for the Kubernetes API server?"
   ```

6. **Cross-component question**:
   ```
   /slack:answer-question "How does authentication work with the API server when using custom certificates?"
   ```

7. **Question with Jira reference**:
   ```
   /slack:answer-question "What's the status of the issue described in OCPBUGS-12345 about etcd performance?"
   ```

## Arguments

- `$1` (required): The customer question as a quoted string
  - Example: "How do I configure custom certificates for the API server?"

- `$2` (optional): The OpenShift component name
  - Common values: apiserver, networking, storage, etcd, authentication, installer, operators
  - If not provided, will be inferred from the question

- `--slack-url` (optional): Slack thread URL for reference
  - Example: `--slack-url https://redhat.slack.com/archives/C123/p123456789`
  - Used for documentation purposes only

## Notes

### Component Mapping

Common Slack channels to component mappings:
- `#forum-ocp-apiserver` → apiserver, api
- `#forum-ocp-etcd` → etcd
- `#forum-ocp-networking` → networking, ovn, sdn
- `#forum-ocp-storage` → storage
- `#forum-ocp-authentication` → authentication, oauth
- `#forum-ocp-installer` → installer

### Search Strategy

When searching repositories:
1. Start with the most specific repository for the component
2. Fall back to openshift/origin for general questions
3. Check openshift/openshift-docs for documentation examples
4. Look at recent issues/PRs for common problems

### Documentation Versions

- Default to the latest stable OpenShift version (4.17 as of early 2025)
- If the question mentions a specific version, search that version's docs
- Note version differences in the answer if significant

### Best Practices

- Prioritize official documentation over code when both exist
- Verify code examples are from recent commits (not deprecated)
- Include version information for configuration examples
- Mention if behavior differs between versions
- Provide both declarative (YAML) and imperative (CLI) approaches when applicable
