# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Running Hook Scripts
All hook scripts use UV single-file scripts and should be run with:
```bash
uv run .claude/hooks/<hook_name>.py [options]
```

### Testing Hooks
To test individual hooks manually:
```bash
# Test user prompt submission hook
echo '{"prompt": "test prompt", "session_id": "test-123"}' | uv run .claude/hooks/user_prompt_submit.py --log-only

# Test pre-tool-use hook with dangerous command
echo '{"tool_name": "Bash", "tool_input": {"command": "rm -rf /"}}' | uv run .claude/hooks/pre_tool_use.py

# Test stop hook
echo '{"stop_hook_active": false}' | uv run .claude/hooks/stop.py --chat
```

### Running Example Apps
```bash
# Python example
python apps/hello.py

# TypeScript example (requires Node.js)
node apps/hello.ts
```

## Architecture

### Claude Code Hooks System
This repository demonstrates all 6 Claude Code hooks with intelligent control flow and TTS capabilities. The hooks intercept and control Claude Code behavior at critical points:

1. **UserPromptSubmit** - Validates/enhances prompts before Claude processes them
2. **PreToolUse** - Security gate for tool execution (can block dangerous commands)  
3. **PostToolUse** - Processes tool results and logs
4. **Notification** - Handles Claude notifications with optional TTS
5. **Stop** - AI-generated completion messages with TTS
6. **SubagentStop** - Handles sub-agent completions

### UV Single-File Scripts Pattern
All hooks are self-contained Python scripts with embedded dependencies declared using UV's script format:
```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = ["package-name"]
# ///
```

This ensures:
- No virtual environment management needed
- Dependencies isolated per script
- Fast execution with UV's caching
- Portable across environments

### Hook Exit Code Control
- **Exit 0**: Success, stdout shown in transcript mode
- **Exit 2**: Blocking error - prevents action and sends stderr to Claude
- **Other**: Non-blocking error - shows stderr but continues

### TTS Priority System
The TTS utilities try providers in order:
1. ElevenLabs (via MCP if available)
2. OpenAI TTS
3. pyttsx3 (fallback)

### Sub-Agent System
- **Project agents**: `.claude/agents/` (project-specific)
- **User agents**: `~/.claude/agents/` (global)
- **Meta-agent**: Generates new agents from descriptions

## Key Patterns

### Security Validation
PreToolUse hook blocks dangerous patterns:
- `rm -rf` commands
- Access to `.env` files
- System directory modifications
- Dangerous chmod operations

### Logging Strategy
All hooks log to `logs/` directory:
- Each hook type has its own JSON log file
- Logs append across sessions (except chat.json which overwrites)
- Structured JSON format for easy parsing

### Context Injection
UserPromptSubmit can add context that Claude sees:
- Project metadata
- Security warnings
- Custom instructions
- Timestamp information

## Environment Variables

Optional environment variables for enhanced features:
- `ENGINEER_NAME`: Personalizes TTS messages
- `OPENAI_API_KEY`: Enables OpenAI TTS and LLM features
- `ANTHROPIC_API_KEY`: Enables Anthropic LLM features
- `ELEVENLABS_API_KEY`: Enables ElevenLabs TTS (if using direct API)

## Working with Hooks

When modifying hooks:
1. Check `.claude/settings.json` for hook configuration
2. Test hooks manually with JSON input via stdin
3. Use appropriate exit codes for control flow
4. Log to `logs/` directory for debugging
5. Follow UV single-file script format for dependencies

When creating new hooks:
1. Create script in `.claude/hooks/`
2. Add UV script header with dependencies
3. Read JSON from stdin
4. Use exit code 2 to block actions
5. Print JSON for advanced control
6. Update `.claude/settings.json` to register hook