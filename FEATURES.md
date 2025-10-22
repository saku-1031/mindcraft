# Mindcraft Features and Capabilities Documentation

This is a comprehensive guide to all features and capabilities available in the Mindcraft project - an AI-powered Minecraft bot framework powered by LLMs and Mineflayer.

---

## Table of Contents

1. [Command System](#command-system)
2. [Actions (Skills)](#actions-skills)
3. [Query Commands](#query-commands)
4. [Mode System](#mode-system)
5. [Core Features](#core-features)
6. [Settings and Configuration](#settings-and-configuration)
7. [Advanced Features](#advanced-features)
8. [Model and LLM Support](#model-and-llm-support)

---

## Command System

The command system allows the AI agent to interact with the Minecraft world through well-defined, structured commands.

**Implementation**: `/src/agent/commands/index.js`

### Command Structure
- Commands follow the syntax: `!commandName` or `!commandName("arg1", 1.2, ...)`
- All arguments must be properly typed (int, float, boolean, string, BlockName, ItemName)
- Commands are parsed and validated before execution
- Only one command per response is processed; trailing commands are ignored
- Unblockable commands: `!stop`, `!stats`, `!inventory`, `!goal`

### Command Blacklisting
- Commands can be disabled via the `blocked_actions` setting in `settings.js`
- Useful for restricting dangerous commands or those not needed for specific tasks
- Blacklisting prevents both execution and documentation display

---

## Actions (Skills)

Actions are commands that perform tasks in the Minecraft world. There are 41 action commands available.

**Implementation**: `/src/agent/commands/actions.js`

### Navigation Actions

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!goToPlayer` | player_name, closeness | Navigate to a specific player |
| `!followPlayer` | player_name, follow_dist | Endlessly follow a player at specified distance |
| `!goToCoordinates` | x, y, z, closeness | Navigate to specific x, y, z coordinates |
| `!searchForBlock` | type, search_range | Find and go to nearest block of given type |
| `!searchForEntity` | type, search_range | Find and go to nearest entity of given type |
| `!moveAway` | distance | Move away from current location in any direction |
| `!goToRememberedPlace` | name | Go to a previously saved location |
| `!goToBed` | - | Go to and sleep in nearest bed |
| `!goToSurface` | - | Move to highest block above current position |
| `!digDown` | distance | Dig downward specified distance (stops at lava/water/falls) |

### Interaction & Building Actions

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!placeHere` | type | Place a block at current location |
| `!collectBlocks` | type, num | Collect nearest blocks of given type |
| `!craftRecipe` | recipe_name, num | Craft recipe specified number of times |
| `!smeltItem` | item_name, num | Smelt item specified number of times |
| `!clearFurnace` | - | Take all items out of nearest furnace |
| `!useOn` | tool_name, target | Use tool on nearest target of given type |

### Inventory Management

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!equip` | item_name | Equip specified item |
| `!consume` | item_name | Eat/drink specified item |
| `!givePlayer` | player_name, item_name, num | Give items to a player |
| `!discard` | item_name, num | Discard items from inventory |
| `!putInChest` | item_name, num | Put items in nearest chest |
| `!takeFromChest` | item_name, num | Take items from nearest chest |
| `!viewChest` | - | View contents of nearest chest |

### Combat Actions

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!attack` | type | Attack and kill nearest entity of given type |
| `!attackPlayer` | player_name | Attack specific player until they die/run |

### Social & Communication

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!startConversation` | player_name, message | Start conversation with bot (for multi-agent) |
| `!endConversation` | player_name | End conversation with bot |
| `!lookAtPlayer` | player_name, direction | Look at or look with player (requires vision) |
| `!lookAtPosition` | x, y, z | Look at specified coordinates (requires vision) |

### Trade & NPC Interaction

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!showVillagerTrades` | id | Show trades of specified villager |
| `!tradeWithVillager` | id, index, count | Execute trade with villager |

### Memory & Goal Management

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!rememberHere` | name | Save current location with given name |
| `!goal` | selfPrompt | Set goal for endless self-prompting |
| `!endGoal` | - | Stop self-prompting and current action |
| `!setMode` | mode_name, on | Enable or disable automatic behavior mode |
| `!stay` | seconds | Stay in place (pauses modes), -1 for forever |

### System & Meta Commands

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!newAction` | prompt | Generate and execute custom code (requires allow_insecure_coding) |
| `!stop` | - | Force stop all executing actions |
| `!stfu` | - | Stop chatting and self-prompting but continue action |
| `!restart` | - | Restart the agent process |
| `!clearChat` | - | Clear chat history |

---

## Query Commands

Query commands return information about the world without modifying it. There are 14 query commands available.

**Implementation**: `/src/agent/commands/queries.js`

### Information Queries

| Command | Description |
|---------|-------------|
| `!stats` | Get bot location, health, hunger, time of day, biome, weather, current action |
| `!inventory` | Get bot inventory contents and armor |
| `!nearbyBlocks` | Get blocks near the bot with distance info |
| `!entities` | Get nearby players, bots, and mobs (with villager details) |
| `!craftable` | Get list of craftable items with current inventory |
| `!modes` | Get all available modes and their on/off status |

### Location & Blueprint Queries

| Command | Description |
|---------|-------------|
| `!savedPlaces` | List all saved location names |
| `!checkBlueprint` | Check what blocks need to be placed for current blueprint |
| `!checkBlueprintLevel` | Check completion status for specific blueprint level |
| `!getBlueprint` | Get full blueprint explanation |
| `!getBlueprintLevel` | Get blueprint explanation for specific level |

### Planning & Reference

| Command | Parameters | Description |
|---------|-----------|-------------|
| `!getCraftingPlan` | targetItem, quantity | Get comprehensive crafting plan with ingredient analysis |
| `!searchWiki` | query | Search Minecraft Wiki for information |
| `!help` | - | List all available commands with syntax |

---

## Mode System

Modes are automatic behaviors that constantly monitor the environment and respond without requiring explicit commands.

**Implementation**: `/src/agent/modes.js`

### Available Modes

| Mode | Description | Interrupts | Default |
|------|-------------|-----------|---------|
| `self_preservation` | Respond to drowning, burning, low health | All | ON |
| `unstuck` | Escape when stuck in same place | Some | ON |
| `cowardice` | Run away from hostile mobs | All | ON |
| `self_defense` | Attack nearby hostile mobs | All | ON |
| `hunting` | Hunt animals when idle | followPlayer action | ON |
| `item_collecting` | Pick up nearby items when idle | followPlayer action | ON |
| `torch_placing` | Place torches in dark areas when idle | followPlayer action | ON |
| `elbow_room` | Move away from other players when idle | followPlayer action | ON |
| `idle_staring` | Look around at nearby entities | None | ON |
| `cheat` | Use cheats for instant block placement & teleport | None | OFF by default |

### Mode Management
- Enable/disable modes with `!setMode(mode_name, true/false)`
- View all modes with `!modes`
- Modes can interrupt actions based on priority rules
- Some modes will pause when certain actions are running

---

## Core Features

### 1. Self-Prompting / Autonomous Behavior

**Implementation**: `/src/agent/self_prompter.js`

The agent can autonomously work toward goals without constant user interaction.

**Features**:
- Set goals with `!goal("your goal description")`
- Agent automatically self-prompts to work toward goal
- Stops after 3 consecutive responses without commands
- Can be paused and resumed
- State persistence (Active/Paused/Stopped)
- Cooldown between self-prompts (2000ms default)
- Memory of goal prompt maintained

**Configuration in settings.js**:
- `max_messages`: Maximum messages to keep in context before summarizing
- `num_examples`: Number of examples to provide for prompting
- `show_command_syntax`: Control command display ("full", "shortened", "none")

### 2. Vision Capabilities

**Implementation**: `/src/agent/vision/vision_interpreter.js`, `/src/agent/vision/camera.js`

Enables the agent to "see" the Minecraft world and analyze screenshots.

**Features**:
- Take screenshots from bot's perspective
- Analyze images using vision models (e.g., GPT-4V)
- Look at specific players or coordinates
- Get center block info in view
- Combine vision analysis with text description

**Commands**:
- `!lookAtPlayer(name, "at"|"with")`: Look at player and analyze
- `!lookAtPosition(x, y, z)`: Look at coordinates and analyze

**Configuration**:
- Enable in `settings.js`: `allow_vision: true`
- Vision model specified in profile: `vision_model: "gpt-4o"`
- Screenshot directory: `./bots/{botname}/screenshots/`

### 3. Code Generation & Execution

**Implementation**: `/src/agent/coder.js`

The agent can write and execute custom JavaScript code for complex tasks.

**Features**:
- LLM generates custom code based on context
- Code is sandboxed in secure compartment (using SES lockdown)
- ESLint validation before execution
- Function availability checking
- Retry mechanism (up to 5 attempts)
- Code output logging and error reporting
- Supported functions: `skills.*`, `world.*`, `Vec3`, `log()`

**Command**:
- `!newAction("detailed description of what you want")`: Generate custom code

**Security**:
- Disabled by default (enable with `allow_insecure_coding: true`)
- Sandboxed execution environment
- Only has access to approved functions
- Interrupt mechanism if code hangs

**Configuration**:
- `allow_insecure_coding`: Enable/disable code generation
- `code_timeout_mins`: Time limit for code execution
- Code files stored in `./bots/{botname}/action-code/`

### 4. Memory System

**Implementation**: `/src/agent/memory_bank.js`, `/src/agent/history.js`

Multi-layered memory system for long-term learning.

**Features**:
- **Short-term**: Current conversation context (configurable max_messages)
- **Working memory**: Recent turns maintained in history
- **Long-term**: Summarized memories of older conversations
- **Persistent**: Saved to disk at `./bots/{botname}/memory.json`
- **Recall**: Save/recall named locations with `!rememberHere` / `!goToRememberedPlace`
- **Memory summarization**: Older messages summarized to save tokens

**Configuration**:
- `load_memory: true/false`: Load previous session memory
- `max_messages: 15`: Keep this many messages before summarizing
- Memory summaries auto-compress to 500 chars

### 5. Conversation Management

**Implementation**: `/src/agent/conversation.js`

Manages multi-agent conversations and interaction flows.

**Features**:
- Support for multiple active agents
- Conversation states (active, queued, ended)
- Per-agent message queueing
- Automatic conversation detection
- Command-based conversation control
- Multi-turn conversation support
- Integration with main message handler

**Commands**:
- `!startConversation(botname, "message")`: Start talking to another bot
- `!endConversation(botname)`: End conversation
- Can track which bot is talking using `!msg @botname message` (in chat)

### 6. NPC Controller

**Implementation**: `/src/agent/npc/controller.js`

Specialized system for controlling NPCs with goals and behaviors.

**Features**:
- **Item Goals**: Gather specified items
- **Build Goals**: Construct predefined structures
- **Villager Integration**: Trade with villagers
- **Construction Templates**: Load pre-defined building blueprints
- **Progress Tracking**: Track what's been built/completed
- **Automatic Execution**: Pursue goals when idle
- **Resume Actions**: Continue interrupted goals

**NPC Profile Configuration**:
- Define in bot profile with `npc` field
- Contains: items to collect, buildings to construct, villagers to trade with
- Construction files stored in `./src/agent/npc/construction/*.json`

---

## Settings and Configuration

**Implementation**: `/settings.js`

### Server Configuration
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `minecraft_version` | string | "auto" | Minecraft version to connect to |
| `host` | string | "127.0.0.1" | Server IP address or hostname |
| `port` | number | 55916 | Server port |
| `auth` | string | "offline" | "offline" or "microsoft" authentication |
| `spawn_timeout` | number | 30 | Seconds to wait for bot spawn |

### UI & Display
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `mindserver_port` | number | 8080 | Web UI port |
| `auto_open_ui` | bool | true | Auto-open browser UI on startup |
| `render_bot_view` | bool | false | Show bot's first-person view (localhost:3000+) |
| `chat_ingame` | bool | true | Show bot responses in Minecraft chat |

### Agent Configuration
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `base_profile` | string | "assistant" | Base profile (survival/assistant/creative/god_mode) |
| `profiles` | array | ["./andy.json"] | Bot profile files |
| `init_message` | string | "Respond with hello..." | Message sent on spawn |
| `only_chat_with` | array | [] | Restrict chat to specific users (empty = all) |

### Language & Speech
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `language` | string | "en" | Translate to/from language |
| `speak` | bool | false | Enable text-to-speech output |

### Memory & Context
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `load_memory` | bool | false | Load memories from previous sessions |
| `max_messages` | number | 15 | Messages kept before summarization |
| `num_examples` | number | 2 | Example count for prompting |
| `relevant_docs_count` | number | 5 | Function docs for context |

### Command & Action Control
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `allow_insecure_coding` | bool | false | Enable newAction code generation |
| `allow_vision` | bool | false | Enable vision/screenshot features |
| `blocked_actions` | array | [...] | Commands to disable |
| `max_commands` | number | -1 | Max consecutive commands (-1 = unlimited) |
| `code_timeout_mins` | number | -1 | Code execution timeout |

### Behavior Control
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `narrate_behavior` | bool | true | Chat automatic actions (picking up items, etc) |
| `chat_bot_messages` | bool | true | Chat messages to other bots |
| `show_command_syntax` | string | "full" | Command display ("full"/"shortened"/"none") |
| `block_place_delay` | number | 0 | Delay between block placements (ms) |
| `log_all_prompts` | bool | false | Log all prompts to file |

---

## Advanced Features

### 1. Task System

**Implementation**: `/src/agent/tasks/tasks.js`, `/src/agent/tasks/construction_tasks.js`, `/src/agent/tasks/cooking_tasks.js`

Structured tasks for evaluating bot performance.

**Task Types**:
- **Construction Tasks**: Build blueprints from JSON definitions
- **Cooking Tasks**: Prepare recipes and food items
- **Item Collection**: Gather specified items

**Features**:
- Blueprint validation with detailed feedback
- Multi-level construction (hierarchical building)
- Progress tracking and reporting
- Task completion detection
- Integration with evaluation framework

**Configuration**:
- Task data in `./tasks/*.json`
- Run with: `python tasks/run_task_file.py --task_path=tasks/example_tasks.json`
- Requires: Python, pip, and optional conda

### 2. Multi-Agent Collaboration

**Implementation**: `/src/agent/conversation.js`, `/src/mindcraft/mindserver.js`

Support for multiple agents working together.

**Features**:
- Multiple agents spawn in same world
- Inter-agent messaging via `!startConversation`/`!endConversation`
- Shared game state awareness
- Distinguish between human and bot players
- Coordinate on tasks via conversation
- Conversation queuing system

**Usage**:
- Configure multiple profiles in `settings.js`
- Each bot has its own LLM context and memory
- Agents aware of other agent locations and inventories

### 3. Browser Viewer

**Implementation**: `/src/agent/vision/browser_viewer.js`

Real-time visualization of bot's perspective.

**Features**:
- First-person view streamed to browser
- Running on localhost:3000, 3001, 3002, etc (one per bot)
- Updates each game tick
- Useful for debugging and monitoring
- Can capture screenshots for vision analysis

**Configuration**:
- Enable in `settings.js`: `render_bot_view: true`
- Access at `http://localhost:3000` (or 3001, 3002 for additional bots)

### 4. Skill Library

**Implementation**: `/src/agent/library/skills.js` (38 functions), `/src/agent/library/world.js` (22 functions)

Low-level skills used by commands and code generation.

**Skill Categories**:

**Movement Skills**:
- `goToGoal()`, `goToPosition()`, `goToNearestBlock()`, `goToNearestEntity()`
- `goToPlayer()`, `followPlayer()`, `moveAway()`, `moveAwayFromEntity()`, `avoidEnemies()`

**Block Interaction**:
- `collectBlock()`, `breakBlockAt()`, `placeBlock()`, `activateNearestBlock()`
- `useDoor()`, `goToNearestBlock()`, `digDown()`, `goToSurface()`

**Inventory Management**:
- `equip()`, `discard()`, `putInChest()`, `takeFromChest()`, `viewChest()`, `consume()`, `giveToPlayer()`
- `pickupNearbyItems()`

**Crafting & Smelting**:
- `craftRecipe()`, `smeltItem()`, `clearNearestFurnace()`, `wait()`

**Combat**:
- `attackNearest()`, `attackEntity()`, `defendSelf()`

**Farming**:
- `tillAndSow()` for planting crops

**Trading**:
- `showVillagerTrades()`, `tradeWithVillager()`, `useToolOn()`

**World Queries** (22 functions in world.js):
- `getNearestBlock()`, `getNearestBlocks()`, `getNearestEntity()`
- `getNearbyPlayers()`, `getNearbyEntities()`, `getInventoryCounts()`
- `getCraftableItems()`, `isEntityType()`, `getVillagerProfession()`
- And many more for world state inspection

### 5. Translation Support

**Implementation**: `/src/utils/translator.js`

Automatic language translation for multilingual support.

**Features**:
- Translate messages to/from specified language
- Supports all Google Translate languages
- Transparent translation in conversation flow
- Preserve code and command syntax during translation
- Configuration in `settings.js`

**Usage**:
- Set `language: "es"` for Spanish, `"fr"` for French, etc.
- Messages automatically translated before processing
- Responses translated back to user's language

### 6. Text-to-Speech (TTS)

**Implementation**: `/src/agent/speak.js`

Output agent responses as audio.

**Features**:
- System TTS: Windows (PowerShell), macOS (say), Linux (espeak)
- Remote TTS: OpenAI, Google Gemini
- Queue-based processing
- FFplay for audio playback
- Configurable voice and model

**Configuration**:
- `speak: false` (default) or set to "system" for local TTS
- Or specify format: `"provider/model/voice"` (e.g., `"openai/tts-1/echo"`)
- In profile: `speak_model` configuration

### 7. Model Providers

**Implementation**: `/src/models/*.js`

Support for 20+ LLM providers.

**Supported Providers**:
| Provider | Models | Config Variable |
|----------|--------|-----------------|
| OpenAI | GPT-4, GPT-4o, GPT-3.5 | `OPENAI_API_KEY` |
| Google Gemini | Gemini 2.0, Gemini 1.5 | `GEMINI_API_KEY` |
| Anthropic Claude | Claude 3.5, 3, Haiku | `ANTHROPIC_API_KEY` |
| XAI Grok | Grok-2, Grok-1 | `XAI_API_KEY` |
| DeepSeek | DeepSeek Chat, R1 | `DEEPSEEK_API_KEY` |
| Groq | Mixtral, Llama | `GROQCLOUD_API_KEY` |
| Mistral | Mistral Large, Nemo | `MISTRAL_API_KEY` |
| Qwen (Alibaba) | Qwen Max, Plus | `QWEN_API_KEY` |
| Replicate | Any Replicate model | `REPLICATE_API_KEY` |
| HuggingFace | Any HF model | `HUGGINGFACE_API_KEY` |
| Novita AI | DeepSeek R1, others | `NOVITA_API_KEY` |
| OpenRouter | Multi-provider aggregator | `OPENROUTER_API_KEY` |
| Ollama | Local models | None (self-hosted) |
| Cerebras | Llama 3.3 | `CEREBRAS_API_KEY` |
| Mercury | Mercury models | `MERCURY_API_KEY` |
| Hyperbolic | DeepSeek V3 | `HYPERBOLIC_API_KEY` |
| GLHF.chat | Meta, Mistral models | `GHLF_API_KEY` |
| vLLM | Local vLLM | None (self-hosted) |
| Azure OpenAI | Azure-hosted models | `OPENAI_API_KEY` |

**Model Configuration**:
- Simple: `"model": "gpt-4o"`
- Advanced: Separate models for chat, code, vision, embeddings
- Custom URLs and parameters

### 8. Embedding Models

**Implementation**: `/src/models/prompter.js`

Used to select relevant examples and skills.

**Supported APIs**: OpenAI, Google, Replicate, HuggingFace, Novita

**Features**:
- Embed examples for semantic matching
- Select most relevant examples per conversation
- Embed skills for code generation context
- Fallback to word-overlap if unsupported

### 9. Profile System

**Implementation**: `./andy.json`, `./profiles/*.json`

Bot profiles define identity, model choice, and behavior.

**Profile Contents**:
```json
{
  "name": "Assistant",
  "model": "gpt-4o-mini",
  "code_model": "gpt-4",
  "vision_model": "gpt-4o",
  "embedding": {"api": "openai"},
  "speak_model": {"api": "openai", "model": "tts-1", "voice": "echo"},
  "system_prompt": "You are helpful...",
  "examples": [...],
  "npc": {...},
  "skin": {...}
}
```

### 10. Docker Support

Build and run in containerized environment.

**Features**:
- Dockerfile provided for sandboxing
- Docker Compose for easy orchestration
- Maps ports 3000-3003 for browser viewers
- Support for local server via host.docker.internal

### 11. ViaProxy Support

**Implementation**: `/services/viaproxy/README.md`

Connect to unsupported Minecraft versions.

**Features**:
- Proxy for older/newer Minecraft versions
- Bridges gap between client and server
- Easy setup instructions included

---

## Architecture Overview

### Core Components

```
mindcraft/
├── src/agent/              # Agent logic
│   ├── agent.js           # Main agent class
│   ├── commands/          # 55 commands (41 actions + 14 queries)
│   ├── library/           # 60+ skills and world functions
│   ├── modes.js           # 10 automatic behavior modes
│   ├── self_prompter.js   # Autonomous goal pursuit
│   ├── conversation.js    # Multi-agent communication
│   ├── history.js         # Memory management
│   ├── npc/               # NPC controller
│   ├── vision/            # Vision/screenshot system
│   ├── tasks/             # Task system
│   ├── coder.js           # Code generation
│   └── speak.js           # Text-to-speech
├── src/models/            # 20+ LLM providers
├── src/mindcraft/         # Web server & UI
├── src/process/           # Agent process management
├── src/utils/             # Utilities (translation, data, etc)
└── settings.js            # Global configuration
```

### Data Flow

1. **User Input** → Chat message in Minecraft or API
2. **Conversation Manager** → Route to appropriate agent
3. **History** → Add to context with memory
4. **Prompter** → Send to LLM with examples and context
5. **LLM Response** → Parse for commands
6. **Command Parser** → Validate syntax and types
7. **Command Executor** → Run command/action
8. **Skills** → Execute low-level Minecraft actions
9. **Bot Response** → Output to chat or save to memory

---

## Key Features Summary

| Feature | Status | Use Case |
|---------|--------|----------|
| 55 Commands | Implemented | World interaction |
| 10 Modes | Implemented | Autonomous behavior |
| Code Generation | Optional | Complex custom tasks |
| Vision | Optional | Visual understanding |
| Multi-agent | Implemented | Cooperation |
| Tasks | Implemented | Evaluation |
| 20+ Providers | Implemented | Model flexibility |
| Memory | Implemented | Learning |
| Translation | Implemented | Multilingual |
| TTS | Implemented | Audio output |
| Docker | Implemented | Sandboxing |
| UI/WebServer | Implemented | Monitoring |

---

## Performance & Optimization

### Configurable Parameters

- **Context Windows**: Adjust `max_messages` for context size
- **Example Selection**: Control with `num_examples`
- **Skill Docs**: Limit with `relevant_docs_count`
- **Timeouts**: Set `code_timeout_mins` and `spawn_timeout`
- **Delays**: `block_place_delay` for anti-cheat compatibility

### Resource Consideration

- Vision requires additional API calls
- Code generation requires linting
- Memory summaries compress automatically
- Browser viewers are optional
- Multi-agent increases memory usage

---

## Security Considerations

- **Code execution**: Disabled by default, sandboxed when enabled
- **Vision models**: Optional, requires explicit API
- **API keys**: Store in `keys.json` (not in git)
- **Docker**: Recommended for code execution on untrusted servers
- **Blocking**: Disable dangerous commands with `blocked_actions`

---

## Getting Started Checklist

- [ ] Install Node.js 18+
- [ ] Install Minecraft Java Edition (v1.21.6 recommended)
- [ ] Clone repository
- [ ] Rename `keys.example.json` to `keys.json` with API keys
- [ ] Run `npm install`
- [ ] Configure `settings.js` for your setup
- [ ] Create/modify profile JSON
- [ ] Start Minecraft world (LAN on port 55916)
- [ ] Run `node main.js`
- [ ] Open UI at `http://localhost:8080`

---

## Extending Mindcraft

### Adding New Commands

1. Add to `/src/agent/commands/actions.js` or `queries.js`
2. Define: `name`, `description`, `params`, `perform` function
3. Use existing skills or write new ones
4. Commands auto-documented in `!help`

### Adding New Skills

1. Add to `/src/agent/library/skills.js` (for general) or create new file
2. Export async function with descriptive name
3. Use existing world functions
4. Automatically available to `!newAction` code

### Adding New Modes

1. Add to `/src/agent/modes.js`
2. Define: `name`, `description`, `on`, `active`, `update` function
3. Manage with `!setMode`
4. Modes shown in `!modes`

### Custom Providers

Add to `/src/models/` directory with required methods for compatibility.

---

**Document generated for Mindcraft v2025**
