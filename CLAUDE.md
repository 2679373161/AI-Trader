# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-Trader is a competitive autonomous trading platform where multiple AI models (Claude, GPT, DeepSeek, Qwen, Gemini) compete to trade NASDAQ 100 stocks. Each AI starts with $10,000 and makes independent trading decisions using only MCP tool calls, with zero human intervention during execution.

**Key Innovation**: Historical replay system with anti-look-ahead controls that prevents AIs from accessing future information, ensuring fair and scientifically rigorous competition.

## Common Development Commands

### Setup and Installation
```bash
# Install dependencies
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env
# Edit .env and add your API keys: OPENAI_API_KEY, ALPHAADVANTAGE_API_KEY, JINA_API_KEY
```

### Running Trading Experiments

**One-click setup and run:**
```bash
./main.sh
```

**Step-by-step manual run:**
```bash
# Step 1: Prepare data
cd data
python get_daily_price.py      # Fetch NASDAQ 100 stock data
python merge_jsonl.py          # Merge into unified format
cd ..

# Step 2: Start MCP services
cd agent_tools
python start_mcp_services.py   # Starts 4 HTTP services
cd ..

# Step 3: Run trading agent
python main.py                              # Uses default config
python main.py configs/custom_config.json   # Uses custom config

# Optional: Start web dashboard
cd docs
python3 -m http.server 8000
```

### Agent-Specific Running Scripts
```bash
./main_step1.sh  # Data preparation only
./main_step2.sh  # Start MCP services only
./main_step3.sh  # Run trading only
```

## Code Architecture

### Core Components

#### 1. Main Entry Point (`main.py`)
- Orchestrates multi-model trading experiments
- Dynamically loads agent classes via `AGENT_REGISTRY`
- Supports `BaseAgent` (daily) and `BaseAgent_Hour` (hourly) modes
- Processes models sequentially to ensure fair comparison

#### 2. Agent System (`agent/base_agent/`)
- **base_agent.py**: Core BaseAgent class with:
  - MCP tool connection via `langchain_mcp_adapters.client.MultiServerMCPClient`
  - AI agent creation using `langchain.agents.create_agent`
  - Date range iteration with historical replay controls
  - Position tracking and logging
- **base_agent_hour.py**: Extended agent for hourly trading

**Agent Execution Flow:**
1. Initialize MCP tools and AI model connection
2. Iterate through trading dates
3. Build system prompt with current position/price data
4. Execute reasoning loop with tool calls
5. Log all trades and decisions
6. Output final position summary

#### 3. MCP Toolchain (`agent_tools/`)
Four independent HTTP services (started via `start_mcp_services.py`):

| Service | Port | File | Purpose |
|---------|------|------|---------|
| Math | 8000 | tool_math.py | Financial calculations |
| Search | 8001 | tool_jina_search.py | Market news/search via Jina AI |
| Trade | 8002 | tool_trade.py | Execute buy/sell orders |
| Price | 8003 | tool_get_price_local.py | Query historical/current prices |

Each tool runs as a separate process and is accessed by agents via HTTP requests.

#### 4. Data System (`data/`)
- **Stock Price Data**: `daily_prices_*.json` files for each NASDAQ 100 stock
- **Unified Format**: `merged.jsonl` - consolidated data in JSONL format
- **Trading Records**: `data/agent_data/{signature}/` - per-agent trading logs
  - `position/position.jsonl` - position history
  - `log/{date}/log.jsonl` - daily trading logs

**Data Flow:**
1. `get_daily_price.py` fetches from Alpha Vantage API
2. `merge_jsonl.py` converts to unified JSONL format
3. Price tools query local data files (never external APIs during trading)
4. Anti-look-ahead: Agents can only access data up to current simulation date

#### 5. Configuration System
- **Default Config**: `configs/default_config.json`
  - Trading date range
  - Model list with basemodel/signature
  - Agent parameters (max_steps, max_retries, base_delay, initial_cash)
- **Runtime Environment**: `data/agent_data/{signature}/.runtime_env.json`
  - `TODAY_DATE`: Current simulation date
  - `IF_TRADE`: Whether trading occurred
  - `SIGNATURE`: Agent identifier

#### 6. Prompt System (`prompts/agent_prompt.py`)
- `agent_system_prompt`: Template for agent instructions
- `get_agent_system_prompt()`: Generates prompts with current market data
- `STOP_SIGNAL`: `<FINISH_SIGNAL>` - agent terminates when output

### Important Files

| File | Purpose |
|------|---------|
| `main.py` | Experiment orchestration |
| `main_parrallel.py` | Parallel execution variant |
| `agent/base_agent/base_agent.py` | Core agent logic |
| `agent_tools/start_mcp_services.py` | Service manager |
| `tools/price_tools.py` | Price data queries |
| `tools/general_tools.py` | Configuration and utilities |
| `tools/result_tools.py` | Performance analysis |
| `prompts/agent_prompt.py` | Agent prompts |
| `docs/index.html` | Web dashboard |

## Environment Configuration

**Required Environment Variables:**
```bash
# API Keys (from .env file)
OPENAI_API_KEY=your_key
ALPHAADVANTAGE_API_KEY=your_key
JINA_API_KEY=your_key

# Optional: Custom OpenAI base URL
OPENAI_API_BASE=https://your-proxy.com/v1

# Service Ports (defaults shown)
MATH_HTTP_PORT=8000
SEARCH_HTTP_PORT=8001
TRADE_HTTP_PORT=8002
GETPRICE_HTTP_PORT=8003

# Agent Limits
AGENT_MAX_STEP=30
```

## Supported Agent Types

Currently registered in `AGENT_REGISTRY`:
1. **BaseAgent**: Daily trading intervals
2. **BaseAgent_Hour**: Hourly trading intervals

To add a new agent type:
1. Create `agent/{type}/{type}.py` with your agent class
2. Register in `AGENT_REGISTRY` dict in `main.py`
3. Set `"agent_type": "YourType"` in config

## Data Access Patterns

### Reading Position Data
Agents receive current positions in prompt format:
```json
{
  "AAPL": 10,
  "MSFT": 5,
  "CASH": 9737.60
}
```

### Querying Prices
- Use `tool_get_price_local.py` service
- Reads from local JSON files only (never live APIs)
- Enforces temporal boundaries (no future data)

### Executing Trades
- Use `tool_trade.py` service
- Records to `position.jsonl`
- Updates agent's position state

### Market Intelligence
- Use `tool_jina_search.py` service
- Searches market news via Jina AI
- Results filtered by date constraints

## Common Patterns and Utilities

### Configuration Management
```python
from tools.general_tools import get_config_value, write_config_value

# Read/write runtime config
today_date = get_config_value("TODAY_DATE")
write_config_value("IF_TRADE", True)
```

### Price Data Access
```python
from tools.price_tools import get_open_prices, get_yesterday_open_and_close_price

# Get today's opening prices
today_prices = get_open_prices(date, symbols)

# Get yesterday's data
yesterday_buy, yesterday_sell = get_yesterday_open_and_close_price(date, symbols)
```

### Adding No-Trade Records
```python
from tools.price_tools import add_no_trade_record

# When agent decides not to trade
add_no_trade_record(date, signature)
```

## Competition and Performance

Each AI model competes under identical conditions:
- **Starting Capital**: $10,000
- **Trading Universe**: NASDAQ 100 stocks
- **Data Sources**: Same historical data for all
- **Tools**: Identical MCP toolchain
- **Evaluation**: Returns, Sharpe ratio, max drawdown (see `tools/result_tools.py`)

## Key Constraints

1. **Zero Human Intervention**: Agents must use only tool calls, no manual operations
2. **Anti-Look-Ahead**: Temporal data filters prevent future information access
3. **Fair Competition**: Sequential execution prevents race conditions
4. **Tool-Only Execution**: All operations must use MCP tools

## Development Tips

### Debugging Agent Behavior
- Check `data/agent_data/{signature}/log/{date}/log.jsonl` for reasoning traces
- Enable verbose logging in agent config
- Monitor MCP service logs in `logs/` directory

### Custom Agent Development
1. Inherit from `BaseAgent`
2. Override `run_date_range()` if needed
3. Register in `AGENT_REGISTRY`
4. Update prompts in `prompts/agent_prompt.py`

### Testing Changes
```bash
# Quick test with single model
python main.py configs/default_config.json

# Verify data flow
python data/get_daily_price.py
python data/merge_jsonl.py

# Test MCP services
cd agent_tools && python start_mcp_services.py
```

### Performance Analysis
```bash
# View trading results
cat data/agent_data/{signature}/position/position.jsonl

# Calculate metrics (requires implementation)
python tools/result_tools.py
```

## Community Resources

- **Discussions**: [GitHub Discussions](https://github.com/HKUDS/AI-Trader/discussions)
- **Issues**: [GitHub Issues](https://github.com/HKUDS/AI-Trader/issues)
- **Communication**: See Communication.md for WeChat/Feishu QR codes

## Disclaimer

This is a research project for educational purposes. All trading is simulated on historical data. Do not use for actual trading decisions without professional financial advice.
