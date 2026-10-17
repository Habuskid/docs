i manually copy everything i think check and help me confirm please \<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# SDK & CLI\
\
\`@paybox-sh/sdk\` is a typed Node SDK and \`paybox\` CLI for the PayBox agent API.\
It wraps the core MCP tool surface — \`list_credentials\`, \`request_payment\`,\
\`claim_payment_credentials\`, \`request_wallet_sign\`, \`request_secret\`,\
\`request_swap\`, \`get_portfolio\`, \`verify_solana_balance\`, \`get_buy_link\`,\
\`request_account_change\`, the x402 tools (\`discover_services\`, \`pay_x402\`,\
\`use_service\`), \`get_request\`, and\
the generic [plugin discovery and invocation flow]\(/reference/mcp-tools#plugin-tools).\
Core operations use the REST API; plugins use the same MCP tools as connector\
hosts, with no hardcoded per-plugin paths.\
Wallet signing is non-custodial and runs in-process: the SDK reuses the\
wallet-sign signing core and signs the MoonX envelope with your \`pbxk1.\` key;\
the MoonX secret never reaches the client.\
\
:::note\
A few surfaces are \*\*MCP-only today\*\*: geofenced World money tools; request history\
([\`list_requests\`]\(/reference/mcp-tools#list_requests)) has no SDK method or CLI\
command, and the wrappers don't expose the advanced x402 options (the v2\
\`x402_version\` + \`resource\` fields on \`pay_x402\`; custom \`headers\` and probe\
mode on \`use_service\`). Use the [MCP connector]\(/connect/mcp) for those.\
:::\
\
The request/response types are generated from the server's MCP \`list_tools\`\
schemas, so the SDK stays in sync with the API rather than being hand-maintained.\
\
\## Install\
\
Use the CLI when the agent runs on \*\*Muse\*\*, \*\*Grok Bot\*\*, \*\*Grok Computer\*\*, or any\
computer/VM with no connector screen. Use the [MCP connector]\(/connect/mcp) when\
the host has one (Claude.ai, Grok at \`grok.com\`, ChatGPT web).\
\
\`\`\`bash\
pnpm add @paybox-sh/sdk      # library\
npx @paybox-sh/sdk login     # or use the CLI directly\
\`\`\`\
\
\## Library\
\
\`\`\`ts\
import { PayboxClient } from "@paybox-sh/sdk";\
\
// Reads \~/.config/paybox + env (PAYBOX_API_KEY / PAYBOX_API_URL), or pass\
// { apiKey, signingKey } / { token } explicitly.\
const paybox = PayboxClient.fromConfig();\
\
const { credentials, ungranted_summary } = await paybox.listCredentials();\
const pay = await paybox.requestPayment({\
&#x20; credentialId: credentials[0].credential.id,\
&#x20; merchant: "Acme",\
&#x20; merchantUrl: "https\://acme.com",\
&#x20; amountCents: 1999,\
});\
\
// Count-only hints; no ungranted credential IDs, names, addresses, or metadata.\
console.log(ungranted_summary.wallet.evm);\
\
const signed = await paybox.requestWalletSign({\
&#x20; credentialId: walletId,\
&#x20; intent: { op: "message", message: "gm" },\
}); // signs in-process when a pbxk1. signing key is configured\
\
const verified = await paybox.verifySolanaBalance({\
&#x20; address: "5EUa…SViS",\
&#x20; tokenMint: "EPjF…Dt1v",\
&#x20; transactionSignature: "5h6x…q2Ms",\
});\
console.log(verified.balance.ui_amount_string);\
\
// Approval-mode operations can wait for the user's passkey approval and then\
// sign in-process — the headless equivalent of the in-chat signing window.\
const swap = await paybox.requestSwap(\
&#x20; {\
&#x20;   credentialId: walletId,\
&#x20;   srcChain: "eip155:8453",\
&#x20;   srcToken: "native",\
&#x20;   dstToken: "0x…",\
&#x20;   amount: "1000000000000000",\
&#x20; },\
&#x20; { waitForApproval: true },\
);\
\`\`\`\
\
\## CLI\
\
\`\`\`bash\
paybox login                              # OAuth 2.1 device code\
paybox login --key pbx_live\_…             # personal API key instead (no OAuth)\
paybox connect                            # pair a signing key with an eight-emoji code\
paybox login --no-open                    # print the link without opening a browser\
\
paybox credentials                        # list usable credentials\
paybox pay --credential \<id> --merchant Acme --url https\://acme.com --amount 1999\
paybox claim \<request_id>                 # claim an approved payment's one-time card\
paybox secret --credential \<id> --purpose "deploy"\
paybox sign --credential \<id> --intent '{"op":"message","message":"gm"}'\
paybox swap --credential \<id> --src-chain eip155:8453 --src-token native --dst-token 0x… --amount 1000000000000000\
paybox portfolio --address 0x… --networks 1,8453\
paybox verify-solana-balance --address 5EUa…SViS --mint EPjF…Dt1v --transaction 5h6x…q2Ms\
paybox buy-link --credential \<id> --amount-usd 50   # MoonPay checkout URL to fund a wallet\
\
paybox discover "weather api"             # browse curated paid x402 services\
paybox pay-x402 --credential \<id> --url https\://svc/api --accepts @accepts.json\
paybox use-service --credential \<id> --url https\://svc/api   # paybox pays + fetches\
\
paybox account-change --note "need a wallet grant"  # link for the user to update access\
paybox request \<request_id> --wait        # poll a pending request\
\
paybox plugins list --status enabled\
paybox plugins contract paybox://plugins/world\
paybox plugins use world world_get_events --input '{"status":"active"}'\
\
paybox version                            # installed version (+ update check)\
paybox update                             # update the global install to latest\
paybox uninstall                          # clear stored config (prints npm rm)\
\
paybox --json credentials                 # machine-readable output on any command\
\`\`\`\
\
Plugin commands are discovered at runtime rather than compiled into one command\
group per plugin. Run \`paybox plugins list\`, pass an official result's exact\
\`contract_uri\` to \`paybox plugins contract\`, then invoke the selected schema with\
\`paybox plugins use\`. Unofficial discovery results include their tools directly.\
\
The \`paybox world\` and \`paybox hyperliquid\` command groups were removed in\
1.0.0. Reach both through \`paybox plugins\`, the same way as any other plugin.\
\
\### Logging in\
\
Setup is two commands:\
\
1\. \*\*\`paybox login\`\*\* starts an OAuth 2.1 \*\*device-code\*\* grant. The CLI prints a\
&#x20;  verification URL, tries to open it locally, and polls until you choose your\
&#x20;  wallets and approve. Because it's device-code, the browser that approves\
&#x20;  doesn't have to be on the machine running the CLI: you can open the printed\
&#x20;  URL on your phone, or on a laptop while the CLI runs on a server. A scoped,\
&#x20;  audience-bound token (and a refresh token) is saved to\
&#x20;  \`\~/.config/paybox/config.json\` (mode \`0600\`). If the code expires before you\
&#x20;  approve, run \`paybox login\` again for a fresh one.\
2\. \*\*\`paybox connect\`\*\* pairs a signing key, which \`sign\` and \`swap\` need. The\
&#x20;  CLI generates the key on this machine, prints eight emoji and a link, and\
&#x20;  polls. Open the link on any device and pick the row with the same emoji. The\
&#x20;  key stays on this machine and PayBox never holds it; there is nothing to copy\
&#x20;  or paste.\
\
Add \`--no-open\` to either command to print the link without launching a browser.\
\`login\` clears a stored signing key, because a key paired to the previous client\
cannot sign, so run \`connect\` after it.\
\
Running two environments from one machine? Point \`PAYBOX_CONFIG_DIR\` at a\
separate directory per environment. \`login\` rewrites the auth block of the config\
it's pointed at, so sharing one directory across environments will overwrite the\
credentials you already had there.\
\
\### When an agent is driving\
\
The section above assumes you are at the terminal. Often you are not: the CLI\
runs on a box, and you are in a chat with the agent. Four things change.\
\
\*\*The links are the agent's job.\*\* The agent runs \`paybox login --no-open\` and\
\`paybox connect --no-open\`, reads the printed links from stderr, and puts each\
one in front of you as something you can tap. Without \`--no-open\` the CLI tries\
to open a browser on its own machine, which is useless when the machine is not\
yours. If the agent does not relay a link, the command polls until it times out\
and nothing tells you why.\
\
\*\*Nothing else should reach the chat.\*\* From the install to a working session you\
need exactly two messages: the Connect CLI link, then the eight emoji with the\
signing-key link. Nothing is pasted, and the host stores nothing, so the agent\
never needs a custom connector, a Client ID, or a host secrets store. Progress\
narration in between is noise the agent is adding.\
\
\*\*Setup is not repeatable.\*\* If \`paybox --json whoami\` already reports\
\`canSign: true\`, the work is done. Running \`login\` again sends you a second\
authorize link, which reads as the first one having failed.\
\
\`@paybox-sh/sdk\` ships a \`SKILL.md\` with this same flow written as instructions\
for the agent rather than for you. It is inside the package, at\
\`node_modules/@paybox-sh/sdk/SKILL.md\` after an install, and on the package's\
[npm page]\(https\://www\.npmjs.com/package/@paybox-sh/sdk). Point your agent at it\
and it will not need any of this page.\
\
\### Signing and approvals\
\
\`sign\`, \`swap\`, \`pay-x402\`, and \`use-service\` complete in-process when a\
\`pbxk1.\` signing key is configured (\`paybox whoami\` shows \`canSign\`) and the\
operation clears immediately under an autonomous grant. Without a key those\
commands stop at \`pending_signature\`.\
\
When a grant requires approval, pass \`--wait\`: the command waits for your\
passkey approval in the app, signs in-process, and then keeps polling until the\
request reaches a terminal state — there is no signing window on this surface.\
Without \`--wait\` you get the first snapshot back instead, which may still be\
settling (\`pending_confirmation\`), so poll \`paybox request \<id>\` yourself before\
treating the operation as finished. Solana swaps are safe to approve at your own\
pace — the SDK refreshes the transaction's recent blockhash right before signing,\
so a swap that waited on approval doesn't expire.\
\
A word on reading results if you're scripting this: the exit code is honest. A\
terminal \`denied\` or \`error\` exits non-zero, and so does a request that could not\
be made. With \`--json\`, those failures also come back as a JSON object on stdout,\
so you can read one channel whichever way it went. Bad usage is the exception:\
a missing flag or an unknown command is reported as plain text on stderr before\
the command runs, so check the exit code there rather than parsing stdout.\
\
You can also integrate over the \*\*[API reference]\(/api-reference)\*\* directly, or\
connect an agent via the \*\*[MCP connector]\(/connect/mcp)\*\*.\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# Getting started\
\
This walks through the fastest path from zero to an agent that can transact:\
sign in, vault a credential, and connect an agent over MCP.\
\
\## 1. Sign in\
\
Open the PayBox app and create an account. Sign-in is \*\*email + passkey\*\* —\
PayBox requires a passkey assertion at every sign-in, and uses it again to\
approve sensitive operations. (Recovery tip: enroll a passkey on a second\
device so you're never locked out.)\
\
\## 2. Vault a credential\
\
In \*\*Credentials\*\*, add one of:\
\
\* a \*\*secret\*\* (e.g. an API key),\
\* a \*\*card\*\* (vaulted via Basis Theory — PayBox never sees the raw PAN),\
\* a \*\*wallet\*\* (created via MoonX MPC — the private key never leaves the TEE).\
\
\## 3. Connect an agent\
\
Two ways to give an agent scoped access:\
\
\* \*\*[MCP connector]\(/connect/mcp)\*\* — paste one URL into your agent's connector\
&#x20; list, sign in once, and approve a scoped grant. Best for Claude.ai, Grok at\
&#x20; \`grok.com\`, Open WebUI, and other MCP clients with a connector screen.\
\* \*\*[SDK & CLI]\(/sdk-cli)\*\* — required for \*\*Muse\*\*, \*\*Grok Bot\*\*, \*\*Grok Computer\*\*, and\
&#x20; any agent running on a computer or VM with no connector UI. Install\
&#x20; \`@paybox-sh/sdk\` on that machine and follow the package \`SKILL.md\`.\
\* \*\*[OAuth 2.1]\(/connect/oauth)\*\* — for apps that integrate directly: standard\
&#x20; authorization-code + PKCE, after which the app holds a scoped access token.\
\
\## 4. Stay in control\
\
\* Pick an \*\*approval mode\*\* per grant: \*human approval\* (every op pauses for your\
&#x20; passkey) or \*autonomous\* (acts within the grant).\
\* Above-threshold and sensitive operations always pause for a \*\*passkey\*\*.\
\* Every operation is an \*\*audit event\*\*, visible in the app.\
\* Revoke any agent client at any time.\
\
:::tip\
New to the model? Read \*\*[Credentials & agents]\(/concepts/model)\*\* next — it\
explains credentials, clients, grants, and approval modes in one place.\
:::\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# Credentials & agents \\[The PayBox model]\
\
Four concepts cover everything PayBox does.\
\
\## Credential\
\
Something sensitive you vault once. Three types:\
\
\| Type | Vaulted via | Agents receive |\
\| --- | --- | --- |\
\| \*\*Card\*\* | Basis Theory tokenization (no raw PAN in PayBox) | a one-time virtual card |\
\| \*\*Wallet\*\* | MoonX MPC (key never leaves the TEE) | a signed transaction |\
\| \*\*Secret\*\* | Envelope-encrypted (KMS) | a short-lived token (or proxied use) |\
\
\## Client\
\
An \*\*agent integration\*\* — Claude Code, an OpenAI agent, your own MCP server —\
runs under its own \*\*client key\*\*. One agent, one key. Rotate or revoke a client\
without touching the underlying credentials.\
\
\## Grant\
\
A client is given \*\*grants\*\*: which credentials it may use, and under what\
\*\*approval mode\*\*. Grants are scoped per credential.\
\
\## Approval mode\
\
Set per grant:\
\
\* \*\*Human approval\*\* — every operation pauses for your passkey approval.\
\* \*\*Autonomous\*\* — the agent acts without per-op approval, \*within the grant\*.\
\
Regardless of mode, \*\*sensitive operations\*\* (revealing a raw secret, signing,\
issuing keys) require a fresh passkey assertion, and every operation is an audit\
event. See \*\*[Approvals & passkeys]\(/concepts/approvals)\*\*.\
\
:::info\
Scoped outputs, never raw credentials. An agent with a card grant can request a\
charge; it never sees the card number. A wallet grant yields signatures, never\
key shares.\
:::\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# Approvals\
\
PayBox authenticates every protected account request with a live MoonX\
session (access token), verified statelessly. This proves account ownership;\
no separate proof-of-presence check runs server-side.\
\
\## Approval flow\
\
When an agent requests something that needs sign-off, PayBox creates a \*\*pending\
approval\*\*. Wallet grants can use \*\*iframe\*\* mode: the user confirms in the\
in-chat signing window, which then signs and submits immediately. App \*\*always\
approve\*\* still requires the user to confirm in PayBox (and a push notification\
on mobile).\
\
Approvals are \*\*operation-bound\*\*: change a parameter (amount, merchant, payload)\
and it's a new request — an approval can't be replayed against different\
operation details.\
\
:::warning\
A removed passkey can't be restored from email alone. If passkeys are the only\
thing protecting your account and you remove the last one, the account is locked\
for security. Keep a passkey on a second device.\
:::\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# Request lifecycle \\[From intent to result]\
\
Every operation an agent requests — a payment, a signature, a secret, a swap —\
runs through the same lifecycle. Understanding it is the difference between an\
integration that waits correctly and one that double-charges.\
\
\## The shape of every request\
\
An agent submits an \*\*intent\*\* (via an [MCP tool]\(/reference/mcp-tools)). PayBox\
evaluates it against the grant and returns one of:\
\
::::steps\
\
\### Allow (autonomous)\
\
The grant's approval mode is \*autonomous\* and the operation is within its limits.\
Payments and secrets complete immediately (\`success\`). Wallet signs and swaps\
move to \`pending_signature\` — they still need the [signing\
window]\(#the-signing-window) to produce the artifact.\
\
\### Needs approval\
\
The grant requires human approval, or the operation crosses a threshold. PayBox\
returns \`pending_approval\` with an \`approval_url\`. Surface it to the user; they\
approve with a passkey in the PayBox app.\
\
\### Deny\
\
Policy rejects it outright (e.g. a chain or merchant not in the grant). PayBox\
returns \`denied\` with a \`reason\`.\
\
::::\
\
\## Statuses\
\
\| Status | Terminal? | Meaning |\
\| --- | --- | --- |\
\| \`pending_approval\` | no | Waiting on the user's passkey approval. |\
\| \`pending_signature\` | no | Approved or autonomous; the signing window is producing the artifact (wallet sign / swap only). |\
\| \`pending_settlement\` | no | A cross-chain swap's source tx is broadcast; the bridge is settling on the destination chain. Each poll re-checks and flips it forward. |\
\| \`pending_confirmation\` | no | Broadcast but not yet confirmed on-chain. Each poll re-checks and flips to \`success\` (confirmed) or \`error\` (dropped or failed). |\
\| \`success\` | yes | Done — \`output\` carries the result. For on-chain operations this means \*\*confirmed on-chain\*\*, never merely broadcast. |\
\| \`denied\` | yes | A policy or user decision. Carries a \`reason\`. |\
\| \`error\` | yes | An internal failure (not a decision). Carries a \`message\`. |\
\
\## The poll pattern\
\
The one rule that matters: \*\*submit once, then poll\*\* — never re-issue the\
original tool call to "finish" it. Re-calling starts a \*new\* operation (a second\
charge, a second signature).\
\
::::steps\
\
\### Submit\
\
Call the \`request\_\*\` tool. Keep the \`request_id\` from the result.\
\
\### Surface approval (if pending)\
\
If \`status\` is \`pending_approval\`, show the \`approval_url\`. The user opens it in\
the PayBox app and approves with their passkey. Approvals are \*\*operation-bound\*\*\
and expire after \~10 minutes — change any parameter and it's a new request, so an\
approval can never be replayed against different details.\
\
\### Poll\
\
Call [\`get_request\`]\(/reference/mcp-tools#get_request) with the \`request_id\`\
until it reaches a terminal status. It walks \`pending_approval\` →\
\`pending_signature\` → \`success\` on its own as the user approves and the signing\
window completes.\
\
\### Use the output\
\
On \`success\`, read \`output\` — a payment token, a \`0x\` signature, a broadcastable\
serialized transaction, a swap transaction hash, or a secret token.\
\
::::\
\
\## The signing window\
\
Wallet signing never happens in agent code. When a wallet sign or swap is cleared\
(\`pending_signature\`), the MCP host opens the \*\*signing window\*\* — an interactive\
UI rendered in the chat (the MCP resource \`ui://paybox/wallet-sign\`).\
\
Hosts differ in \*when\* they draw it. Most draw it as soon as the tool result\
arrives. Grok draws it only once the agent's reply is finished, so nothing is on\
screen while the agent is still writing — the agent hands off and ends its turn\
rather than polling for a window that isn't drawn yet.\
\
\* The window signs \*\*client-side\*\* via MoonX MPC. The private key never reaches\
&#x20; PayBox, the agent, or the model.\
\* It posts the signed artifact back to PayBox itself. The agent's only job is to\
&#x20; check \`get_request\` until \`success\`.\
\
So a basic wallet flow looks like: \`request_wallet_sign\` → (\`pending_approval\`\
→ user passkey) → \`pending_signature\` → \*window signs\* → \`success\` with the\
signature.\
\
The same window also handles higher-level wallet operations:\
\
\* \`request_swap\` signs the swap transaction or gasless UserOperation envelopes;\
&#x20; PayBox broadcasts after the window submits the signatures.\
\* \`pay_x402\` signs the EIP-3009 typed data; PayBox assembles the x402 payment\
&#x20; header and, in gateway mode, can re-fetch the paid resource.\
\* When x402 auto-undelegate is enabled, delegated x402 payers use one composite\
&#x20; window flow: undelegate first, verify code is gone, then sign the x402 payment.\
\
These operations still follow the same rule: the agent submits once, surfaces\
approval if needed, and polls \`get_request\`.\
\
:::info\
The window's own calls back to PayBox use internal tools that are hidden from the\
agent. You don't implement or call them — just surface \`approval_url\` when asked\
and poll for the result.\
:::\
\
\## Audit\
\
Every step — request, approval, denial, completion, revocation — is an audit\
event tied to the user, the client, and the request. Users review the trail in\
the app; see \*\*[Approvals & passkeys]\(/concepts/approvals)\*\* for the auth tiers\
each action requires.\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# MCP connector\
\
PayBox speaks the \*\*Model Context Protocol\*\*. Add one URL to your agent's\
connector list, sign in once, and the agent gets scoped, passkey-gated access to\
your wallets, cards, and secrets.\
\
\## The URL\
\
\`\`\`txt\
https\://api.paybox.sh/mcp\
\`\`\`\
\
It's a \*\*streamable HTTP\*\* MCP endpoint (protocol version \`2025-06-18\`).\
\
\## Steps\
\
::::steps\
\
\### Add the URL\
\
\*\*Muse\*\*, \*\*Grok Bots\*\*, \*\*Grok Computer\*\*, and other agents on a computer have no\
connector screen on the machine where the agent runs. Use the [CLI]\(/sdk-cli) on\
that machine instead (\`npm install -g @paybox-sh/sdk\`, then the package\
\`SKILL.md\`). Same API, same grants, same passkey approvals. Do not add a custom\
connector, enter a Client ID, or use a host secrets store there. Chat products\
built on the same model often have no connector screen either; install the CLI\
on the machine the agent runs on.\
\
At \`claude.ai/new#settings/customize-connectors\` (Customize → Connectors), or your\
agent's equivalent, paste the MCP URL above as a custom connector.\
\
At \`grok.com\` it's \`grok.com/connectors\` → \*\*New Connector\*\* → \*\*Custom\*\* → paste\
the URL above, then complete the sign-in it opens. That path is \*\*Grok\*\*, not\
Grok Bots.\
\
\### Sign in\
\
Your agent opens PayBox in a tab. Sign in (email + passkey) and \*\*approve a\
scoped grant\*\* on just the credentials you want this agent to use, with an\
approval mode per credential.\
\
\### Stay in control\
\
Above-threshold and sensitive operations pause for your passkey. Every operation\
is an audit event. Revoke the connector from the app's \*\*Clients\*\* screen at any\
time.\
\
::::\
\
\## Signing: in the chat, or in your browser\
\
PayBox never holds your wallet key. Anything that spends or signs is confirmed by\
you in the \*\*PayBox signing window\*\*, which holds your key and signs there and\
nowhere else.\
\
Agents render that window inside the chat. It appears next to the conversation and\
signs as soon as your key is connected to it — you take no separate action.\
\
What differs is \*when\* it appears. Most agents draw it the moment the tool runs.\
Grok draws it only after the agent finishes replying, so there's a gap where\
nothing is on screen yet. That's expected: let the reply finish, the window shows\
up, you sign, and the agent reports the outcome on its next message.\
\
Your agent is told which applies to it, so you don't have to work it out.\
\
\## How it works\
\
The MCP endpoint is the agent's whole interface to PayBox. Adding the URL drives\
an \*\*[OAuth 2.1 + PKCE]\(/connect/oauth)\*\* flow under the hood: the client\
registers dynamically, you consent to a scoped grant, and the client receives an\
access token it presents as \`Authorization: Bearer …\` on every call.\
\
From there the agent calls the \*\*[MCP tools]\(/reference/mcp-tools)\*\* —\
\`list_credentials\`, \`request_payment\`, \`claim_payment_credentials\`,\
\`request_wallet_sign\`, \`request_secret\`, \`request_swap\`, \`get_portfolio\`,\
\`verify_solana_balance\`,\
\`get_buy_link\`, the x402 tools (\`discover_services\`, \`pay_x402\`, \`use_service\`),\
\`request_account_change\`, \`get_request\`, and \`list_requests\` — plus\
[\`discover_plugins\`]\(/reference/mcp-tools#plugin-tools), \`get_contract\`, and\
\`use_plugin\` for \*\*[plugin tools]\(/reference/mcp-tools#plugin-tools)\*\* the user\
has enabled (they also appear on MCP \`tools/list\`). All the same approval, policy, and audit\
rules apply, and wallet signing happens in an in-chat \*\*[signing\
window]\(/concepts/requests#the-signing-window)\*\* so the key never leaves MoonX\
MPC.\
\
:::tip\
New to the flow? \*\*[Request lifecycle]\(/concepts/requests)\*\* explains the\
submit → approve → poll pattern every tool follows.\
:::\
\
\## A Claude Code request is refused before it runs\
\
If Claude Code refuses a swap with language about being unauthorized for\
financial operations, run \`claude --debug-file \<path>\` to capture a log. Then\
inspect it for the \`tools/call\` for \`request_swap\`. That call is the evidence\
boundary.\
\
\* No \`tools/call\` means the host declined before Paybox could evaluate the\
&#x20; request. Check Claude Code custom instructions, skills, and \`/permissions\`\
&#x20; for rules that prohibit financial operations. Then use Claude Chat with the\
&#x20; same connected Paybox account as the safe workaround.\
\* A \`request_swap\` response means Paybox received the call. Keep the exact\
&#x20; response, including an error response without a \`request_id\`. A parked\
&#x20; request includes a \`request_id\` that you can use with \`get_request\`. Check\
&#x20; that \`list_credentials\` shows the intended wallet and grant before\
&#x20; reconnecting or changing authorization.\
\
Do not change a wallet grant or approval mode to address a refusal that occurs\
before the tool call. Those controls only apply after Paybox receives a request.\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# OAuth 2.1\
\
PayBox is an \*\*OAuth 2.1 Authorization Server\*\* implementing the MCP\
authorization spec. Third-party apps and MCP clients connect with standard\
\*\*authorization-code + PKCE\*\*; after the user consents, your app holds a scoped\
access token it presents on the MCP endpoint.\
\
All endpoints live on the API origin:\
\
\`\`\`txt\
https\://api.paybox.sh\
\`\`\`\
\
The consent screen the user sees is served from the app origin\
(\`https\://app.paybox.sh\`). Everything else below is on the API origin.\
\
:::info\
Most integrations don't implement this by hand — an MCP client (Claude.ai, Grok,\
…) drives the whole flow when you add the \*\*[MCP connector]\(/connect/mcp)\*\* URL.\
This page is for building that client, or integrating directly.\
:::\
\
\## Flow\
\
::::steps\
\
\### Discover\
\
Fetch the metadata documents (no auth):\
\
\`\`\`txt\
GET /.well-known/oauth-authorization-server\
GET /.well-known/oauth-protected-resource\
\`\`\`\
\
The authorization-server document tells you every endpoint and capability:\
\
\`\`\`json\
{\
&#x20; "issuer": "https\://api.paybox.sh",\
&#x20; "authorization_endpoint": "https\://api.paybox.sh/oauth/authorize",\
&#x20; "token_endpoint": "https\://api.paybox.sh/oauth/token",\
&#x20; "registration_endpoint": "https\://api.paybox.sh/oauth/register",\
&#x20; "response_types_supported": ["code"],\
&#x20; "grant_types_supported": ["authorization_code", "refresh_token"],\
&#x20; "code_challenge_methods_supported": ["S256"],\
&#x20; "token_endpoint_auth_methods_supported": ["none"],\
&#x20; "scopes_supported": ["mcp", "offline_access"]\
}\
\`\`\`\
\
The protected-resource document points at the MCP resource and its authorization\
server:\
\
\`\`\`json\
{\
&#x20; "resource": "https\://api.paybox.sh/mcp",\
&#x20; "authorization_servers": ["https\://api.paybox.sh"],\
&#x20; "bearer_methods_supported": ["header"],\
&#x20; "scopes_supported": ["mcp"]\
}\
\`\`\`\
\
\### Register (dynamic)\
\
Register a public client (RFC 7591). PayBox supports public clients only —\
\`token_endpoint_auth_method\` must be \`none\`; there is no client secret.\
\
\`\`\`http\
POST /oauth/register\
Content-Type: application/json\
\
{\
&#x20; "client_name": "Acme Agent",\
&#x20; "redirect_uris": ["https\://acme.example.com/oauth/callback"],\
&#x20; "token_endpoint_auth_method": "none"\
}\
\`\`\`\
\
\`\`\`json\
// 201 Created\
{\
&#x20; "client_id": "pbx-oauth-a1b2c3d4e5f6",\
&#x20; "client_name": "Acme Agent",\
&#x20; "redirect_uris": ["https\://acme.example.com/oauth/callback"],\
&#x20; "token_endpoint_auth_method": "none",\
&#x20; "grant_types": ["authorization_code", "refresh_token"],\
&#x20; "response_types": ["code"],\
&#x20; "client_id_issued_at": 1749200000\
}\
\`\`\`\
\
\`redirect_uris\` must be HTTPS (or \`http\://localhost\` for local dev).\
\
\### Authorize (PKCE)\
\
Send the user to \`/oauth/authorize\` with a PKCE \`code_challenge\` (S256). PayBox\
validates the request, then redirects the user to the consent screen, where they\
sign in, \*\*select which credentials to grant\*\* and the approval mode per\
credential, and approve with a passkey.\
\
\`\`\`txt\
GET /oauth/authorize\
&#x20; ?response_type=code\
&#x20; &client_id=pbx-oauth-a1b2c3d4e5f6\
&#x20; &redirect_uri=https\://acme.example.com/oauth/callback\
&#x20; &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM\
&#x20; &code_challenge_method=S256\
&#x20; &scope=mcp%20offline_access\
&#x20; &resource=https\://api.paybox.sh/mcp\
&#x20; &state=xyz\
\`\`\`\
\
On approval the user's browser is redirected back to your \`redirect_uri\` with a\
\`code\` (and your \`state\`):\
\
\`\`\`txt\
https\://acme.example.com/oauth/callback?code=\<authorization_code>&state=xyz\
\`\`\`\
\
\### Exchange\
\
Exchange the code (with your PKCE \`code_verifier\`) for tokens:\
\
\`\`\`http\
POST /oauth/token\
Content-Type: application/x-www-form-urlencoded\
\
grant_type=authorization_code\
&client_id=pbx-oauth-a1b2c3d4e5f6\
&code=\<authorization_code>\
&redirect_uri=https\://acme.example.com/oauth/callback\
&code_verifier=\<pkce_verifier>\
\`\`\`\
\
\`\`\`json\
{\
&#x20; "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9…",\
&#x20; "token_type": "Bearer",\
&#x20; "expires_in": 3600,\
&#x20; "refresh_token": "…",\
&#x20; "scope": "mcp offline_access"\
}\
\`\`\`\
\
Now call the [MCP endpoint]\(/connect/mcp) with \`Authorization: Bearer \<access_token>\`.\
\
::::\
\
\## Scopes\
\
\| Scope | Grants |\
\| --- | --- |\
\| \`mcp\` | Access to the MCP tools. |\
\| \`offline_access\` | A \`refresh_token\` so the client can stay connected. Opt-in — omit it and no refresh token is issued. |\
\
\## Tokens\
\
\| Token | Format | TTL | Notes |\
\| --- | --- | --- | --- |\
\| Authorization code | opaque | 5 min | Single-use. Replaying a used code revokes the client (treated as compromise). |\
\| Access token | JWT (HS256) | 60 min | Audience-bound to the MCP \`resource\`. Presented as a bearer on \`/mcp\`. |\
\| Refresh token | opaque | 30 days, sliding | \*\*Rotated on every use\*\* — the response returns a new one; replace your stored token. Replaying a rotated token revokes the client. |\
\
The token's authority is exactly the \*\*grant set\*\* the user approved — which\
credentials, and the approval mode per credential. Revoking the client (from the\
app's \*\*Clients\*\* screen) immediately kills its access tokens and refresh chain.\
\
\### Refresh\
\
\`\`\`http\
POST /oauth/token\
Content-Type: application/x-www-form-urlencoded\
\
grant_type=refresh_token\
&client_id=pbx-oauth-a1b2c3d4e5f6\
&refresh_token=\<current_refresh_token>\
\`\`\`\
\
The response is a new access token \*\*and a new refresh token\*\*. Always store the\
new refresh token; the old one is now spent.\
\
\## Notes\
\
\* \*\*PKCE S256 is required.\*\* \`code_challenge_method\` must be \`S256\`.\
\* \`redirect_uri\` is matched \*\*exactly\*\* against what was registered. A\
&#x20; mismatched or malformed \`redirect_uri\` is rejected with an error page, not a\
&#x20; redirect (sending errors to an unverified URI is unsafe).\
\* Sensitive operations still pause for the \*\*user's passkey\*\* regardless of the\
&#x20; app's token — the token alone never bypasses step-up. See\
&#x20; \*\*[Request lifecycle]\(/concepts/requests)\*\*.\
\
\## Errors\
\
Token-endpoint failures use standard OAuth error codes:\
\
\| Error | Cause |\
\| --- | --- |\
\| \`invalid_grant\` | Code expired, already used, or PKCE verification failed. |\
\| \`invalid_client\` | Unknown \`client_id\`. |\
\| \`invalid_request\` | Missing or malformed parameters. |\
\| \`invalid_scope\` | Requested scope exceeds what was granted. |\
\
A \`/mcp\` call with a missing or invalid bearer returns \`401\` with a\
\`WWW-Authenticate\` header pointing back at \`/.well-known/oauth-protected-resource\`\
so the client can re-run discovery and re-authorize.\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# MCP tools \\[What an agent can call]\
\
These are the tools the PayBox MCP server exposes to a connected agent. Once an\
agent holds a scoped access token (see \*\*[OAuth 2.1]\(/connect/oauth)\*\*) it calls\
them over the MCP endpoint at \`https\://api.paybox.sh/mcp\` (streamable HTTP,\
protocol version \`2025-06-18\`).\
\
Every tool acts \*\*on behalf of the signed-in user\*\*, scoped to the credentials\
they granted this client. An agent never sees a raw card, key share, or secret —\
it gets a \*\*scoped output\*\* (a one-time card, a signature, a short-lived token),\
and anything sensitive can pause for the user's passkey.\
\
:::info\
The agent only ever calls the tools below. Signing a wallet operation happens in\
the \*\*[signing window]\(/concepts/requests#the-signing-window)\*\* — an in-chat UI\
the host renders — not in agent code. The internal \`submit\_\*\` / \`moonx\_\*\` tools\
the window uses are hidden from the model and are not part of this surface.\
:::\
\
\## The result envelope\
\
Most write tools return a status-tagged result. Branch on \`status\`:\
\
\| \`status\` | Meaning | What to do |\
\| --- | --- | --- |\
\| \`success\` | Completed. \`output\` carries the result. For on-chain operations this means \*\*confirmed on-chain\*\*, never merely broadcast. | Use \`output\`. |\
\| \`pending_approval\` | The user must approve with their passkey. | Surface \`approval_url\`, then poll \`get_request\`. |\
\| \`pending_signature\` | Cleared to sign (autonomous, or already approved). The signing window finishes it. | Poll \`get_request\` for the artifact. |\
\| \`pending_settlement\` | A cross-chain swap's source tx is broadcast; the bridge is settling on the destination chain. | Keep polling \`get_request\`. |\
\| \`pending_confirmation\` | Broadcast but not yet confirmed on-chain. | Keep polling \`get_request\`; it flips to \`success\` (confirmed) or \`error\` (dropped). |\
\| \`denied\` | Policy or the user rejected it. | Read \`reason\`; do not retry blindly. |\
\| \`error\` | Something failed (not a policy decision). | Read \`message\`. |\
\
The golden rule: \*\*never re-call a write tool to "finish" it\*\* — that starts a\
new operation. Always poll \*\*[\`get_request\`]\(#get_request)\*\* with the\
\`request_id\` you got back. See \*\*[Request lifecycle]\(/concepts/requests)\*\* for\
the full state machine.\
\
\*\*\*\
\
\## list\\\_credentials\
\
Discover the credentials this client may use. Call it first — every other tool\
takes a \`credential_id\` from here.\
\
\*\*Input:\*\* none.\
\
\*\*Returns:\*\*\
\
\`\`\`json\
{\
&#x20; "credentials": [\
&#x20;   {\
&#x20;     "credential_id": "9f8e…",\
&#x20;     "name": "Personal Visa",\
&#x20;     "kind": "card",\
&#x20;     "metadata": { "brand": "VISA", "last4": "4242" },\
&#x20;     "approval_mode": "always_approve"\
&#x20;   },\
&#x20;   {\
&#x20;     "credential_id": "1a2b…",\
&#x20;     "name": "Trading wallet",\
&#x20;     "kind": "wallet",\
&#x20;     "metadata": {\
&#x20;       "provider_wallet_id": "b3f1…",\
&#x20;       "chains": ["evm"],\
&#x20;       "address": "0x70a0…2b1f"\
&#x20;     },\
&#x20;     "approval_mode": "autonomous"\
&#x20;   },\
&#x20;   {\
&#x20;     "credential_id": "7c6d…",\
&#x20;     "name": "OpenAI key",\
&#x20;     "kind": "secret",\
&#x20;     "metadata": { "encrypted": true },\
&#x20;     "approval_mode": "always_approve"\
&#x20;   }\
&#x20; ],\
&#x20; "ungranted_summary": {\
&#x20;   "wallet": { "evm": 1, "solana": 0 },\
&#x20;   "card": 0,\
&#x20;   "secret": 0\
&#x20; }\
}\
\`\`\`\
\
\* \`kind\` is \`card\` | \`wallet\` | \`secret\`.\
\* \`approval_mode\` is \`always_approve\` (passkey in the PayBox app), \`iframe\` (confirm in the signing window), or \`autonomous\`\
&#x20; (acts within the grant — see \*\*[the model]\(/concepts/model)\*\*).\
\* \`ungranted_summary\` contains only counts by credential kind and wallet family.\
&#x20; It never includes an ungranted credential's ID, name, address, last4, or\
&#x20; metadata. A non-zero count means the credential exists in PayBox but has not\
&#x20; been granted to this connector.\
\* \`metadata\` is public display data only:\
&#x20; \* \*\*wallet\*\* → \`address\`, \`provider_wallet_id\`, and \`chains\` — the chain\
&#x20;   \*\*family\*\* (\`["evm"]\` or \`["solana"]\`), not a per-chain list. An EVM MPC\
&#x20;   wallet shares one address across every \`eip155:\*\` chain. \`address\` may be\
&#x20;   \`null\` for older wallets until it's captured.\
&#x20; \* \*\*card\*\* → \`brand\`, \`last4\` (client-attested and unverified — never\
&#x20;   authorize on them). The saved card token is never returned.\
&#x20; \* \*\*secret\*\* → \`{ "encrypted": true }\`.\
\
\*\*\*\
\
\## request\\\_payment\
\
Authorize issuance of a merchant-scoped, one-time virtual card for a \*\*card\*\*\
credential. This does \*\*not\*\* submit the merchant checkout, charge the merchant,\
or top up an account by itself.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A \`card\`-kind credential from \`list_credentials\`. |\
\| \`merchant\` | string | Merchant identifier — shown to the user at approval and in the audit log. |\
\| \`merchant_url\` | string | The real HTTPS origin Basis Theory binds the one-time card to. Don't synthesize it from the name. |\
\| \`amount_cents\` | integer | Amount in cents. |\
\| \`currency\` | string | ISO 4217. Currently \`USD\` only. |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope). On \`success\`, \`output\`\
carries the one-time virtual-card details when immediately available. For\
human-approved/cardholder-verified payments, \`get_request\` intentionally\
redacts the card after approval; call\
[\`claim_payment_credentials\`]\(#claim_payment_credentials) once to get the\
usable card. The agent must then use those card details at the merchant's\
checkout/API and only report the purchase complete after the merchant confirms\
the payment or credit top-up.\
\
\*\*\*\
\
\## claim\\\_payment\\\_credentials\
\
Claim the usable card details of an \*\*approved\*\* payment. For human-approved\
payments the polled request redacts the card, so this is the one call that\
returns it — \*\*one-time\*\*: the claim is consumed, and a second call fails.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`request_id\` | string | An approved \`request_payment\` request. |\
\
\*\*Returns\*\* a credential output (\`output_type: "payment_token"\`) whose \`value\`\
carries the one-time virtual-card details, plus its \`expires_at\`. Claim it only\
when you're ready to pay at the merchant's checkout.\
\
\*\*\*\
\
\## request\\\_wallet\\\_sign\
\
Sign with a \*\*wallet\*\* credential. You pass \*what\* to sign as an \`intent\` —\
\*\*not\*\* a pre-hashed digest. PayBox builds the correct bytes and the\
[signing window]\(/concepts/requests#the-signing-window) signs it client-side; the\
private key never leaves MoonX MPC.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A \`wallet\`-kind credential. |\
\| \`intent\` | object | What to sign — see the intent table below. The chain, destination, and value are read from the intent itself; there are no separate declared fields. |\
\
The \`intent\` is tagged by an \`op\` field:\
\
\| \`op\` | Shape | Result on success |\
\| --- | --- | --- |\
\| \`message\` | \`{ "op": "message", "message": "…" }\` | EIP-191 \`{ "signature": "0x…" }\` |\
\| \`typedData\` | \`{ "op": "typedData", "typedData": { … } }\` | EIP-712 \`{ "signature": "0x…" }\` |\
\| \`transaction\` | \`{ "op": "transaction", "transaction": { …eip1559… } }\` | \`{ "serializedTransaction": "0x…" }\` (ready for \`eth_sendRawTransaction\`) |\
\| \`authorization\` | \`{ "op": "authorization", "authorization": { … } }\` | EIP-7702 \`{ "authorization": { … } }\` |\
\| \`solanaMessage\` | \`{ "op": "solanaMessage", "address": "\<base58>", "message": "…" }\` | \`{ "signature": "\<hex>" }\` |\
\| \`solanaTransaction\` | \`{ "op": "solanaTransaction", "address": "\<base58>", "transactionBase64": "…" }\` | \`{ "signedTransactionBase64": "…" }\` |\
\| \`raw\` | \`{ "op": "raw", "rawSigningPayloadHex": "0x…" }\` | Signs a pre-computed digest as-is (escape hatch). |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope):\
\
\* \`pending_signature\` — autonomous: the signing window can sign right away.\
\* \`pending_approval\` — the user approves with their passkey \*\*first\*\*; surface\
&#x20; \`approval_url\`, then the window signs.\
\
Either way, \*\*poll \`get_request\`\*\* for the signed artifact in \`output\`.\
\
\`\`\`json\
// example call\
{\
&#x20; "credential_id": "1a2b…",\
&#x20; "intent": { "op": "message", "message": "Sign in to Acme #4821" }\
}\
\`\`\`\
\
\*\*\*\
\
\## request\\\_secret\
\
Reveal a \*\*secret\*\* credential (e.g. an API key) so the agent can use it.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A \`secret\`-kind credential. |\
\| \`raw\` | boolean | \`false\` (default) returns a one-time-use \`secret_token\` for PayBox-mediated egress; \`true\` returns the actual plaintext (only if the grant allows raw). |\
\| \`purpose\` | string? | Why the secret is needed — shown at approval, recorded in audit. |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope). Prefer \`raw: false\` so\
the plaintext never transits the model. Returns \`pending_approval\` when policy\
requires the user's passkey.\
\
\*\*\*\
\
\## request\\\_swap\
\
Swap one token for another from a \*\*wallet\*\* credential. You submit an\
\*\*intent\*\* (what to swap); PayBox quotes the route via MoonX, builds the\
transactions, the signing window signs them, and the server broadcasts.\
\
:::warning\
On Solana (\`solana:\*\`), \`src_token\` and \`dst_token\` must be exact SPL mint\
addresses or \`"native"\` for SOL — never symbols like \`USDC\` or \`SOL\`. Mainnet\
USDC mint: \`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v\`. Native SOL:\
\`"native"\` in swaps (reference mint\
\`So11111111111111111111111111111111111111111\`).\
:::\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A \`wallet\`-kind credential. |\
\| \`src_chain\` | string | CAIP-2 source chain — an \`eip155:\*\` chain for an EVM wallet, \`solana:\*\` for Solana. |\
\| \`dst_chain\` | string? | Defaults to \`src_chain\`. Set a different chain to \*\*bridge\*\*: after the source tx broadcasts, \`get_request\` returns \`pending_settlement\` until the bridge delivers. |\
\| \`src_token\` | string | Token address, or \`"native"\` for the chain's native asset. On Solana, pass mint addresses only — never symbols. |\
\| \`dst_token\` | string | Destination token address. On Solana, pass mint addresses only — never symbols. |\
\| \`amount\` | string | Amount in \`src_token\`'s smallest unit, as a decimal string. |\
\| \`swap_direction\` | string? | \`exact-amount-in\` (default) or \`exact-amount-out\`. |\
\| \`slippage_bps\` | integer? | Default \`50\` (0.5%). Becomes the on-chain \`amountOutMin\` floor. |\
\| \`recipient\` | string? | Defaults to the wallet's own address. For a cross-VM swap (e.g. Solana → Base) pass an address on \`dst_chain\`; omitted, it defaults to the user's single vaulted wallet there. |\
\| \`value_cents\` | integer? | Rough USD value of the sell side, for policy. |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope): \`pending_signature\`\
(autonomous) or \`pending_approval\`. A bad quote (e.g. an unsupported token pair)\
fails \*\*here\*\*, synchronously. On completion, \`get_request\` returns the swap\
transaction hash in \`output\`; a cross-chain swap's \`success\` usually includes\
the destination \`dst_tx_hash\` too.\
\
:::tip\
Resolve token addresses with \*\*[\`discover_tokens\`]\(#discover_tokens)\*\*, then use\
\*\*[\`get_portfolio\`]\(#get_portfolio)\*\* to size the \`amount\` from available funds\
before you swap.\
:::\
\
\*\*\*\
\
\## discover\\\_tokens\
\
Opens \*\*Explore\*\* in the shared PayBox iframe. Search by name, symbol or contract\
address, browse \`trending\`, \`gainers\` or \`new\`, or request \`token_detail\` with a\
price chart. Pass \`chain\` as a name, slug, numeric network ID or \`eip155:\<id>\`;\
it defaults to Solana. Token results include addresses, decimals and market data.\
\
Hyperliquid uses \`eip155:999\`, native HYPE, and the chain name \`hyperliquid\`;\
\`hyperevm\` is accepted as an alias.\
\
Use \`mode: "chains"\` for featured networks (including Robinhood, Arc, and Hyperliquid), or\
add \`query\` to search the full production-chain catalog. Supports optional\
\`query\`, \`limit\` (up to 100) and \`offset\`; follow \`next_offset\` for more results.\
Catalog membership does not guarantee provider coverage or trading support.\
Numeric IDs let newly supported provider networks work without a new chain enum.\
\
For swaps: resolve the exact token addresses with \`discover_tokens\`, check the\
source wallet's balance with \`get_portfolio\`, then call \`request_swap\`.\
Discovery is read-only and moves no funds.\
\
\## get\\\_portfolio\
\
List a wallet's token balances across chains (public on-chain data via MoonX).\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`address\` | string | Wallet address (EVM \`0x…\` or Solana base58). |\
\| \`network_ids\` | string? | Comma-separated network IDs (e.g. \`"1,8453,137"\`); omit to auto-detect from the address. |\
\
\*\*Returns\*\* the requested wallet \`address\`, \`wallet_name\` when it matches an\
active owned wallet, each holding's token address, resolved \`name\` + \`symbol\`,\
24h \`priceChange24h\` (a fraction, e.g. \`0.0052\` = +0.52%; \`null\` when unknown),\
raw + USD balance, network id, freshness metadata, and \`total_usd\` (\`null\` if\
any nonzero holding is unpriced).\
When World is enabled, outcome \`balanceUsd\` is a number from the held side's\
orderbook midpoint, or \`null\` with \`price_unavailable_reason\`.\
\
\*\*\*\
\
\## verify\\\_solana\\\_balance\
\
Verify that one confirmed Solana transaction changed an exact wallet and SPL or\
Token-2022 mint, then read the current token balance at or after that\
transaction's slot. This read comes directly from the configured Alchemy Solana\
Mainnet RPC and does not wait for MoonX's portfolio index.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`address\` | string | Canonical base58 Solana wallet address. |\
\| \`token_mint\` | string | Canonical base58 SPL or Token-2022 mint — never a symbol. Mainnet USDC: \`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v\`. |\
\| \`transaction_signature\` | string | Canonical base58 64-byte transaction signature. |\
\
\*\*Returns\*\* the transaction slot and exact pre/post/delta amounts, plus the\
current raw balance, decimals, display string, and observation slot.\
\`read_covers_transaction: true\` means the balance read was pinned to at least\
the transaction slot. It does \*\*not\*\* mean MoonX or\
[\`get_portfolio\`]\(#get_portfolio) has indexed the transaction.\
\
This tool is read-only and requires authentication. It fails closed when the\
Alchemy key is missing, the RPC remains behind the transaction slot, the\
transaction failed, or the wallet/mint pair was not affected.\
\
\*\*\*\
\
\## get\\\_buy\\\_link\
\
Generate a \*\*signed MoonPay checkout URL\*\* that buys crypto with fiat\
(card/bank) straight into a \*\*wallet\*\* credential — the way to fund a wallet\
that has no balance yet. The buyer opens the link in a browser and completes\
the purchase on MoonPay's page (payment + KYC happen there); the purchased\
crypto is delivered to the wallet's address, pre-filled from the credential.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A \`wallet\`-kind credential from \`list_credentials\`. Its captured \`address\` becomes the purchase destination. |\
\| \`destination_chain\` | string? | Exact destination. Omit for Base (\`eip155:8453\`) on EVM wallets or Solana mainnet on Solana wallets. Ethereum mainnet requires explicit \`eip155:1\`. |\
\| \`currency_code\` | string? | MoonPay code enabled for this integration and bound by MoonPay's catalog to the exact destination. Omit for \`usdc_base\` on Base, \`eth\` on explicit Ethereum mainnet, or native SOL on Solana. On Solana, USDC is \`usdc_sol\` (mint \`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v\`). Bare \`usdc\` is canonicalized to \`usdc_base\` only when the destination resolves to Base. |\
\| \`amount_usd\` | number? | Fiat USD amount to pre-fill (e.g. \`50\` = $50). Omit to let the buyer pick at checkout. |\
\
\*\*Returns\*\* the finished link synchronously (no result envelope — there is\
nothing to poll):\
\
\`\`\`json\
{\
&#x20; "url": "https\://buy.moonpay.com?apiKey=…&currencyCode=usdc_sol&walletAddress=…&signature=…",\
&#x20; "currency_code": "usdc_sol",\
&#x20; "wallet_address": "5EUa…SViS",\
&#x20; "network": "solana:mainnet",\
&#x20; "message": "Share this MoonPay checkout url with the user…"\
}\
\`\`\`\
\
\* The URL is \*\*signed server-side\*\* (HMAC over the query string), so the\
&#x20; pre-filled destination can't be tampered with — MoonPay refuses to load an\
&#x20; altered link. The signing key never leaves the server.\
\* Generating a link \*\*moves no money and needs no approval\*\* — it only points a\
&#x20; purchase at the user's own wallet. The payment itself happens on MoonPay's\
&#x20; page, by the human.\
\* A disabled code or a code that settles on a different exact network is\
&#x20; rejected before signing. If USDC on Base is unavailable, no link is returned\
&#x20; and the caller must not retry another network unless the user named it.\
\* On a deployment without buy-link support the tool returns an \`error\`.\
\
:::tip\
After the user reports the purchase complete, use\
\*\*[\`verify_solana_balance\`]\(#verify_solana_balance)\*\* when the completed Solana\
checkout gives you a transaction signature. Otherwise confirm arrival with\
\*\*[\`get_portfolio\`]\(#get_portfolio)\*\*. Card purchases can take a few minutes to\
settle on-chain.\
:::\
\
\*\*\*\
\
\## discover\\\_services\
\
Search the curated x402 Bazaar set for paid services. By default, PayBox returns\
services with recent usage, at least 10 unique payers in the last 30 days, and a\
supported x402 rail. This is read-only; it does not make a payment.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`query\` | string? | Search query, e.g. \`"weather api"\`. Omit for the curated listing. |\
\| \`raw\` | boolean? | Debug escape hatch. \`false\` by default; \`true\` returns the uncurated Bazaar catalog/search response. |\
\
\*\*Returns\*\* service summaries with resource URLs, descriptions, and accepted\
payment requirements. Pick a service, then pay it with [\`pay_x402\`]\(#pay_x402)\
or let PayBox fetch it with [\`use_service\`]\(#use_service).\
\
Hidden on deployments where x402 is disabled.\
\
\*\*\*\
\
\## pay\\\_x402\
\
Pay an x402 endpoint from a \*\*wallet\*\* credential and return the payment header\
to the agent. Use this when the agent or client wants to retry the resource\
itself.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A wallet-kind credential from \`list_credentials\`. |\
\| \`accepts\` | array | The 402's \`accepts\` PaymentRequirements array, verbatim. |\
\| \`resource_url\` | string | The paid resource URL, used for audit and display. |\
\| \`x402_version\` | integer? | \`1\` for JSON-body requirements, \`2\` for \`PAYMENT-REQUIRED\` header requirements. Defaults to \`1\`. |\
\| \`resource\` | object? | x402 v2 resource block from \`PAYMENT-REQUIRED\`; omit for v1. |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope). On success,\
\`output.value.x_payment\` has the header name and value the agent can send to the\
paid resource.\
\
:::warning\
x402 requires a plain EOA payer. If the wallet is still EIP-7702 delegated, the\
tool either returns a plain-EOA error or — where auto-undelegate is enabled for\
the chain — runs one composite signing flow that undelegates first, verifies the\
wallet is plain, and only then produces the x402 payment.\
:::\
\
Hidden on deployments where x402 is disabled.\
\
\*\*\*\
\
\## use\\\_service\
\
Fetch a paid x402 resource through PayBox gateway mode. PayBox probes the\
resource, parses the 402, opens the signing window, then re-fetches the resource\
with the payment header and stores the response in the request output.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`credential_id\` | string | A wallet-kind credential from \`list_credentials\`. |\
\| \`url\` | string | Paid x402 resource URL. |\
\| \`method\` | string? | HTTP method. Defaults to \`GET\`. |\
\| \`body\` | object? | JSON body sent only on the paid re-fetch, never on the unpaid 402 probe. |\
\| \`headers\` | object? | Custom headers forwarded on the 402 probe and paid re-fetch. |\
\
\*\*Returns\*\* the [result envelope]\(#the-result-envelope). On success,\
\`output.value.response\` contains \`{ status, content_type, body }\`.\
\
Use this when the agent just wants the paid content. Use\
[\`pay_x402\`]\(#pay_x402) when the agent needs the payment header and will call the\
resource itself.\
\
Exposed only on deployments with x402 gateway mode enabled; check \`tools/list\`.\
\
\*\*\*\
\
\## get\\\_request\
\
Poll a previously-issued request. This is how every \`pending\_\*\` operation\
reaches its result — the agent never calls the internal completion tools.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`request_id\` | string | From a prior \`request\_\*\` result. |\
\
\*\*Returns\*\* the same [result envelope]\(#the-result-envelope) shape, reflecting\
the latest state:\
\
\* \`pending_approval\` → still waiting on the user's passkey.\
\* \`pending_signature\` → approved/autonomous; the signing window is finishing it.\
\* \`success\` → done; \`output\` holds the artifact (signature, tx hash, token, …).\
\* \`denied\` → includes \`reason\`.\
\
Only the client that minted a request can read it. Polling is idempotent.\
\
\*\*\*\
\
\## list\\\_requests\
\
List this client's own recent requests, newest first — the agent's\
request/trade history. Read-only: no funds move, nothing is signed, and it\
returns \*\*only requests this client created\*\*, never another client's.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`status\` | string? | Filter by status (snake\\\_case, e.g. \`success\`, \`denied\`, \`pending_approval\`). Omit for every status. |\
\| \`limit\` | integer? | Max rows, newest first. Default 200, capped at 500. |\
\| \`before\` | string? | Keyset cursor, first half: the last row's \`created_at\` (RFC 3339). Pair with \`before_id\`. |\
\| \`before_id\` | string? | Keyset cursor, second half: the last row's \`request_id\`. The pair forms a total cursor, so rows sharing a timestamp are never skipped. |\
\
\*\*Returns\*\* rows carrying \`request_id\`, \`kind\` (\`payment\` | \`wallet_sign\` |\
\`secret\` | \`swap\` | …), \`status\`, \`created_at\` + \`completed_at\`, a \*\*redacted\*\*\
\`output\` (present only on \`success\`), \`approval_id\`, and \`error\`.\
\
\* Outputs are redacted exactly like \`get_request\` polls — to obtain the actual\
&#x20; artifact of one request, call [\`get_request\`]\(#get_request) with its\
&#x20; \`request_id\`.\
\* To page, pass \*\*both\*\* \`before\` and \`before_id\` from the last row of the\
&#x20; previous page.\
\* \`pending\_\*\` rows reflect the last stored status and may lag a request's true\
&#x20; terminal state; \`get_request\` gives the live answer.\
\
\*\*\*\
\
\## request\\\_account\\\_change\
\
Ask the \*\*user\*\* to update this connector's access — grant another credential,\
raise a limit, change an approval mode, or create a wallet/card/secret they\
don't have yet. Call it when an operation is blocked by policy or a missing\
grant.\
\
\| Field | Type | Notes |\
\| --- | --- | --- |\
\| \`note\` | string? | One short sentence describing what needs to change (e.g. \`"needs an EVM wallet to sign"\`). Shown verbatim to the user so they know why the window opened. |\
\
\*\*Returns\*\* synchronously (nothing is parked — the agent only ever receives a\
link, never the user's vault):\
\
\`\`\`json\
{\
&#x20; "kind": "account_change",\
&#x20; "client_id": "…",\
&#x20; "manage_url": "https\://app.paybox.sh/clients/…/manage?note=…",\
&#x20; "note": "needs an EVM wallet to sign"\
}\
\`\`\`\
\
Surface \`manage_url\` and ask the user to open it. They make the change in their\
own passkey-gated PayBox session; nothing is applied on the agent's behalf.\
Don't claim the change is done until the user confirms or the retried operation\
succeeds.\
\
\*\*\*\
\
\## Plugin tools\
\
Beyond the core tools above, PayBox has \*\*first-party plugins\*\* — batches of\
extra tools (e.g. prediction-market or spot-market access) a user switches on\
from the app's \*\*Plugins\*\* screen.\
\
You don't need a separate per-plugin reference — \*\*discover them at runtime\*\*:\
\
\* Users turn plugins on in the PayBox app at\
&#x20; [\`https\://app.paybox.sh/plugins\`]\(https\://app.paybox.sh/plugins). That is\
&#x20; not a host Settings → Plugins screen.\
\* Call \*\*\`discover_plugins\`\*\* to find plugins at runtime. Omit \`query\` to\
&#x20; browse a page (default 10, max 25). Set \`query\` to the plugin or job the\
&#x20; user named. Set \`status: enabled\` to see what this account has turned on.\
&#x20; Page with \`next_cursor\` only while the current page has no suitable match.\
&#x20; Do not exhaust the catalog into Welcome. After setup, point at the Plugins\
&#x20; page, then suggest a few concrete moves from the enabled set (you can\
&#x20; combine two enabled plugins in one suggestion). For an official result,\
&#x20; pass its \`contract_uri\` unchanged to \*\*\`get_contract\`\*\*, then invoke the\
&#x20; selected tool with \*\*\`use_plugin\`\*\* (\`plugin_id\`, \`tool_id\`, \`input\`\
&#x20; matching that contract). Do not invent CLI flags for a plugin that has no\
&#x20; \`paybox \<plugin>\` group.\
\* Enabled plugin tools also appear in MCP \*\*\`tools/list\`\*\* (with full input\
&#x20; schemas). List tools after connecting, and again when the user says they've\
&#x20; changed their plugins.\
\* Calling a plugin tool the user hasn't enabled returns an error naming the\
&#x20; plugin; point the user at the app's Plugins screen (or\
&#x20; [\`request_account_change\`]\(#request_account_change)) and retry after they\
&#x20; enable it.\
\* Plugin tools follow all the same rules as the core surface: the\
&#x20; [result envelope]\(#the-result-envelope), the poll pattern, and passkey\
&#x20; approvals — plugin \*\*money tools always pause for the user's approval\*\*, even\
&#x20; under an autonomous grant.\
\
\*\*\*\
\
\## Errors\
\
Calls to \`/mcp\` without a valid bearer token return \*\*\`401\`\*\* with a\
\`WWW-Authenticate\` header pointing at discovery, so the client can re-authorize.\
That is an auth failure, not a down server. If a host labels this\
\`unreachable\` while the CLI \`whoami\` against the same origin still works, say\
the connector is unreachable and do not report money as sent:\
\
\`\`\`http\
HTTP/1.1 401 Unauthorized\
WWW-Authenticate: Bearer resource_metadata="https\://api.paybox.sh/.well-known/oauth-protected-resource", error="invalid_token"\
\`\`\`\
\
Beyond auth, problems surface in the result envelope: \`denied\` (a policy or user\
decision, with a \`reason\`) versus \`error\` (an internal failure, with a\
\`message\`). See \*\*[OAuth 2.1 → Errors]\(/connect/oauth#errors)\*\* for token-exchange\
error codes.\
\
\
\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# API & endpoints\
\
PayBox's developer surface is small on purpose: \*\*OAuth 2.1\*\* to authorize, then\
\*\*MCP\*\* to act. Connecting wallets and credentials, client management, approvals, and audit are\
done by the user in the PayBox app — they are not a public API.\
\
\## Origins\
\
\| Role | URL |\
\| --- | --- |\
\| API origin (OAuth issuer + MCP) | \`https\://api.paybox.sh\` |\
\| App / consent UI | \`https\://app.paybox.sh\` |\
\
The OAuth \*\*issuer\*\* and the MCP \*\*resource\*\* derive from the API origin: the\
resource is \`\<origin>/mcp\`. Always start from discovery rather than hard-coding\
paths.\
\
\## Public endpoints\
\
\| Endpoint | Method | Auth | Purpose |\
\| --- | --- | --- | --- |\
\| \`/.well-known/oauth-authorization-server\` | GET | none | OAuth 2.1 server metadata (RFC 8414). |\
\| \`/.well-known/oauth-protected-resource\` | GET | none | Protected-resource metadata (RFC 9728). |\
\| \`/oauth/register\` | POST | none | Dynamic client registration (RFC 7591). |\
\| \`/oauth/authorize\` | GET | user (browser) | Authorization request (code + PKCE S256). |\
\| \`/oauth/token\` | POST | none (public client) | Exchange code or refresh token for an access token. |\
\| \`/mcp\` | POST | Bearer access token | The MCP endpoint — all agent operations. |\
\
See \*\*[OAuth 2.1]\(/connect/oauth)\*\* for request/response shapes, scopes, token\
TTLs, and error codes, and \*\*[MCP tools]\(/reference/mcp-tools)\*\* for everything an\
agent can call on \`/mcp\`.\
\
\## Authentication\
\
Agents authenticate to \`/mcp\` with a \*\*bearer access token\*\* obtained via the\
OAuth flow:\
\
\`\`\`http\
POST /mcp\
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9…\
\`\`\`\
\
The token is \*\*audience-bound\*\* to the MCP resource and carries the \*\*grant set\*\*\
the user approved — which credentials, and the approval mode for each. It never\
overrides a passkey step-up: sensitive operations still pause for the user. See\
\*\*[Approvals & passkeys]\(/concepts/approvals)\*\*.\
\
A personal \*\*\`pbx_live\_\*\` API key\*\* (issued from the app's Clients screen) works\
as the same bearer — both resolve to the same scoped client. Keys suit headless\
use (CI, servers) where the OAuth browser flow doesn't; the SDK and CLI accept\
either.\
\
\## Not a public API\
\
The first-party PayBox app (web + mobile) uses internal \`/api/\*\` routes for\
sign-in, vaulting credentials, managing clients, reviewing approvals, and reading\
the audit log. These are \*\*not part of the developer surface\*\* and are not\
documented here — an integration never calls them. Everything an external agent\
needs is the OAuth flow plus the MCP tools above.\<!--\
Sitemap:\
\- [PayBox]\(/index): The non-custodial wallet for AI agents\
\- [Getting started]\(/getting-started)\
\- [Credentials & agents]\(/concepts/model): The PayBox model\
\- [Approvals]\(/concepts/approvals)\
\- [Request lifecycle]\(/concepts/requests): From intent to result\
\- [MCP connector]\(/connect/mcp)\
\- [OAuth 2.1]\(/connect/oauth)\
\- [MCP tools]\(/reference/mcp-tools): What an agent can call\
\- [API & endpoints]\(/api-reference)\
\- [SDK & CLI]\(/sdk-cli)\
\- [PayBox integration manual]\(/skill): Connect an AI agent to PayBox: CLI on Muse and Grok Bot, MCP on connector hosts.\
\-->\
\
\# SDK & CLI\
\
\`@paybox-sh/sdk\` is a typed Node SDK and \`paybox\` CLI for the PayBox agent API.\
It wraps the core MCP tool surface — \`list_credentials\`, \`request_payment\`,\
\`claim_payment_credentials\`, \`request_wallet_sign\`, \`request_secret\`,\
\`request_swap\`, \`get_portfolio\`, \`verify_solana_balance\`, \`get_buy_link\`,\
\`request_account_change\`, the x402 tools (\`discover_services\`, \`pay_x402\`,\
\`use_service\`), \`get_request\`, and\
the generic [plugin discovery and invocation flow]\(/reference/mcp-tools#plugin-tools).\
Core operations use the REST API; plugins use the same MCP tools as connector\
hosts, with no hardcoded per-plugin paths.\
Wallet signing is non-custodial and runs in-process: the SDK reuses the\
wallet-sign signing core and signs the MoonX envelope with your \`pbxk1.\` key;\
the MoonX secret never reaches the client.\
\
:::note\
A few surfaces are \*\*MCP-only today\*\*: geofenced World money tools; request history\
([\`list_requests\`]\(/reference/mcp-tools#list_requests)) has no SDK method or CLI\
command, and the wrappers don't expose the advanced x402 options (the v2\
\`x402_version\` + \`resource\` fields on \`pay_x402\`; custom \`headers\` and probe\
mode on \`use_service\`). Use the [MCP connector]\(/connect/mcp) for those.\
:::\
\
The request/response types are generated from the server's MCP \`list_tools\`\
schemas, so the SDK stays in sync with the API rather than being hand-maintained.\
\
\## Install\
\
Use the CLI when the agent runs on \*\*Muse\*\*, \*\*Grok Bot\*\*, \*\*Grok Computer\*\*, or any\
computer/VM with no connector screen. Use the [MCP connector]\(/connect/mcp) when\
the host has one (Claude.ai, Grok at \`grok.com\`, ChatGPT web).\
\
\`\`\`bash\
pnpm add @paybox-sh/sdk      # library\
npx @paybox-sh/sdk login     # or use the CLI directly\
\`\`\`\
\
\## Library\
\
\`\`\`ts\
import { PayboxClient } from "@paybox-sh/sdk";\
\
// Reads \~/.config/paybox + env (PAYBOX_API_KEY / PAYBOX_API_URL), or pass\
// { apiKey, signingKey } / { token } explicitly.\
const paybox = PayboxClient.fromConfig();\
\
const { credentials, ungranted_summary } = await paybox.listCredentials();\
const pay = await paybox.requestPayment({\
&#x20; credentialId: credentials[0].credential.id,\
&#x20; merchant: "Acme",\
&#x20; merchantUrl: "https\://acme.com",\
&#x20; amountCents: 1999,\
});\
\
// Count-only hints; no ungranted credential IDs, names, addresses, or metadata.\
console.log(ungranted_summary.wallet.evm);\
\
const signed = await paybox.requestWalletSign({\
&#x20; credentialId: walletId,\
&#x20; intent: { op: "message", message: "gm" },\
}); // signs in-process when a pbxk1. signing key is configured\
\
const verified = await paybox.verifySolanaBalance({\
&#x20; address: "5EUa…SViS",\
&#x20; tokenMint: "EPjF…Dt1v",\
&#x20; transactionSignature: "5h6x…q2Ms",\
});\
console.log(verified.balance.ui_amount_string);\
\
// Approval-mode operations can wait for the user's passkey approval and then\
// sign in-process — the headless equivalent of the in-chat signing window.\
const swap = await paybox.requestSwap(\
&#x20; {\
&#x20;   credentialId: walletId,\
&#x20;   srcChain: "eip155:8453",\
&#x20;   srcToken: "native",\
&#x20;   dstToken: "0x…",\
&#x20;   amount: "1000000000000000",\
&#x20; },\
&#x20; { waitForApproval: true },\
);\
\`\`\`\
\
\## CLI\
\
\`\`\`bash\
paybox login                              # OAuth 2.1 device code\
paybox login --key pbx_live\_…             # personal API key instead (no OAuth)\
paybox connect                            # pair a signing key with an eight-emoji code\
paybox login --no-open                    # print the link without opening a browser\
\
paybox credentials                        # list usable credentials\
paybox pay --credential \<id> --merchant Acme --url https\://acme.com --amount 1999\
paybox claim \<request_id>                 # claim an approved payment's one-time card\
paybox secret --credential \<id> --purpose "deploy"\
paybox sign --credential \<id> --intent '{"op":"message","message":"gm"}'\
paybox swap --credential \<id> --src-chain eip155:8453 --src-token native --dst-token 0x… --amount 1000000000000000\
paybox portfolio --address 0x… --networks 1,8453\
paybox verify-solana-balance --address 5EUa…SViS --mint EPjF…Dt1v --transaction 5h6x…q2Ms\
paybox buy-link --credential \<id> --amount-usd 50   # MoonPay checkout URL to fund a wallet\
\
paybox discover "weather api"             # browse curated paid x402 services\
paybox pay-x402 --credential \<id> --url https\://svc/api --accepts @accepts.json\
paybox use-service --credential \<id> --url https\://svc/api   # paybox pays + fetches\
\
paybox account-change --note "need a wallet grant"  # link for the user to update access\
paybox request \<request_id> --wait        # poll a pending request\
\
paybox plugins list --status enabled\
paybox plugins contract paybox://plugins/world\
paybox plugins use world world_get_events --input '{"status":"active"}'\
\
paybox version                            # installed version (+ update check)\
paybox update                             # update the global install to latest\
paybox uninstall                          # clear stored config (prints npm rm)\
\
paybox --json credentials                 # machine-readable output on any command\
\`\`\`\
\
Plugin commands are discovered at runtime rather than compiled into one command\
group per plugin. Run \`paybox plugins list\`, pass an official result's exact\
\`contract_uri\` to \`paybox plugins contract\`, then invoke the selected schema with\
\`paybox plugins use\`. Unofficial discovery results include their tools directly.\
\
The \`paybox world\` and \`paybox hyperliquid\` command groups were removed in\
1.0.0. Reach both through \`paybox plugins\`, the same way as any other plugin.\
\
\### Logging in\
\
Setup is two commands:\
\
1\. \*\*\`paybox login\`\*\* starts an OAuth 2.1 \*\*device-code\*\* grant. The CLI prints a\
&#x20;  verification URL, tries to open it locally, and polls until you choose your\
&#x20;  wallets and approve. Because it's device-code, the browser that approves\
&#x20;  doesn't have to be on the machine running the CLI: you can open the printed\
&#x20;  URL on your phone, or on a laptop while the CLI runs on a server. A scoped,\
&#x20;  audience-bound token (and a refresh token) is saved to\
&#x20;  \`\~/.config/paybox/config.json\` (mode \`0600\`). If the code expires before you\
&#x20;  approve, run \`paybox login\` again for a fresh one.\
2\. \*\*\`paybox connect\`\*\* pairs a signing key, which \`sign\` and \`swap\` need. The\
&#x20;  CLI generates the key on this machine, prints eight emoji and a link, and\
&#x20;  polls. Open the link on any device and pick the row with the same emoji. The\
&#x20;  key stays on this machine and PayBox never holds it; there is nothing to copy\
&#x20;  or paste.\
\
Add \`--no-open\` to either command to print the link without launching a browser.\
\`login\` clears a stored signing key, because a key paired to the previous client\
cannot sign, so run \`connect\` after it.\
\
Running two environments from one machine? Point \`PAYBOX_CONFIG_DIR\` at a\
separate directory per environment. \`login\` rewrites the auth block of the config\
it's pointed at, so sharing one directory across environments will overwrite the\
credentials you already had there.\
\
\### When an agent is driving\
\
The section above assumes you are at the terminal. Often you are not: the CLI\
runs on a box, and you are in a chat with the agent. Four things change.\
\
\*\*The links are the agent's job.\*\* The agent runs \`paybox login --no-open\` and\
\`paybox connect --no-open\`, reads the printed links from stderr, and puts each\
one in front of you as something you can tap. Without \`--no-open\` the CLI tries\
to open a browser on its own machine, which is useless when the machine is not\
yours. If the agent does not relay a link, the command polls until it times out\
and nothing tells you why.\
\
\*\*Nothing else should reach the chat.\*\* From the install to a working session you\
need exactly two messages: the Connect CLI link, then the eight emoji with the\
signing-key link. Nothing is pasted, and the host stores nothing, so the agent\
never needs a custom connector, a Client ID, or a host secrets store. Progress\
narration in between is noise the agent is adding.\
\
\*\*Setup is not repeatable.\*\* If \`paybox --json whoami\` already reports\
\`canSign: true\`, the work is done. Running \`login\` again sends you a second\
authorize link, which reads as the first one having failed.\
\
\`@paybox-sh/sdk\` ships a \`SKILL.md\` with this same flow written as instructions\
for the agent rather than for you. It is inside the package, at\
\`node_modules/@paybox-sh/sdk/SKILL.md\` after an install, and on the package's\
[npm page]\(https\://www\.npmjs.com/package/@paybox-sh/sdk). Point your agent at it\
and it will not need any of this page.\
\
\### Signing and approvals\
\
\`sign\`, \`swap\`, \`pay-x402\`, and \`use-service\` complete in-process when a\
\`pbxk1.\` signing key is configured (\`paybox whoami\` shows \`canSign\`) and the\
operation clears immediately under an autonomous grant. Without a key those\
commands stop at \`pending_signature\`.\
\
When a grant requires approval, pass \`--wait\`: the command waits for your\
passkey approval in the app, signs in-process, and then keeps polling until the\
request reaches a terminal state — there is no signing window on this surface.\
Without \`--wait\` you get the first snapshot back instead, which may still be\
settling (\`pending_confirmation\`), so poll \`paybox request \<id>\` yourself before\
treating the operation as finished. Solana swaps are safe to approve at your own\
pace — the SDK refreshes the transaction's recent blockhash right before signing,\
so a swap that waited on approval doesn't expire.\
\
A word on reading results if you're scripting this: the exit code is honest. A\
terminal \`denied\` or \`error\` exits non-zero, and so does a request that could not\
be made. With \`--json\`, those failures also come back as a JSON object on stdout,\
so you can read one channel whichever way it went. Bad usage is the exception:\
a missing flag or an unknown command is reported as plain text on stderr before\
the command runs, so check the exit code there rather than parsing stdout.\
\
You can also integrate over the \*\*[API reference]\(/api-reference)\*\* directly, or\
connect an agent via the \*\*[MCP connector]\(/connect/mcp)\*\*.\
\
\
:::note\
A typed Node SDK and CLI ship over this surface — \`@paybox-sh/sdk\`, with\
in-process (iframe-free) wallet signing. See \*\*[SDK & CLI]\(/sdk-cli)\*\*.\
:::\
\
\
