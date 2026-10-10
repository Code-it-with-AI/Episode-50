# Episode-50: MCP 2 Dialogs with Multi Round-Trip Requests

Rocky shows how an MCP server can ask follow-up questions in the middle of a tool call, using the new multi round-trip requests (MRTR) feature in the MCP `2026-07-28` spec.

📺 YouTube video: _(link coming soon)_

🏠 Code it with AI Home Page: https://codeitwithai.com

## The goal

Real tools rarely get everything they need in the first call. A shipping quote needs a destination, a weight, and a delivery speed. If the package is heavy and going overnight, the user also has to accept a surcharge. The follow-up questions depend on the answers that came before them.

We wanted an MCP server that can say "I need more information before I can answer," get those details from the user, and then finish the job. We also wanted it to work on a modern, **stateless** MCP server that can sit behind a load balancer with no sticky sessions.

## The solution

The `2026-07-28` MCP spec adds **multi round-trip requests (MRTR)**, and the MCP C# SDK 2.x supports them. The sample works like this:

1. The agent calls the `get_shipping_quote` tool with whatever it already knows, which might be nothing.
2. If the tool is missing information, it returns an `input_required` result instead of a final answer. That result contains one or more questions (elicitations) and an opaque `requestState` token.
3. The client SDK asks the user those questions, then re-sends the **same** tool call with the answers and the `requestState` it received. All of this happens inside one `CallToolAsync` call, so the agent code never sees the intermediate rounds.
4. The tool runs again, reads the answers, and either asks the next question or returns the quote. Each round can ask different questions based on earlier answers.

### No sessions required

The `2026-07-28` spec removes `Mcp-Session-Id` and the `initialize` handshake from Streamable HTTP. So how does the server remember the conversation between rounds?

**`requestState` is the correlation mechanism.** The server puts the dialog state (dialog id, round number, answers so far) into that token and encrypts it with ASP.NET Core Data Protection, with a 10-minute expiration. The client can't read it or forge it, and the server keeps nothing in memory. Any server instance can handle any round.

### Seeing it work

- The agent is a Spectre.Console CLI. Its `--wire` mode shows every JSON-RPC message, including the `input_required` responses and the retries.
- An **Aspire AppHost** runs the server and the Aspire Dashboard. The whole multi-round dialog appears as **one distributed trace**, with spans for each round on both the client and the server.
- The sample also covers fallbacks: clients that don't support MRTR, tampered or expired state, users who cancel, and out-of-range answers that get re-asked.

## Tech stack

- .NET 10
- [MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) 2.2 (`ModelContextProtocol`, `ModelContextProtocol.AspNetCore`)
- ASP.NET Core Data Protection
- Spectre.Console
- Serilog and OpenTelemetry
- Aspire 13.6

## Resource Links

- **Sample code (Mcp2-Dialog):** https://github.com/MarimerLLC/Mcp2-Dialog
- **MCP C# SDK:** https://github.com/modelcontextprotocol/csharp-sdk
- **Model Context Protocol:** https://modelcontextprotocol.io
- **About Rocky Lhotka:** https://about.me/rockfordlhotka
