### TL;DR

- Monid can expose a native CAPTCHA-solving connector through the same `discover`, `inspect`, and `run` flow that its agents use for other API tools.
- The proposed first release covers proxyless reCAPTCHA v2, reCAPTCHA v3, and Cloudflare Turnstile tasks for authorized automation workflows.
- The connector keeps the CapSolver API key inside Monid's credential injection layer instead of placing it in agent prompts, fixtures, or lifecycle state.
- A local prototype already compiles into the Monid catalog and passes the asynchronous `createTask → processing → ready` lifecycle test.
- The hosted Monid catalog does not include this connector yet. The code and commands below describe the tested implementation and intended user flow pending upstream review.

## Why AI Agents Need a CAPTCHA Tool in Monid

Browser agents can navigate pages and call web-data tools, but a CAPTCHA checkpoint creates a separate technical step. The agent must identify the challenge type, submit the correct page URL and public site key, wait for a result, and apply the returned token to the authorized browser session. Without a dedicated tool, every agent framework has to implement that lifecycle independently.

The proposed integration adds [CapSolver](https://www.capsolver.com/?utm_source=offcial&utm_medium=blog&utm_campaign=how-to-add-captcha-solving-to-ai-agents-with-monid) as a native Monid provider. Monid remains the tool-discovery and execution layer, while CapSolver handles supported CAPTCHA tasks through its documented API. This approach fits Monid's existing model: an agent discovers a tool, inspects its schema and pricing, and starts a structured run through one interface. Monid documents that flow in its <a href="https://docs.monid.ai/" rel="nofollow">official introduction</a>.

## What the Monid and CapSolver Integration Enables

The native connector gives an AI agent a structured way to request CAPTCHA solving without embedding provider-specific HTTP logic in every workflow.

The first connector scope contains three endpoints:

| Monid endpoint | CapSolver task type | Intended use |
| --- | --- | --- |
| `capsolver#solve/recaptcha-v2` | `ReCaptchaV2TaskProxyLess` | Standard and invisible reCAPTCHA v2 token tasks |
| `capsolver#solve/recaptcha-v3` | `ReCaptchaV3TaskProxyLess` | Score-based reCAPTCHA v3 token tasks with an action |
| `capsolver#solve/turnstile` | `AntiTurnstileTaskProxyLess` | Cloudflare Turnstile token tasks |

This small initial scope keeps the connector easy to review. Enterprise task types, proxy-based tasks, and browser-side token application can be added separately after the core API lifecycle is stable.

The integration is designed for QA, RPA, browser automation, and permitted public-web data workflows. Technical capability does not grant permission to access private, restricted, or sensitive data. Review the target site's terms and obtain authorization before automating it.

## How the Native Connector Works

The native connector wraps CapSolver's asynchronous task API in Monid's standard provider lifecycle.

1. The agent discovers or selects a CapSolver endpoint.
2. Monid validates `websiteURL`, `websiteKey`, and any task-specific input.
3. Monid injects the protected `clientKey` into the JSON request body.
4. The connector calls `/createTask` and stores only the returned task ID.
5. Monid polls `/getTaskResult` while the task is `idle` or `processing`.
6. When the task becomes `ready`, the connector returns the provider's structured solution.

This is intentionally a native REST connector rather than an MCP-to-MCP bridge. Monid already exposes its own <a href="https://docs.monid.ai/guide/quickstart-mcp.html" rel="nofollow">MCP interface for AI clients</a>; the provider layer underneath it calls CapSolver's API directly.

```text
AI agent
  → Monid discover / inspect / run
  → CapSolver connector
  → createTask
  → getTaskResult polling
  → structured solution
  → authorized browser workflow continues
```

> ### **Redeem Your CapSolver Bonus Code**
>
>Boost your automation budget instantly!
Use bonus code **CAP26** when topping up your CapSolver account to get an extra **5% bonus** on every recharge — with no limits.
Redeem it now in your [CapSolver Dashboard](https://dashboard.capsolver.com/dashboard/overview/?utm_source=offcial&utm_medium=blog&utm_campaign=how-to-add-captcha-solving-to-ai-agents-with-monid)
> ![Bonus Code](https://assets.capsolver.com/prod/posts/cloudflare-blocking-your-ai-agent-solution/uyhK7mNTRCcA-22664fd10ef4221d9ca40218ca5b7863.png)
> 

## Configure CapSolver as a Monid Provider

CapSolver authentication differs from providers that use an HTTP authorization header. The API expects `clientKey` in the JSON body, so the connector adds it inside Monid's credential injection layer.

The tested provider definition uses this pattern:

```typescript
export default defineProvider({
  name: "capsolver",
  auth: {
    inject: ({ data, utils }) => ({
      ...data.request,
      body: utils.json.merge(data.request.body ?? {}, {
        clientKey: data.params.apiKey,
      }),
    }),
  },
  request: {
    baseUrl: "https://api.capsolver.com",
    headers: { "Content-Type": "application/json" },
  },
  timeouts: { requestMs: 30_000, runMs: 300_000, pollMs: 3_000 },
});
```

The secret stays out of the endpoint definition, replay fixtures, logs, and lifecycle state. Only the transport layer receives the credential when it sends the provider request.

You can review the current CapSolver task lifecycle in the official [`createTask`](https://docs.capsolver.com/en/guide/api-createtask/) and [`getTaskResult`](https://docs.capsolver.com/en/guide/api-gettaskresult/) documentation.

## Define a reCAPTCHA v2 Endpoint

Each Monid endpoint maps a validated input schema to one CapSolver task type. The reCAPTCHA v2 endpoint accepts a page URL, public site key, and optional fields for invisible widgets or page actions.

```typescript
const zBody = z.object({
  websiteURL: z.url(),
  websiteKey: z.string().min(1),
  isInvisible: z.boolean().optional(),
  pageAction: z.string().min(1).optional(),
}).strict();

export default defineEndpoint({
  endpoint: "/solve/recaptcha-v2",
  request: { method: "POST", path: "/createTask" },
  input: {
    schema: { body: zBody },
    toRequest: ({ data, utils }) => ({
      ...data.input,
      body: {
        task: utils.json.merge(data.input.body ?? {}, {
          type: "ReCaptchaV2TaskProxyLess",
        }),
      },
    }),
  },
});
```

The same structure applies to reCAPTCHA v3 and Turnstile, with task-specific schemas and task types. Monid's <a href="https://github.com/monid-ai/monid#writing-a-connector" rel="nofollow">connector authoring guide</a> requires replay fixtures, type checks, and a pull request before a provider is added to the hosted catalog.

## Handle the Asynchronous Result

The connector must treat CapSolver as an asynchronous provider. A successful `/createTask` response normally supplies a task ID, while `/getTaskResult` reports `idle`, `processing`, or `ready`.

The tested lifecycle follows these rules:

- `idle` or `processing`: keep the Monid run active and poll again.
- `ready`: return the structured solution and complete the run.
- non-zero `errorId`: return a provider error rather than a successful result.
- non-2xx HTTP response: preserve the provider response and record zero completed usage.
- missing task ID or an unknown status: stop with a clear connector error.

This separation matters because Monid distinguishes its own run status from the upstream provider's HTTP status. Its <a href="https://docs.monid.ai/cli/run.html" rel="nofollow">CLI run documentation</a> explains how asynchronous tools start a run and expose the final provider response.

## Inspect and Run the Connector After Publication

Once the connector is accepted into Monid's hosted catalog, users should inspect the published schema before sending a task. The exact command remains pending upstream publication, but the intended CLI flow is:

```bash
monid inspect -p capsolver -e /solve/recaptcha-v2

monid run -p capsolver -e /solve/recaptcha-v2 \
  -i '{"websiteURL":"https://authorized.example/test","websiteKey":"PUBLIC_SITE_KEY"}' \
  --wait
```

Use an authorized test page and replace `PUBLIC_SITE_KEY` with the widget's public site key. Keep API credentials in Monid's credential configuration rather than placing them in commands, prompts, or source files.

At the time of writing, these commands are a publication target, not a claim that the hosted endpoint is already available. The local prototype has passed connector compilation, catalog generation, and the synthetic `createTask → processing → ready` lifecycle test. A live paid task on an authorized demo page and Monid's upstream review remain prerequisites for general availability.

For direct agent integrations available today, the [CapSolver MCP Server](https://www.capsolver.com/agent-automation/mcp) exposes CAPTCHA tools to compatible clients. The broader [agentic browser integration guide](https://www.capsolver.com/blog/ai/agentic-browser-capsolver) explains where CAPTCHA handling belongs in a browser-agent workflow.

## Implementation Limits and Next Steps

The first connector should stay narrow until the basic lifecycle is accepted and observed in real workloads.

- It does not automatically detect a CAPTCHA in browser DOM content.
- It does not apply a returned token to an active Playwright or Puppeteer page.
- It does not include Enterprise or proxy-based CapSolver task types.
- It requires the calling workflow to provide the correct URL, public site key, and task-specific fields.
- It should not be used for unauthorized access, account abuse, or collection of private or sensitive information.

After upstream review, the next useful addition is a verified browser example showing how an agent detects a supported widget, calls the Monid endpoint, and applies the result to the same authorized session. That example should be tested against an explicit demo environment rather than a third-party production site.

## Conclusion

A native Monid connector is the cleanest way to make CAPTCHA solving discoverable to AI agents without adding provider-specific code to every agent framework. The local prototype demonstrates that Monid can validate task input, protect the API key, manage asynchronous polling, and return a structured result through its normal tool lifecycle.

The remaining work is procedural and verification-focused: confirm the category with Monid, complete one authorized live test, submit the connector for review, and publish reproducible commands only after the endpoint appears in the hosted catalog. Teams that need direct access before then can integrate through the official [CapSolver](https://www.capsolver.com/?utm_source=offcial&utm_medium=blog&utm_campaign=how-to-add-captcha-solving-to-ai-agents-with-monid) API or MCP Server.

## Build Reliable Automation Workflows with CapSolver

Create an account from the [CapSolver Dashboard](https://dashboard.capsolver.com/dashboard/overview/?utm_source=offcial&utm_medium=blog&utm_campaign=how-to-add-captcha-solving-to-ai-agents-with-monid), review the [supported integration documentation](https://docs.capsolver.com/en/), and test only on pages you are permitted to automate. For product and account questions, consult the [CapSolver FAQ](https://www.capsolver.com/faq).

## FAQ

**Q: Is the CapSolver connector already available in Monid?**

No. The native connector is currently a tested local prototype awaiting Monid's category confirmation, upstream review, and catalog publication.

**Q: Does Monid call the CapSolver MCP Server directly?**

No. Monid should call CapSolver's REST API through a native provider connector, while Monid's own MCP server exposes the resulting tools to compatible AI clients.

**Q: Which CAPTCHA types are included in the proposed first release?**

The proposed first release covers proxyless reCAPTCHA v2, reCAPTCHA v3, and Cloudflare Turnstile token tasks.

**Q: Where is the CapSolver API key stored?**

The connector passes the API key through Monid's credential injection layer and adds it to the provider request body only when the request is sent.

**Q: Can the connector automatically control a browser page?**

No. The initial connector returns a structured solution; the authorized browser workflow remains responsible for applying that result to its active page or session.

**Q: Can the integration be used on any website?**

No. Use the integration only on sites and workflows you are authorized to automate, and follow applicable terms, privacy requirements, and laws.


