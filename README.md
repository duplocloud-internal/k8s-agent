# Kubernetes AI Troubleshooting Agent

An AI-powered agent for troubleshooting Kubernetes clusters using natural language queries, available as a REST API.

## Features

- Natural language interface for Kubernetes troubleshooting
- Powered by AWS Bedrock (Claude 3 Haiku) to interpret your requests
- Suggests appropriate kubectl commands based on your questions
- Maintains conversation context for coherent troubleshooting
- Supports namespace-specific diagnostics
- Available as a REST API

## Project Structure

```
k8s-agent/
├── common/         # Common utilities
│   └── llm.py      # LLM integration with AWS Bedrock
└── k8s/            # Kubernetes agent implementation
    ├── k8s_api_agent.py   # REST API implementation
    └── requirements.txt   # Python dependencies
```

## Requirements

- Python 3.7+
- kubectl configured with access to your Kubernetes cluster
- AWS credentials with access to Bedrock (Claude 3 Haiku model must be enabled in your AWS account and region)

## Installation

1. Clone this repository
2. Install dependencies:
   ```
   pip install -r k8s/requirements.txt
   ```
3. Set your AWS credentials (not needed when running in DuploCloud):
   ```
   # Create a .env file or set environment variables
   AWS_ACCESS_KEY_ID=your-access-key
   AWS_SECRET_ACCESS_KEY=your-secret-key
   AWS_REGION=us-east-1  # Region where Bedrock/Claude is enabled
   ```

## Usage

### REST API

Start the REST API server:
```
python k8s/k8s_api_agent.py
```

This starts a Flask server on port 5002 with the following endpoints:

#### Send Message API

**Endpoint:** `POST /api/sendMessage`

**Request and Response Structure:**

Both requests and responses use the same structure, with `Cmds` as an array of objects containing `Command` and `Output` fields, nested under a `data` field.

**Request Body Options:**

1. Regular question:
```json
{
  "Content": "This is the user message",
  "thread_id": "optional-thread-id-for-conversation-context",
  "data": {
    "Cmds": [],
    "kubeconfig": "base64-encoded-kubeconfig-content"
  }
}
```

> **Note:** The `kubeconfig` field inside the `data` object is optional. If provided, it will be used for this specific user/thread. If not provided, the system will use the default configuration. AWS credentials are automatically detected from environment variables or instance metadata when running in DuploCloud.

2. Command execution (manual):
```json
{
  "Content": "run: kubectl get pods -n kube-system",
  "thread_id": "optional-thread-id-for-conversation-context",
  "data": {
    "Cmds": [],
    "kubeconfig": "base64-encoded-kubeconfig-content"
  }
}
```

3. Command analysis (for commands run outside the agent):
```json
{
  "Content": "Please analyze this command output",
  "thread_id": "optional-thread-id-for-conversation-context",
  "data": {
    "Cmds": [
      {
        "Command": "kubectl get pods -n kube-system",
        "Output": "NAME                                  READY   STATUS    RESTARTS   AGE\ncoredns-5d78c9869d-q8s9h              1/1     Running   0          45d\nkube-proxy-wlqbg                      1/1     Running   0          45d"
      }
    ],
    "kubeconfig": "base64-encoded-kubeconfig-content"
  }
}
```

4. Auto-execute all suggested commands (global execute flag):
```json
{
  "Content": "Check for failed pods in all namespaces",
  "thread_id": "optional-thread-id-for-conversation-context",
  "data": {
    "Cmds": [],
    "kubeconfig": "base64-encoded-kubeconfig-content",
    "execute": true
  }
}
```

5. Execute specific commands only:
```json
{
  "Content": "Check for failed pods in all namespaces",
  "thread_id": "optional-thread-id-for-conversation-context",
  "data": {
    "Cmds": [
      {
        "Command": "kubectl get pods --all-namespaces",
        "Output": "",
        "execute": true
      },
      {
        "Command": "kubectl get nodes",
        "Output": "",
        "execute": false
      }
    ],
    "kubeconfig": "base64-encoded-kubeconfig-content"
  }
}
```

**Response:**
```json
{
  "Content": "This is the message from the agent",
  "thread_id": "conversation-thread-id",
  "data": {
    "Cmds": [
      {
        "Command": "kubectl get deployments -n kube-system",
        "Output": "",
        "execute": false
      },
      {
        "Command": "kubectl describe pod coredns-5d78c9869d-q8s9h -n kube-system",
        "Output": "",
        "execute": false
      }
    ]
  }
}
```

**Response with executed commands:**
```json
{
  "Content": "This is the message from the agent with executed command results",
  "thread_id": "conversation-thread-id",
  "data": {
    "Cmds": [
      {
        "Command": "kubectl get deployments -n kube-system",
        "Output": "NAME      READY   UP-TO-DATE   AVAILABLE   AGE\ncoredns   2/2     2            2           45d",
        "execute": true
      },
      {
        "Command": "kubectl describe pod coredns-5d78c9869d-q8s9h -n kube-system",
        "Output": "",
        "execute": false
      }
    ]
  }
}
```

- If no `thread_id` is provided, a new conversation thread will be created
- To execute a command, send a message with `Content` starting with `run:` followed by the kubectl command
- To analyze command output from commands run outside the agent, include the commands and their outputs in the `Cmds` array
- In responses, the `Cmds` array contains suggested kubectl commands (with empty `Output` fields) and executed commands (with populated `Output` fields)

#### Health Check API

**Endpoint:** `GET /api/health?data.kubeconfig=base64-encoded-kubeconfig-content`

Checks the health of the API and verifies kubectl access. You can optionally provide a base64-encoded kubeconfig to test connectivity to a specific cluster.
