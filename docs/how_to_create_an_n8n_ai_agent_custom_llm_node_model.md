# How to Create an n8n AI Agent Custom LLM Node Model (Deterministic, End-to-End)

This guide teaches you how to build, deploy, and validate a **custom AI chat model node** for n8n that behaves like native chat model nodes in AI Agent executions.

Validated on **February 21, 2026** with a real end-to-end run:
- Script result: `SUCCESS`
- Created workflow ID: `jazp5jAQZQbi8tcC`
- Execution ID: `687`

## What This Solves

If your custom model works but does not show as an executed model node in n8n execution logs, the root issue is usually missing LLM tracing integration.

Native model nodes (for example, Google Gemini Chat Model) emit language-model run data in the format n8n expects. A plain custom `SimpleChatModel` implementation does not do this automatically.

## Why Gemini Shows Up But a Custom Model Does Not

Gemini and other first-party model nodes are wired to n8n's AI tracing path. They create AI language-model run records that populate:

- `runData["<Model Node Name>"]`
- `executionData.metadata["<Model Node Name>"]` with sub-run details

A custom model must explicitly do the same by attaching a callback handler that:

1. Calls `addInputData(NodeConnectionTypes.AiLanguageModel, ...)` on LLM start.
2. Calls `addOutputData(NodeConnectionTypes.AiLanguageModel, ...)` on LLM end.
3. Emits a proper error via `addOutputData(..., NodeOperationError)` on LLM failure.

Without this, the AI Agent can still return text, but n8n won't render model execution traces like Gemini.

## Deterministic Script (Tested)

The script below is deterministic and fail-fast:

- scaffolds a complete `n8n-nodes-claudecode-model` package,
- includes tracing integration (`N8nLlmTracing.ts`),
- builds and packs,
- deploys to server community nodes directory,
- restarts n8n,
- waits for authenticated API readiness,
- creates a test workflow via public API,
- runs it via internal REST manual-run endpoint,
- verifies trace presence in execution data.

### Required Environment Variables

Set all of these before running:

- `PROJECT_ROOT`
- `N8N_BASE` (must end with `/api/v1`)
- `N8N_API_KEY`
- `N8N_EMAIL`
- `N8N_PASSWORD`
- `N8N_SSH_HOST`
- `N8N_SERVER_NODE_DIR`
- `N8N_PLIST_PATH`
- `CLAUDE_CREDENTIAL_ID`
- `CLAUDE_CREDENTIAL_NAME`

### Run

```bash
chmod +x ./create_custom_n8n_ai_model_test.sh
./create_custom_n8n_ai_model_test.sh
```

### Script

```bash
#!/usr/bin/env bash
set -euo pipefail

required_env_vars=(
  PROJECT_ROOT
  N8N_BASE
  N8N_API_KEY
  N8N_EMAIL
  N8N_PASSWORD
  N8N_SSH_HOST
  N8N_SERVER_NODE_DIR
  N8N_PLIST_PATH
  CLAUDE_CREDENTIAL_ID
  CLAUDE_CREDENTIAL_NAME
)

for var in "${required_env_vars[@]}"; do
  if [[ -z "${!var:-}" ]]; then
    echo "Missing required env var: $var" >&2
    exit 1
  fi
done

required_bins=(npm jq curl ssh scp)
for bin in "${required_bins[@]}"; do
  if ! command -v "$bin" >/dev/null 2>&1; then
    echo "Missing required binary: $bin" >&2
    exit 1
  fi
done

if [[ "$N8N_BASE" != */api/v1 ]]; then
  echo "N8N_BASE must end with /api/v1. Got: $N8N_BASE" >&2
  exit 1
fi

REST_BASE="${N8N_BASE%/api/v1}"

if [[ -e "$PROJECT_ROOT/package.json" ]]; then
  echo "Refusing to overwrite existing package at $PROJECT_ROOT" >&2
  exit 1
fi

mkdir -p "$PROJECT_ROOT/nodes/LmChatClaudeCode"

cat >"$PROJECT_ROOT/package.json" <<'JSON'
{
  "name": "n8n-nodes-claudecode-model",
  "version": "0.1.4",
  "description": "n8n language model sub-node for Claude Code CLI",
  "license": "MIT",
  "keywords": [
    "n8n-community-node-package"
  ],
  "author": {
    "name": "Adam Bertram"
  },
  "n8n": {
    "n8nNodesApiVersion": 1,
    "strict": false,
    "nodes": [
      "dist/nodes/LmChatClaudeCode/LmChatClaudeCode.node.js"
    ],
    "credentials": []
  },
  "scripts": {
    "build": "n8n-node build",
    "build:watch": "tsc --watch",
    "dev": "n8n-node dev",
    "lint": "n8n-node lint",
    "lint:fix": "n8n-node lint --fix",
    "release": "n8n-node release",
    "prepublishOnly": "n8n-node prerelease"
  },
  "files": [
    "dist"
  ],
  "peerDependencies": {
    "n8n-workflow": "*",
    "@langchain/core": ">=0.3.0"
  },
  "devDependencies": {
    "@langchain/core": "0.3.44",
    "@n8n/node-cli": "*",
    "@types/node": "^25.2.3",
    "eslint": "9.32.0",
    "prettier": "3.6.2",
    "typescript": "5.9.2"
  }
}
JSON

cat >"$PROJECT_ROOT/tsconfig.json" <<'JSON'
{
  "compilerOptions": {
    "strict": false,
    "module": "commonjs",
    "target": "es2019",
    "lib": ["es2019"],
    "declaration": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": ".",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": [
    "nodes/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
JSON

cat >"$PROJECT_ROOT/nodes/LmChatClaudeCode/LmChatClaudeCode.node.ts" <<'TS'
import {
	NodeConnectionTypes,
	type INodeType,
	type INodeTypeDescription,
	type ISupplyDataFunctions,
	type SupplyData,
} from 'n8n-workflow';

import { ChatClaudeCode } from './ChatClaudeCode';
import { N8nLlmTracing } from './N8nLlmTracing';

export class LmChatClaudeCode implements INodeType {
	description: INodeTypeDescription = {
		displayName: 'Claude Code Chat Model',
		name: 'lmChatClaudeCode',
		icon: 'file:claudecode.svg',
		group: ['transform'],
		version: 1,
		description: 'Language model using Claude Code CLI with full agentic capabilities',
		defaults: {
			name: 'Claude Code Chat Model',
		},
		codex: {
			categories: ['AI'],
			subcategories: {
				AI: ['Language Models', 'Root Nodes'],
				'Language Models': ['Chat Models (Recommended)'],
			},
			resources: {
				primaryDocumentation: [
					{
						url: 'https://docs.anthropic.com/en/docs/claude-code',
					},
				],
			},
		},
		usableAsTool: true,
		inputs: [],
		outputs: [NodeConnectionTypes.AiLanguageModel],
		outputNames: ['Model'],
		credentials: [
			{
				name: 'claudeCodeApi',
				required: true,
			},
		],
		properties: [
			{
				displayName: 'Model',
				name: 'model',
				type: 'options',
				default: 'sonnet',
				options: [
					{ name: 'Haiku', value: 'haiku' },
					{ name: 'Sonnet', value: 'sonnet' },
					{ name: 'Opus', value: 'opus' },
					{ name: 'Custom', value: 'custom' },
				],
				description: 'The Claude model to use',
			},
			{
				displayName: 'Custom Model',
				name: 'customModel',
				type: 'string',
				default: '',
				displayOptions: {
					show: {
						model: ['custom'],
					},
				},
				description: 'Custom model identifier (e.g. claude-sonnet-4-20250514)',
			},
			{
				displayName: 'Max Turns',
				name: 'maxTurns',
				type: 'number',
				default: 100,
				description: 'Maximum number of agentic turns before stopping (safety cap)',
			},
			{
				displayName: 'Permission Mode',
				name: 'permissionMode',
				type: 'options',
				default: 'bypassPermissions',
				options: [
					{ name: 'Default', value: 'default' },
					{ name: 'Accept Edits', value: 'acceptEdits' },
					{ name: 'Bypass Permissions', value: 'bypassPermissions' },
					{ name: "Don't Ask", value: 'dontAsk' },
				],
				description: 'Claude Code permission mode for tool execution',
			},
			{
				displayName: 'Timeout (Seconds)',
				name: 'timeout',
				type: 'number',
				default: 300,
				description: 'Maximum time in seconds to wait for Claude Code to respond',
			},
			{
				displayName: 'No Session Persistence',
				name: 'noSessionPersistence',
				type: 'boolean',
				default: true,
				description: 'Whether to disable session persistence (prevents disk bloat from automated calls)',
			},
			{
				displayName: 'Options',
				name: 'options',
				type: 'collection',
				placeholder: 'Add Option',
				default: {},
				options: [
					{
						displayName: 'Allowed Tools',
						name: 'allowedTools',
						type: 'string',
						default: '',
						description: 'Comma-separated list of allowed Claude Code tools (e.g. Read,Edit,Bash,Glob,Grep)',
					},
					{
						displayName: 'Append System Prompt',
						name: 'appendSystemPrompt',
						type: 'string',
						typeOptions: { rows: 4 },
						default: '',
						description: 'Additional system prompt text appended after the n8n execution context',
					},
					{
						displayName: 'Max Budget (USD)',
						name: 'maxBudgetUsd',
						type: 'number',
						default: 0,
						description: 'Maximum budget in USD for this call. 0 = no limit.',
					},
					{
						displayName: 'Working Directory',
						name: 'workingDirectory',
						type: 'string',
						default: '',
						description: 'Working directory for Claude Code execution',
					},
				],
			},
		],
	};

	async supplyData(this: ISupplyDataFunctions, itemIndex: number): Promise<SupplyData> {
		const credentials = await this.getCredentials('claudeCodeApi');
		const claudePath = (credentials.claudePath as string) || '/Users/adam/.local/bin/claude';
		const runAsUser = (credentials.runAsUser as string) || '';
		const apiKey = (credentials.apiKey as string) || '';

		const modelParam = this.getNodeParameter('model', itemIndex) as string;
		const customModel = modelParam === 'custom'
			? (this.getNodeParameter('customModel', itemIndex) as string)
			: '';
		const effectiveModel = modelParam === 'custom' ? customModel : modelParam;

		const maxTurns = this.getNodeParameter('maxTurns', itemIndex, 10) as number;
		const permissionMode = this.getNodeParameter('permissionMode', itemIndex, 'bypassPermissions') as string;
		const timeout = this.getNodeParameter('timeout', itemIndex, 300) as number;
		const noSessionPersistence = this.getNodeParameter('noSessionPersistence', itemIndex, true) as boolean;

		const options = this.getNodeParameter('options', itemIndex, {}) as Record<string, string | number | boolean>;

		const model = new ChatClaudeCode({
			claudePath,
			runAsUser,
			apiKey,
			model: effectiveModel,
			appendSystemPrompt: (options.appendSystemPrompt as string) || '',
			maxTurns,
			maxBudgetUsd: (options.maxBudgetUsd as number) || 0,
			permissionMode,
			allowedTools: (options.allowedTools as string) || '',
			workingDirectory: (options.workingDirectory as string) || '',
			timeoutSeconds: timeout,
			noSessionPersistence,
			callbacks: [new N8nLlmTracing(this)],
		});

		return {
			response: model,
		};
	}
}
TS

cat >"$PROJECT_ROOT/nodes/LmChatClaudeCode/ChatClaudeCode.ts" <<'TS'
import { exec } from 'child_process';
import { SimpleChatModel } from '@langchain/core/language_models/chat_models';
import type { BaseMessage } from '@langchain/core/messages';
import type { CallbackManagerForLLMRun, Callbacks } from '@langchain/core/callbacks/manager';
import type { BindToolsInput } from '@langchain/core/language_models/chat_models';

export interface ChatClaudeCodeInput {
	claudePath: string;
	runAsUser: string;
	apiKey: string;
	model: string;
	appendSystemPrompt: string;
	maxTurns: number;
	maxBudgetUsd: number;
	permissionMode: string;
	allowedTools: string;
	workingDirectory: string;
	timeoutSeconds: number;
	noSessionPersistence: boolean;
	callbacks?: Callbacks;
}

interface ExecError extends Error {
	killed?: boolean;
	code?: number;
}

export class ChatClaudeCode extends SimpleChatModel {
	claudePath: string;
	runAsUser: string;
	apiKey: string;
	model: string;
	appendSystemPrompt: string;
	maxTurns: number;
	maxBudgetUsd: number;
	permissionMode: string;
	allowedTools: string;
	workingDirectory: string;
	timeoutSeconds: number;
	noSessionPersistence: boolean;

	private boundTools: BindToolsInput[] = [];

	constructor(fields: ChatClaudeCodeInput) {
		super({ callbacks: fields.callbacks });
		this.claudePath = fields.claudePath;
		this.runAsUser = fields.runAsUser;
		this.apiKey = fields.apiKey;
		this.model = fields.model;
		this.appendSystemPrompt = fields.appendSystemPrompt;
		this.maxTurns = fields.maxTurns;
		this.maxBudgetUsd = fields.maxBudgetUsd;
		this.permissionMode = fields.permissionMode;
		this.allowedTools = fields.allowedTools;
		this.workingDirectory = fields.workingDirectory;
		this.timeoutSeconds = fields.timeoutSeconds;
		this.noSessionPersistence = fields.noSessionPersistence;
	}

	_llmType(): string {
		return 'claude-code';
	}

	bindTools(tools: BindToolsInput[]): this {
		this.boundTools = tools;
		return this;
	}

	async _call(
		messages: BaseMessage[],
		_options: this['ParsedCallOptions'],
		_runManager?: CallbackManagerForLLMRun,
	): Promise<string> {
		void _options;
		void _runManager;

		let prompt = '';
		let systemContent = '';

		for (const msg of messages) {
			const type = msg._getType();
			const content = typeof msg.content === 'string'
				? msg.content
				: JSON.stringify(msg.content);

			if (type === 'system') {
				systemContent += (systemContent ? '\n\n' : '') + content;
			} else if (type === 'human') {
				prompt = content;
			}
		}

		if (!prompt) {
			return 'No prompt provided.';
		}

		const args: string[] = ['-p'];

		if (this.model) {
			args.push('--model', this.model);
		}

		args.push('--output-format', 'json');

		const n8nContext = [
			'You are running inside an n8n workflow, invoked by another AI agent.',
			'Return your complete response as plain text output.'
		].join('\n');

		const systemParts = [n8nContext];
		if (this.appendSystemPrompt) {
			systemParts.push(this.appendSystemPrompt);
		}
		if (systemContent) {
			systemParts.push(systemContent);
		}
		args.push('--append-system-prompt', systemParts.join('\n\n'));

		if (this.maxTurns > 0) {
			args.push('--max-turns', String(this.maxTurns));
		}
		if (this.maxBudgetUsd > 0) {
			args.push('--max-budget-usd', String(this.maxBudgetUsd));
		}
		if (this.permissionMode && this.permissionMode !== 'default') {
			args.push('--permission-mode', this.permissionMode);
		}
		if (this.allowedTools) {
			args.push('--allowedTools', this.allowedTools);
		}
		if (this.noSessionPersistence) {
			args.push('--no-session-persistence');
		}

		args.push(prompt);

		const escapedArgs = args.map((arg) => {
			const escaped = arg.replace(/'/g, "'\\''");
			return `'${escaped}'`;
		});
		const claudeCmd = `${this.claudePath} ${escapedArgs.join(' ')}`;

		let fullCommand: string;
		if (this.runAsUser) {
			const escapedCmd = claudeCmd.replace(/'/g, "'\\''");
			fullCommand = `su - ${this.runAsUser} -c '${escapedCmd}'`;
		} else {
			fullCommand = claudeCmd;
		}
		fullCommand += ' </dev/null';

		const env = { ...process.env };
		if (this.apiKey) {
			env.ANTHROPIC_API_KEY = this.apiKey;
		}

		const timeoutMs = this.timeoutSeconds * 1000;
		const cwd = this.workingDirectory || undefined;

		const output = await new Promise<string>((resolve, reject) => {
			const child = exec(
				fullCommand,
				{
					timeout: timeoutMs,
					maxBuffer: 50 * 1024 * 1024,
					killSignal: 'SIGKILL',
					env,
					cwd,
				},
				(error, stdout, stderr) => {
					if (error) {
						const execErr = error as ExecError;
						const detail = stderr || stdout.trim() || error.message;
						let errMsg: string;
						if (execErr.killed) {
							errMsg = `Process killed after ${this.timeoutSeconds}s timeout. Output: ${detail}`;
						} else if (execErr.code) {
							errMsg = `Exit code ${execErr.code}: ${detail}`;
						} else {
							errMsg = detail;
						}
						reject(new Error(`Claude Code failed: ${errMsg}`));
					} else {
						resolve(stdout);
					}
				},
			);

			const killTimer = setTimeout(() => {
				if (child.pid && !child.killed) {
					try {
						process.kill(-child.pid, 'SIGKILL');
					} catch {
					}
				}
			}, timeoutMs + 10000);
			child.on('exit', () => clearTimeout(killTimer));
		});

		try {
			const parsed = JSON.parse(output);
			if (parsed.result) {
				return parsed.result;
			}
			if (parsed.subtype === 'error_max_turns') {
				return `[Claude Code hit max turns (${parsed.num_turns || 'unknown'}).`;
			}
			if (parsed.is_error || (parsed.errors && parsed.errors.length > 0)) {
				const errMsgs = (parsed.errors || []).map((e: any) => e.message || String(e)).join('; ');
				return `[Claude Code error: ${errMsgs || parsed.subtype || 'unknown'}]`;
			}
			return parsed.text || parsed.result || `[Claude Code returned no result text.]`;
		} catch {
			return output.trim();
		}
	}
}
TS

cat >"$PROJECT_ROOT/nodes/LmChatClaudeCode/N8nLlmTracing.ts" <<'TS'
import { BaseCallbackHandler } from '@langchain/core/callbacks/base';
import type { Serialized } from '@langchain/core/load/serializable';
import type { LLMResult } from '@langchain/core/outputs';
import {
	NodeConnectionTypes,
	NodeOperationError,
	type ISupplyDataFunctions,
} from 'n8n-workflow';

interface TokenUsage {
	completionTokens: number;
	promptTokens: number;
	totalTokens: number;
}

interface TracingRunDetails {
	index: number;
	messages: string[];
	options: unknown;
	promptTokensEstimate: number;
}

export class N8nLlmTracing extends BaseCallbackHandler {
	name = 'N8nLlmTracing';
	awaitHandlers = true;

	private readonly executionFunctions: ISupplyDataFunctions;
	private readonly runsMap: Record<string, TracingRunDetails> = {};
	private parentRunIndex: number | undefined;

	constructor(executionFunctions: ISupplyDataFunctions) {
		super();
		this.executionFunctions = executionFunctions;
	}

	setParentRunIndex(runIndex: number): void {
		this.parentRunIndex = runIndex;
	}

	async handleLLMStart(llm: Serialized, prompts: string[], runId: string): Promise<void> {
		const sourceNodeRunIndex = this.parentRunIndex !== undefined
			? this.parentRunIndex + this.executionFunctions.getNextRunIndex()
			: undefined;

		const llmDetails = llm as unknown as { type?: string; kwargs?: Record<string, unknown> };
		const options = (
			llmDetails.type === 'constructor' ? llmDetails.kwargs : llm
		) as Record<string, unknown>;

		const promptTokensEstimate = this.estimateTokensFromStrings(prompts);

		const { index } = this.executionFunctions.addInputData(
			NodeConnectionTypes.AiLanguageModel,
			[
				[
					{
						json: {
							messages: prompts,
							estimatedTokens: promptTokensEstimate,
							options,
						},
					},
				],
			],
			sourceNodeRunIndex,
		);

		this.runsMap[runId] = {
			index,
			messages: prompts,
			options,
			promptTokensEstimate,
		};
	}

	async handleLLMEnd(output: LLMResult, runId: string): Promise<void> {
		const runDetails = this.runsMap[runId] ?? {
			index: Object.keys(this.runsMap).length,
			messages: [],
			options: {},
			promptTokensEstimate: 0,
		};

		const generations = (output.generations ?? []).map((generationGroup) =>
			generationGroup.map((generation) => ({
				text: generation.text,
				generationInfo: generation.generationInfo,
			})),
		);

		const completionTokensEstimate = this.estimateTokensFromStrings(
			generations.flatMap((group) => group.map((generation) => generation.text ?? '')),
		);

		const tokenUsage = this.parseTokenUsage(output);

		const response: {
			response: { generations: Array<Array<{ text: string; generationInfo: unknown }>> };
			tokenUsage?: TokenUsage;
			tokenUsageEstimate?: TokenUsage;
		} = {
			response: { generations },
		};

		if (tokenUsage.totalTokens > 0) {
			response.tokenUsage = tokenUsage;
		} else {
			response.tokenUsageEstimate = {
				promptTokens: runDetails.promptTokensEstimate,
				completionTokens: completionTokensEstimate,
				totalTokens: runDetails.promptTokensEstimate + completionTokensEstimate,
			};
		}

		const sourceNodeRunIndex = this.parentRunIndex !== undefined
			? this.parentRunIndex + runDetails.index
			: undefined;

		this.executionFunctions.addOutputData(
			NodeConnectionTypes.AiLanguageModel,
			runDetails.index,
			[[{ json: response }]],
			undefined,
			sourceNodeRunIndex,
		);
	}

	async handleLLMError(error: unknown, runId: string): Promise<void> {
		const runDetails = this.runsMap[runId] ?? { index: Object.keys(this.runsMap).length };
		const message = error instanceof Error ? error.message : String(error);
		const nodeError = new NodeOperationError(this.executionFunctions.getNode(), message, {
			functionality: 'configuration-node',
		});
		this.executionFunctions.addOutputData(
			NodeConnectionTypes.AiLanguageModel,
			runDetails.index,
			nodeError,
		);
	}

	private parseTokenUsage(output: LLMResult): TokenUsage {
		const llmOutput = output.llmOutput as
			| { tokenUsage?: { completionTokens?: number; promptTokens?: number } }
			| undefined;

		const completionTokens = llmOutput?.tokenUsage?.completionTokens ?? 0;
		const promptTokens = llmOutput?.tokenUsage?.promptTokens ?? 0;

		return {
			completionTokens,
			promptTokens,
			totalTokens: completionTokens + promptTokens,
		};
	}

	private estimateTokensFromStrings(values: string[]): number {
		const text = values.join(' ').trim();
		if (text.length === 0) {
			return 0;
		}
		return text.split(/\s+/).length;
	}
}
TS

cat >"$PROJECT_ROOT/nodes/LmChatClaudeCode/claudecode.svg" <<'SVG'
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect x="8" y="8" width="48" height="48" rx="8" fill="#111827"/>
  <path d="M20 34c0-8 6-14 14-14h10v6H34c-4 0-8 4-8 8s4 8 8 8h10v6H34c-8 0-14-6-14-14z" fill="#fff"/>
</svg>
SVG

cd "$PROJECT_ROOT"
npm install
npm run build
tarball="$(npm pack | tail -n 1)"

scp "$tarball" "${N8N_SSH_HOST}:/tmp/${tarball}"
ssh "$N8N_SSH_HOST" "cd ${N8N_SERVER_NODE_DIR} && /usr/local/bin/npm install /tmp/${tarball}"
ssh "$N8N_SSH_HOST" "sudo launchctl unload ${N8N_PLIST_PATH} && sudo launchctl load ${N8N_PLIST_PATH}"

api_ready=0
for _ in $(seq 1 120); do
  if curl -fsS "${N8N_BASE}/workflows?limit=1" \
    -H "X-N8N-API-KEY: ${N8N_API_KEY}" 2>/dev/null \
    | jq -e '.data | type == "array"' >/dev/null 2>&1; then
    api_ready=1
    break
  fi
  sleep 1
done

if [[ "$api_ready" -ne 1 ]]; then
  echo "n8n API readiness check failed after restart: ${N8N_BASE}/workflows?limit=1" >&2
  exit 1
fi

workflow_body="$(
  jq -n \
    --arg credId "$CLAUDE_CREDENTIAL_ID" \
    --arg credName "$CLAUDE_CREDENTIAL_NAME" \
    '{
      name: "Test: Claude Code Custom Model Script Validation",
      nodes: [
        {
          parameters: {},
          id: "11111111-1111-1111-1111-111111111111",
          name: "Manual Trigger",
          type: "n8n-nodes-base.manualTrigger",
          typeVersion: 1,
          position: [320, 304]
        },
        {
          parameters: {
            promptType: "define",
            text: "Reply with exactly: trace test ok",
            options: {}
          },
          id: "22222222-2222-2222-2222-222222222222",
          name: "AI Agent",
          type: "@n8n/n8n-nodes-langchain.agent",
          typeVersion: 3.1,
          position: [576, 304]
        },
        {
          parameters: {
            options: {}
          },
          id: "33333333-3333-3333-3333-333333333333",
          name: "Claude Code Chat Model",
          type: "n8n-nodes-claudecode-model.lmChatClaudeCode",
          typeVersion: 1,
          position: [576, 512],
          credentials: {
            claudeCodeApi: {
              id: $credId,
              name: $credName
            }
          }
        }
      ],
      connections: {
        "Manual Trigger": {
          main: [[{ node: "AI Agent", type: "main", index: 0 }]]
        },
        "Claude Code Chat Model": {
          ai_languageModel: [[{ node: "AI Agent", type: "ai_languageModel", index: 0 }]]
        }
      },
      settings: {
        saveManualExecutions: true,
        saveDataSuccessExecution: "all",
        saveDataErrorExecution: "all",
        callerPolicy: "workflowsFromSameOwner",
        availableInMCP: false
      }
    }'
)"

create_resp="$(curl -sS "${N8N_BASE}/workflows" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  -H 'Content-Type: application/json' \
  -d "$workflow_body")"

workflow_id="$(echo "$create_resp" | jq -r '.id')"
if [[ -z "$workflow_id" || "$workflow_id" == "null" ]]; then
  echo "Workflow creation failed: $create_resp" >&2
  exit 1
fi

cookie_file="$(mktemp)"
trap 'rm -f "$cookie_file"' EXIT

login_payload="$(
  jq -n \
    --arg email "$N8N_EMAIL" \
    --arg password "$N8N_PASSWORD" \
    '{emailOrLdapLoginId:$email,password:$password}'
)"

curl -sS -c "$cookie_file" "${REST_BASE}/rest/login" \
  -H 'Content-Type: application/json' \
  -d "$login_payload" >/dev/null

workflow_data="$(curl -sS "${N8N_BASE}/workflows/${workflow_id}" \
  -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  | jq '{id,name,active,activeVersionId,nodes,connections,settings,pinData}')"

run_payload="$(jq -n --argjson wf "$workflow_data" '{workflowData:$wf,destinationNode:{nodeName:"AI Agent",mode:"inclusive"}}')"
run_resp="$(curl -sS -b "$cookie_file" -X POST "${REST_BASE}/rest/workflows/${workflow_id}/run" \
  -H 'Content-Type: application/json' \
  -d "$run_payload")"

execution_id="$(echo "$run_resp" | jq -r '.data.executionId')"
if [[ -z "$execution_id" || "$execution_id" == "null" ]]; then
  echo "Manual run failed: $run_resp" >&2
  exit 1
fi

execution_data=""
execution_status=""
for _ in $(seq 1 90); do
  execution_data="$(curl -sS "${N8N_BASE}/executions/${execution_id}?includeData=true" \
    -H "X-N8N-API-KEY: ${N8N_API_KEY}")"
  execution_status="$(echo "$execution_data" | jq -r '.status')"
  if [[ "$execution_status" == "success" || "$execution_status" == "error" || "$execution_status" == "canceled" ]]; then
    break
  fi
  sleep 1
done

if [[ "$execution_status" != "success" ]]; then
  echo "Execution did not succeed. Status: $execution_status" >&2
  echo "$execution_data" >&2
  exit 1
fi

echo "$execution_data" | jq -e '.data.resultData.runData["Claude Code Chat Model"]' >/dev/null
echo "$execution_data" | jq -e '.data.executionData.metadata["Claude Code Chat Model"][0].subRun[0].node == "Claude Code Chat Model"' >/dev/null

echo "SUCCESS"
echo "workflow_id=${workflow_id}"
echo "execution_id=${execution_id}"
```

## How the Script Proves Correctness

The script only returns success when all of the following are true:

1. Build succeeds (`npm run build`).
2. Package installs on server in community-node location.
3. n8n restart succeeds and authenticated API becomes ready.
4. Test workflow creation succeeds.
5. Manual execution reaches `success` status.
6. Execution data includes model run data:
   - `.data.resultData.runData["Claude Code Chat Model"]`
7. Execution metadata includes model sub-run:
   - `.data.executionData.metadata["Claude Code Chat Model"][0].subRun[0].node == "Claude Code Chat Model"`

If any check fails, the script exits non-zero immediately.

## Architecture Notes (Minimal but Complete)

The custom model package has three critical files:

1. `LmChatClaudeCode.node.ts`
- Node definition for n8n UI and AI Agent model output.
- In `supplyData`, it creates `ChatClaudeCode` and injects callbacks:
  - `callbacks: [new N8nLlmTracing(this)]`

2. `ChatClaudeCode.ts`
- Extends LangChain `SimpleChatModel`.
- Executes Claude Code CLI with strict command construction.
- Returns parsed text output to the AI Agent.

3. `N8nLlmTracing.ts`
- Extends LangChain `BaseCallbackHandler`.
- Bridges LangChain run lifecycle into n8n runData/metadata via `addInputData` and `addOutputData` using `NodeConnectionTypes.AiLanguageModel`.
- This is the key implementation difference vs an untraced custom model.

## n8n API and Runtime Facts You Must Respect

- Public API uses `X-N8N-API-KEY`, not Bearer.
- Public API has no manual-run endpoint for workflows.
- Manual run used here is internal REST:
  - login: `POST /rest/login`
  - run: `POST /rest/workflows/:id/run`
- For unknown-trigger full manual runs, payload needs `destinationNode`.
- Community-node package install on this setup is done in user node dir (`~/.n8n/nodes`), not in global n8n install directory.

## Operational Gotchas

1. Restart race condition
- n8n process can be up before API endpoints are fully ready.
- Gate on authenticated API readiness (`/api/v1/workflows?limit=1`) before creating/running workflows.

2. Custom model executes but not visible in logs
- Root cause: missing tracing callback integration.
- Fix: add `N8nLlmTracing` callback and emit language-model input/output data.

3. API run errors about `nodeName`
- Root cause: missing `destinationNode` in internal run payload.
- Fix: include `destinationNode: { nodeName: "AI Agent", mode: "inclusive" }`.

## Adapting This to Your Own Model Node

Use this checklist:

1. Keep package name as `n8n-nodes-*`.
2. Ensure your model node outputs `NodeConnectionTypes.AiLanguageModel`.
3. Wrap your model class with callback-based tracing to n8n run data.
4. Test with a minimal workflow:
   - Trigger -> AI Agent
   - Custom Model connected to Agent `Chat Model` input
5. Validate both:
   - response text correctness
   - runData/metadata visibility for the model node

## Fast Verification Commands (Post-Deploy)

```bash
# 1) Verify package is installed where n8n loads community nodes
ssh "$N8N_SSH_HOST" "cd $N8N_SERVER_NODE_DIR && npm ls n8n-nodes-claudecode-model"

# 2) Verify tracing artifact exists in built output
ssh "$N8N_SSH_HOST" "test -f $N8N_SERVER_NODE_DIR/node_modules/n8n-nodes-claudecode-model/dist/nodes/LmChatClaudeCode/N8nLlmTracing.js && echo TRACE_FILE_OK"

# 3) Verify API is ready
curl -fsS "$N8N_BASE/workflows?limit=1" -H "X-N8N-API-KEY: $N8N_API_KEY" | jq '.data | length'
```

This is the complete deterministic baseline. If you preserve this structure and tracing integration, your custom model node will show execution traces the same way first-party chat model nodes do.
