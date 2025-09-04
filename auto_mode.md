# AUTO_MODE Implementation Guide

## Overview

This document provides a comprehensive explanation of the AUTO_MODE feature implementation in the NANDA adapter. AUTO_MODE enables agents to automatically respond to incoming messages from other agents without requiring user intervention, creating fully autonomous agent-to-agent communication.

## Table of Contents

1. [Implementation Overview](#implementation-overview)
2. [Code Changes Breakdown](#code-changes-breakdown)
3. [Message Flow Analysis](#message-flow-analysis)
4. [Configuration and Usage](#configuration-and-usage)
5. [Testing and Validation](#testing-and-validation)

---

## Implementation Overview

### What is AUTO_MODE?

AUTO_MODE is an environment variable flag that changes how the NANDA agent handles incoming external messages:

- **AUTO_MODE=false (default):** External messages are forwarded to the user/UI for manual handling
- **AUTO_MODE=true:** External messages are processed directly by Claude and responded to automatically

### Key Benefits

- **Autonomous Communication:** Enables true agent-to-agent conversations without human intervention
- **Scalability:** Allows agents to operate independently in large networks
- **Backward Compatibility:** Default behavior remains unchanged for existing users
- **Flexibility:** Can be toggled per agent instance based on use case

---

## Code Changes Breakdown

### 1. Environment Variable Flag Addition

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 28-29

```python
# Toggle for message improvement feature
IMPROVE_MESSAGES = os.getenv("IMPROVE_MESSAGES", "true").lower() in ("true", "1", "yes", "y")

# Toggle for auto mode - when enabled, agent answers questions directly without asking user
AUTO_MODE = os.getenv("AUTO_MODE", "false").lower() in ("true", "1", "yes", "y")
```

**Explanation:**
- Added `AUTO_MODE` environment variable that defaults to `false`
- Uses the same boolean parsing logic as existing `IMPROVE_MESSAGES` flag
- Supports multiple formats: `"true"`, `"1"`, `"yes"`, `"y"` (case-insensitive)
- Placed near other configuration flags for consistency

### 2. System Prompt Enhancement

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 55-58

```python
# Configure system prompts based on agent ID (examples from the original code)
SYSTEM_PROMPTS = {
    "default": "You are Claude assisting a user (Agent). Assume the messages you get are part of a conversation with other agents. Help the user communicate effectively with other agents.",
    "auto_mode": "You are an autonomous AI agent. When you receive questions or requests, answer them directly and comprehensively without asking the user for clarification or confirmation. Provide complete, helpful responses based on your knowledge and capabilities. Act independently and decisively."
}
```

**Explanation:**
- Extended existing `SYSTEM_PROMPTS` dictionary with `"auto_mode"` prompt
- **Default prompt:** Focuses on collaborative assistance and communication facilitation
- **Auto mode prompt:** Emphasizes autonomous behavior, direct responses, and independent decision-making
- Instructs Claude to avoid asking for clarification when in autonomous mode

### 3. Dynamic System Prompt Selection

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 166-167

**Before:**
```python
# Use the agent's specific prompt if available, otherwise use default
system = SYSTEM_PROMPTS["default"]
```

**After:**
```python
# Use auto mode prompt if AUTO_MODE is enabled, otherwise use default
system = SYSTEM_PROMPTS["auto_mode"] if AUTO_MODE else SYSTEM_PROMPTS["default"]
```

**Explanation:**
- Modified the `call_claude()` function to automatically select appropriate system prompt
- Uses conditional logic to switch between collaborative and autonomous behavior
- Only applies when no explicit system prompt is provided as a parameter
- Ensures consistent behavior across all Claude API calls

### 4. Core Auto-Response Logic

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 503-537

This is the heart of the AUTO_MODE implementation. Here's the complete logic:

```python
print("Message Text: ", message_content)
print("UI MODE: ", UI_MODE)
print("AUTO_MODE: ", AUTO_MODE)

# If AUTO_MODE is enabled, process the message directly with Claude and respond
if AUTO_MODE:
    print(f"AUTO_MODE enabled: Processing message directly with Claude")
    agent_id = get_agent_id()
    
    # Process the message content with Claude using auto mode system prompt
    claude_response = call_claude(
        message_content, 
        "", 
        conversation_id, 
        f"external>{from_agent}>{agent_id}",
        SYSTEM_PROMPTS["auto_mode"]
    )
    
    if claude_response:
        # Send the Claude response back to the originating agent
        response_text = f"Agent {agent_id} response: {claude_response}"
        
        # Log the auto response
        log_message(conversation_id, f"external>{from_agent}>{agent_id}", f"Auto Agent {agent_id}", claude_response)
        
        return Message(
            role=MessageRole.AGENT,
            content=TextContent(text=response_text),
            parent_message_id=msg.message_id,
            conversation_id=conversation_id
        )
    else:
        # If Claude fails, fall back to acknowledgment
        return Message(
            role=MessageRole.AGENT,
            content=TextContent(text=f"Agent {agent_id} processed your message but couldn't generate a response"),
            parent_message_id=msg.message_id,
            conversation_id=conversation_id
        )

# If in UI mode, forward to all registered UI clients
elif UI_MODE:
    print(f"Forwarding message to UI client")
    send_to_ui_client(formatted_text, from_agent, conversation_id)
    
    # Acknowledge receipt to sender
    agent_id = get_agent_id()
    return Message(
        role=MessageRole.AGENT,
        content=TextContent(text=f"Message received by Agent {agent_id}"),
        parent_message_id=msg.message_id,
        conversation_id=conversation_id
    )
```

**Explanation:**

1. **Debug Logging:** Added `AUTO_MODE` status to debug output for troubleshooting
2. **Conditional Processing:** 
   - `if AUTO_MODE:` - Process message with Claude directly
   - `elif UI_MODE:` - Forward to UI (existing behavior)
   - `else:` - Forward to terminal (existing behavior)
3. **Claude Integration:** 
   - Calls `call_claude()` with the external message content
   - Uses explicit `auto_mode` system prompt for autonomous behavior
   - Generates path string for logging: `f"external>{from_agent}>{agent_id}"`
4. **Response Handling:**
   - Formats Claude's response with agent ID prefix
   - Returns proper `Message` object for A2A protocol compliance
   - Maintains conversation threading with `parent_message_id`
5. **Error Handling:** Graceful fallback if Claude API fails
6. **Audit Trail:** Logs all autonomous responses for debugging and compliance

### 5. Status Display Updates

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 937-938

```python
print(f"Message improvement feature is {'ENABLED' if IMPROVE_MESSAGES else 'DISABLED'}")
print(f"Auto mode is {'ENABLED' if AUTO_MODE else 'DISABLED'}")
print(f"Logging conversations to {os.path.abspath(LOG_DIR)}")
```

**File:** `nanda_adapter/core/nanda.py`
**Lines:** 98-103

```python
# Start the server
IMPROVE_MESSAGES = os.getenv("IMPROVE_MESSAGES", "true").lower() in ("true", "1", "yes", "y")
AUTO_MODE = os.getenv("AUTO_MODE", "false").lower() in ("true", "1", "yes", "y")

print(f"\n🚀 Starting Agent {AGENT_ID} bridge on port {PORT}")
print(f"Agent terminal port: {TERMINAL_PORT}")
print(f"Message improvement feature is {'ENABLED' if IMPROVE_MESSAGES else 'DISABLED'}")
print(f"Auto mode is {'ENABLED' if AUTO_MODE else 'DISABLED'}")
```

**Explanation:**
- Added status display to both main entry points (`agent_bridge.py` and `nanda.py`)
- Users can immediately verify AUTO_MODE status at startup
- Consistent formatting with existing feature flags
- Helps with debugging and configuration validation

### 6. Help Command Enhancement

**File:** `nanda_adapter/core/agent_bridge.py`
**Lines:** 845-853

**Before:**
```python
help_text = """Available commands:
    /help - Show this help message
    /quit - Exit the terminal
    /query [message] - Get a response from the agent privately
    @<agent_id> [message] - Send a message to a specific agent"""
```

**After:**
```python
auto_status = "ENABLED" if AUTO_MODE else "DISABLED"
help_text = f"""Available commands:
    /help - Show this help message
    /quit - Exit the terminal
    /query [message] - Get a response from the agent privately
    @<agent_id> [message] - Send a message to a specific agent
    
Current settings:
    Auto Mode: {auto_status} - When enabled, agent responds directly to external messages"""
```

**Explanation:**
- Enhanced `/help` command to show runtime AUTO_MODE status
- Provides contextual explanation of what AUTO_MODE does
- Helps users understand current agent behavior during operation
- Uses dynamic status checking rather than static text

### 7. Documentation Updates

**File:** `README.md`
**Lines:** 250, 282-304

Added AUTO_MODE to environment variables list:
```markdown
- `AUTO_MODE`: Enable/disable autonomous response mode (optional, default: false)
```

Added comprehensive Auto Mode section:
```markdown
### Auto Mode

When `AUTO_MODE=true` is set, agents will automatically respond to incoming messages from other agents without requiring user intervention. This enables fully autonomous agent-to-agent communication.

**Behavior with AUTO_MODE enabled:**
- External messages from other agents are processed directly by Claude
- The agent provides immediate, comprehensive responses
- No user input or confirmation is required
- Responses are automatically sent back to the originating agent

**Behavior with AUTO_MODE disabled (default):**
- External messages are forwarded to the user/UI for manual handling
- User decides how to respond to incoming messages
- Traditional interactive agent behavior

**Example usage:**
```bash
# Enable autonomous mode
export AUTO_MODE=true

# Disable autonomous mode (default)
export AUTO_MODE=false
```
```

**Explanation:**
- Added AUTO_MODE to the environment variables reference section
- Created dedicated documentation section explaining the feature
- Provided clear behavior comparison between enabled/disabled states
- Included practical usage examples
- Emphasized default behavior for backward compatibility

---

## Message Flow Analysis

### Traditional Message Flow (AUTO_MODE=false)

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Agent A │───▶│ Agent B │───▶│ User/UI │───▶│ Agent A │
│         │    │ Bridge  │    │         │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
     │              │              │              │
     │              │              │              │
  Sends msg    Forwards to     Manual         Response
               User/UI      Response/Action    sent back
```

**Steps:**
1. Agent A sends message to Agent B
2. Agent B receives message via `handle_external_message()`
3. Message is forwarded to user interface or terminal
4. Human user reads message and decides on response
5. User types response manually
6. Response is sent back to Agent A

### Autonomous Message Flow (AUTO_MODE=true)

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Agent A │───▶│ Agent B │───▶│ Claude  │───▶│ Agent A │
│         │    │ Bridge  │    │   API   │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
     │              │              │              │
     │              │              │              │
  Sends msg    Auto-processes   Generates      Response
               with Claude      Response       sent back
```

**Steps:**
1. Agent A sends message to Agent B
2. Agent B receives message via `handle_external_message()`
3. AUTO_MODE check passes, message is sent directly to Claude API
4. Claude generates autonomous response using `auto_mode` system prompt
5. Response is automatically sent back to Agent A
6. All interactions are logged for audit trail

### Code Flow in `handle_external_message()`

```python
def handle_external_message(msg_text, conversation_id, msg):
    # Parse external message format
    # Extract: from_agent, to_agent, message_content
    
    if AUTO_MODE:
        # 🤖 AUTONOMOUS PATH
        claude_response = call_claude(
            message_content,
            "",
            conversation_id, 
            f"external>{from_agent}>{agent_id}",
            SYSTEM_PROMPTS["auto_mode"]  # Key difference!
        )
        return Message(response_text)  # Direct response
        
    elif UI_MODE:
        # 👤 UI PATH (Traditional)
        send_to_ui_client(formatted_text, from_agent, conversation_id)
        return Message("Message received")  # Just acknowledgment
        
    else:
        # 💻 TERMINAL PATH (Traditional)  
        terminal_client.send_message_threaded(formatted_text)
        return Message("Message received")  # Just acknowledgment
```

---

## Configuration and Usage

### Environment Variable Setup

```bash
# Enable autonomous mode
export AUTO_MODE=true
export ANTHROPIC_API_KEY="your-anthropic-api-key"

# Optional: Configure other settings
export AGENT_ID="my_auto_agent"
export PORT=6000
export IMPROVE_MESSAGES=true
```

### NANDA Integration Example

```python
from nanda_adapter import NANDA
import os

def create_custom_improvement():
    def custom_logic(message_text: str) -> str:
        return f"Enhanced: {message_text}"
    return custom_logic

# Create agent with custom logic
nanda = NANDA(create_custom_improvement())

# Start with AUTO_MODE support
anthropic_key = os.getenv("ANTHROPIC_API_KEY")
domain = os.getenv("DOMAIN_NAME")

# AUTO_MODE will be automatically detected from environment
nanda.start_server_api(anthropic_key, domain)
```

### Runtime Verification

When starting an agent, you'll see status output:

```
🚀 Starting Agent agent123 bridge on port 6000
Agent terminal port: 6010
Message improvement feature is ENABLED
Auto mode is ENABLED                    # ← AUTO_MODE status
Logging conversations to /path/logs
🔧 Using custom improvement logic: custom_logic
```

### Testing AUTO_MODE

1. **Start Agent A with AUTO_MODE=true:**
```bash
export AUTO_MODE=true
export AGENT_ID=agent_auto
python agent_a.py
```

2. **Start Agent B normally:**
```bash
export AGENT_ID=agent_normal  
python agent_b.py
```

3. **Send message from Agent B to Agent A:**
```
@agent_auto Hello, can you help me with a calculation?
```

4. **Expected behavior:**
- Agent A receives the message
- Claude processes it with autonomous prompt
- Agent A responds directly: "Agent agent_auto response: I'd be happy to help with calculations! What specific calculation do you need assistance with?"

---

## Testing and Validation

### Automated Configuration Test

Created `test_auto_mode_simple.py` to validate implementation:

```python
def test_auto_mode_in_agent_bridge():
    """Test that AUTO_MODE flag is added to agent_bridge.py"""
    with open("nanda_adapter/core/agent_bridge.py", 'r') as f:
        content = f.read()
    
    assert 'AUTO_MODE = os.getenv("AUTO_MODE"' in content
    assert '"auto_mode":' in content  
    assert 'SYSTEM_PROMPTS["auto_mode"] if AUTO_MODE' in content
    assert 'if AUTO_MODE:' in content
    print("✅ All AUTO_MODE implementations verified")
```

### Manual Testing Scenarios

1. **Default Behavior Test (AUTO_MODE=false):**
   - External messages should forward to UI/terminal
   - User should see: "FROM agent123: Hello there!"
   - Manual response required

2. **Autonomous Behavior Test (AUTO_MODE=true):**
   - External messages should get automatic Claude responses
   - Sender should receive: "Agent agent456 response: [Claude's response]"
   - No user interaction required

3. **System Prompt Test:**
   - With AUTO_MODE=false: Claude acts collaboratively
   - With AUTO_MODE=true: Claude acts autonomously and decisively

### Debugging Tools

1. **Debug Logging:**
```python
print("Message Text: ", message_content)
print("UI MODE: ", UI_MODE) 
print("AUTO_MODE: ", AUTO_MODE)  # ← Added for debugging
```

2. **Conversation Logs:**
```python
log_message(conversation_id, f"external>{from_agent}>{agent_id}", f"Auto Agent {agent_id}", claude_response)
```

3. **Help Command Status:**
```
/help
# Shows: Auto Mode: ENABLED - When enabled, agent responds directly to external messages
```

---

## Key Design Decisions

### 1. Backward Compatibility
- **Decision:** Default AUTO_MODE=false
- **Rationale:** Existing users experience no behavior change
- **Implementation:** `os.getenv("AUTO_MODE", "false")`

### 2. Conditional Logic Structure
- **Decision:** Use `if AUTO_MODE:` / `elif UI_MODE:` / `else:` structure
- **Rationale:** Clear precedence and mutually exclusive paths
- **Implementation:** AUTO_MODE takes priority over UI_MODE

### 3. Explicit System Prompt
- **Decision:** Always pass `SYSTEM_PROMPTS["auto_mode"]` in AUTO_MODE
- **Rationale:** Ensure consistent autonomous behavior regardless of default prompt selection
- **Implementation:** Explicit parameter in `call_claude()`

### 4. Response Format
- **Decision:** Prefix responses with "Agent {agent_id} response:"
- **Rationale:** Clear identification of autonomous responses vs manual responses
- **Implementation:** `f"Agent {agent_id} response: {claude_response}"`

### 5. Error Handling Strategy
- **Decision:** Graceful fallback to acknowledgment if Claude fails
- **Rationale:** Maintain A2A protocol compliance even during API failures
- **Implementation:** `else:` block with fallback message

---

## Benefits and Use Cases

### Benefits

1. **Scalability:** Agents can operate in large networks without human bottlenecks
2. **24/7 Operation:** Autonomous agents can respond at any time
3. **Consistency:** Claude provides consistent, high-quality responses
4. **Flexibility:** Per-agent configuration allows mixed autonomous/manual networks
5. **Audit Trail:** All autonomous interactions are logged

### Use Cases

1. **Customer Support Networks:** Autonomous agents handle common queries
2. **Research Collaboration:** Agents share findings and ask questions autonomously  
3. **Task Coordination:** Agents negotiate and coordinate work automatically
4. **Information Gathering:** Agents autonomously request and provide data
5. **Educational Networks:** Teaching agents provide explanations and answers

---

## Future Enhancements

### Potential Improvements

1. **Selective AUTO_MODE:** Enable/disable per conversation or sender
2. **Response Filtering:** Configure which message types get autonomous responses
3. **Confidence Thresholds:** Only respond autonomously if Claude is confident
4. **Custom Auto Prompts:** Allow per-agent autonomous system prompts
5. **Response Templates:** Structured autonomous response formats

### Configuration Extensions

```python
# Future configuration options
AUTO_MODE_SELECTIVE = {
    "agents": ["trusted_agent1", "trusted_agent2"],  # Only auto-respond to these
    "message_types": ["question", "request"],         # Only for certain types
    "confidence_threshold": 0.8                       # Only if Claude is confident
}
```

This implementation provides a solid foundation for autonomous agent communication while maintaining full backward compatibility and extensive configurability.