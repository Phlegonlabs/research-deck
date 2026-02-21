# Sherwin Wu — OpenAI's API Engineering Leader & The "Sorcerer" Thesis

## 1. Who is Sherwin Wu

### Background

Sherwin Wu is the Head of Engineering for OpenAI's API and Developer Platform — the infrastructure layer that powers every third-party application built on OpenAI's models. He is an early OpenAI team member who has risen from Member of Technical Staff to leading the entire API engineering organization.

**Education:** BS & MEng in Computer Science from MIT (2010-2014).

**Career Path:**

| Period | Company | Role | Key Work |
|--------|---------|------|----------|
| ~2014 | Palantir Technologies | Engineer | Early career (internship/short stint) |
| ~2014 | Doximity | Engineer | Healthcare tech |
| 2014-2016 | Quora | Software Engineer | News Feed Ranking & Product Teams |
| 2016-2022 | Opendoor | Product Lead → Engineering Manager | 5+ years on Pricing Team — ML models for residential real estate pricing. "A single wrong prediction could cost millions." |
| ~2022-present | OpenAI | MTS → Head of Engineering, API Platform | Developer Platform team, grew from IC to org leader |

**Side Roles:**
- **Sancus Ventures** — listed on team page (likely advisor/LP role)
- Active on X/Twitter (@sherwinwu) sharing OpenAI developer insights

### Why He Matters

Wu sits at the intersection of OpenAI's research and its developer ecosystem. His team builds the API, Responses API, Agents SDK, AgentKit, and the developer tools that 15M+ developers use. He has a unique vantage point: he sees what developers are building, what patterns work, and where models are heading — all simultaneously. Martin Casado (a16z GP) called him "one of the best systems thinkers in AI."

---

## 2. Content Catalog (Chronological)

### 2.1 Tiktoken Model-to-Encoding Mapping Tweet (Feb 2023)

**Source:** X/Twitter — [@sherwinwu](https://x.com/sherwinwu/status/1621918059203072001)
**Date:** ~February 2023

Wu announced a developer quality-of-life improvement for tiktoken:

> "A pretty big quality-of-life improvement for all users of `tiktoken` — we just added a mapping from model name to tokenizer encoding! It's been hard to track the mapping of OpenAI API model to encoding — now you can do it programmatically."

He also promoted the open-source tokenizer as "3-6x faster than other open source alternatives."

**Takeaway:** Wu's attention to developer experience details — even tokenizer UX — reflects his product-engineering mindset.

---

### 2.2 OpenAI Cookbook Trending Tweet (~early 2023)

**Source:** X/Twitter
**Date:** Early 2023

> "The OpenAI cookbook is the top trending repo on Github this month! If you haven't checked it out, it's an incredible resource for getting started on the API — maintained by our team (and others!)"

**Takeaway:** Wu's team actively maintains developer resources and treats the cookbook as a first-class product.

---

### 2.3 QCon New York 2023 — "A Bicycle for the (AI) Mind: GPT-4 + Tools" (June 2023)

**Source:** [InfoQ Presentation](https://www.infoq.com/presentations/bicycle-ai-gpt-4-tools/) | [QCon Speaker Page](https://qconnewyork.com/speakers/sherwinwu)
**Date:** June 14, 2023
**Co-presenter:** Atty Eleti (OpenAI engineer, ex-Stripe)

**Core Thesis:** If Steve Jobs called the computer "a bicycle for the mind," then function calling is "a bicycle for the AI mind" — it extends LLMs from isolated knowledge boxes into tools that can interact with the real world.

**Key Technical Insights:**

1. **LLM Limitations Identified:**
   - Training data is fixed and outdated
   - No access to real-time information
   - Probabilistic output makes API integration unreliable (70-80% accuracy for structured JSON)
   - Models hallucinate and sometimes add explanatory text before JSON, breaking parsers

2. **Function Calling as Solution:**
   - Three-step loop: user query → model constructs function call → developer executes → model summarizes result
   - Fine-tuned on "hundreds of thousands" of examples to improve reliability
   - GPT-4 performs at 80th-90th percentile on professional exams "despite just predicting next tokens"

3. **Three Demos:**
   - **Natural Language to SQL** — converting business questions into executable queries
   - **Multiple External APIs** — combining location + Yelp APIs for restaurant recommendations
   - **Code Review Automation** — GPT reasoning applied to professional code review with customizable personality

4. **Limitations Acknowledged:**
   - Models remain hallucination-prone even with function calling
   - No native parallel function invocation (workaround: wrapper functions)
   - Non-English output can occur despite English prompts

**Takeaway:** This talk was the public debut of function calling — the primitive that later evolved into tool use, then Structured Outputs, then the Agents SDK. Wu was there at the foundation.

---

### 2.4 QCon New York 2023 — Panel: "Navigating the Future: LLM in Production" (June 2023)

**Source:** [InfoQ Panel Summary](https://www.infoq.com/news/2023/06/qcon-ny-llm-panel/)
**Date:** June 2023

Wu as panelist shared several predictions:

> "A mix of use cases: calling out to APIs for 'closed' foundation models vs. running self-hosted open-source models."

On where to focus in ML pipelines:
> "The input and output of the pipeline: the input being the data needed to fine-tune models, and the output being careful evaluation."

On prompt engineering's future:
> "[It will remain essential] for at least five more years."

On his personal wish list:
> "More progress on multimodal models as well as improvements in AI safety."

**Takeaway:** By mid-2023, Wu was already positioning input data quality and evaluation as the critical bottlenecks — a view that has proven prescient with the rise of context engineering and evals.

---

### 2.5 GPT-4 Turbo with Vision GA Tweet (April 2024)

**Source:** [X/Twitter](https://x.com/sherwinwu/status/1777882109086159100)
**Date:** April 9, 2024

> "GPT-4 Turbo with Vision now out of preview. This new model is quite an upgrade from even the previous GPT-4 Turbo — excited to see what new frontiers people can push with this one!"

**Takeaway:** Wu actively promotes API capability milestones, functioning as OpenAI's developer-facing voice.

---

### 2.6 OpenAI o1 "Paradigm Shift" Tweet (September 2024)

**Source:** [X/Twitter](https://x.com/sherwinwu/status/1834374228529086555)
**Date:** September 12, 2024

> "OpenAI o1 is a paradigm shift not only on the research side, but also on the product side — we now need to build with the 'limitation' of a model that takes some time to think. Can't wait to see what kinds of new products are built on this via the API!"

**Takeaway:** This is a critical insight. Wu recognized that reasoning models fundamentally change the developer experience — latency becomes a feature, not a bug. Products built on o1+ must embrace "thinking time" as a design constraint.

---

### 2.7 BG2 Pod — "Inside OpenAI Enterprise" (September 2025)

**Source:** [Spotify](https://open.spotify.com/episode/0HnYWQc9retjyFVSEAmFmQ) | [Apple Podcasts](https://podcasts.apple.com/us/podcast/inside-openai-enterprise-forward-deployed-engineering/id1727278168?i=1000726402888) | [Chain of Thought Summary](https://podcasts.chainofthought.xyz/podcast-summaries/inside-openai-enterprise-forward-deployed-engineering-gpt-5-and-more-bg2-guest-interview)
**Date:** September 11, 2025 (~1h 9min)
**Co-guest:** Olivier Godement (Head of Product, OpenAI Platform)
**Host:** Apoorv Agrawal (Altimeter)

**Key Insights:**

**1. OpenAI's First Product Was the API, Not ChatGPT**
The API/platform is the primary vehicle for distributing AI benefits. ChatGPT came later as a consumer product.

**2. Forward Deployed Engineering (FDE) Model:**
OpenAI adopted Palantir's FDE model — engineers embed deeply with enterprise customers to build custom solutions.
> "Embed very deeply with customers and build things specific to their systems."

FDEs handle system design, API integrations, and connect models to enterprise tools — many of which lack clean APIs.

**3. Enterprise Case Studies:**

| Customer | Use Case | Key Detail |
|----------|----------|------------|
| **T-Mobile** | Automated text + voice customer support | Real-time API enables natural conversations; improvements informed GPT-5 model enhancements |
| **Amgen** | Drug development acceleration + regulatory documentation | Automating admin-heavy processes that delay medication releases |
| **Los Alamos** | National security research on air-gapped supercomputer | Custom on-premises deployment of o3 reasoning model on Venado supercomputer |

On the Los Alamos deployment:
> "We literally had to bring the weights of the model physically into their supercomputer."

**4. GPT-5 Development Philosophy:**
> "For GPT-5, equally important and impactful was the craft: the style, the tone, the behavior of the model."

GPT-5 is so good at instruction following that it creates a new problem:
> "It turns out when you give that to GPT-5, it's like oh my gosh, this person really wants to be concise. And so the response would be like one sentence."

**5. Reinforcement Fine-Tuning (RFT):**
RFT is superior to traditional supervised fine-tuning for enterprise. It enables organizations to develop specialized, domain-leading models using proprietary data.

**6. Long/Short Positions:**
- **Long:** eSports (Asian growth potential, cultural relevance among younger demographics)
- **Short:** AI tooling startups (eval platforms, vector stores — too competitive with rapidly evolving technology)

**Takeaway:** The FDE model is OpenAI's enterprise moat. They're not just selling an API — they're embedding engineers who learn customer problems and feed those insights back into model development. The T-Mobile → GPT-5 improvement loop is the clearest example.

---

### 2.8 GPT-5 Instruction Following PSA Tweet (August 2025)

**Source:** [X/Twitter](https://x.com/sherwinwu/status/1953520864592375933)
**Date:** August 7, 2025

> "PSA that — because GPT-5 is so strong at instruction following — you may need to tweak your 4o or Sonnet prompts because it'll be almost _too good_ at following them. Begging/threatening it to be concise might make it overly so, and just simply asking it is often enough now!"

**Takeaway:** Each model generation requires prompt re-calibration. The era of "prompt hacking" (begging, threatening, role-playing) is being replaced by straightforward instruction.

---

### 2.9 Latent Space — "Developers as the Distribution Layer of AGI" (October 2025)

**Source:** [Latent Space](https://www.latent.space/p/devday-2025)
**Date:** October 6, 2025 (OpenAI Dev Day 2025)
**Co-guest:** Christina Huang (Platform Experience, OpenAI)

**Key Quotes:**

On OpenAI's developer mission:
> "We really need to rely on developers and we need to open up our technology to the rest of the world so that they can partake for us to really fulfill our mission."

On MCP adoption:
Wu praised Anthropic's MCP as genuinely open, with Nick Cooper representing OpenAI on the MCP steering committee.

On the Responses API:
Wu positioned it as a potential industry standard beyond just MCP — solving problems multiple vendors now adopt.

On agent evals:
> "Agent evals is still a work in progress. I think we've like made maybe 10% of the progress that we need here."

On prompt optimization:
> "It's a really cool time right now in prompt optimization... prompting is not going to be dead."

On scale:
> OpenAI serves over **6 billion tokens per minute** and targets five nines (99.999%) uptime.

**Takeaway:** Wu's framing of "developers as the distribution layer of AGI" is the strategic thesis behind OpenAI's entire platform investment. They can't reach every use case through ChatGPT alone — they need developers to carry AGI into verticals OpenAI can't.

---

### 2.10 a16z Show — "How OpenAI Builds for 800 Million Weekly Users" (November 2025)

**Source:** [Spotify](https://open.spotify.com/episode/36DXaD9KdlWh2Ksn7OVAdc) | [Apple Podcasts](https://podcasts.apple.com/hn/podcast/how-openai-builds-for-800-million-weekly-users-model/id842818711?i=1000738780411) | [Podwise](https://podwise.ai/dashboard/episodes/6063995) | [01Cloud Analysis](https://engineering.01cloud.com/2025/12/22/how-openai-scales-to-800-million-weekly-active-users-insights-from-sherwin-wu-on-model-specialization-and-fine-tuning/)
**Date:** November 28, 2025 (~53 min)
**Host:** Martin Casado (a16z GP)

Martin Casado called Wu "one of the best systems thinkers in AI."

**Key Insights:**

**1. Death of the "God Model":**
> "Even within OpenAI, the thinking was that there would be one model that rules them all. It's definitely completely changed. It's coming increasingly clear that there will be room for a bunch of specialized models."

**2. Model Stickiness is Real:**
> "Users already know and care which model they're using."

Models are "anti-disintermediation technology" — they're hard to abstract away because users build relationships with specific models.

**3. Reinforcement Fine-Tuning as the Big Unlock:**
> "The big unlock that has happened recently is with reinforcement fine-tuning."

Companies have "giant treasure troves of data that they are sitting on." RFT lets them leverage this data to create domain-specific models.

**4. Usage-Based Pricing:**
Usage-based pricing remains optimal because it aligns with actual consumption patterns. Cost-plus pricing ensures responsible margins.

**5. Platform vs. Product Tension:**
> "We want ChatGPT as a first-party app, which is a great way to get 800 million WAUs or whatever now. A tenth of the globe, right?"

OpenAI explicitly manages the tension between its consumer product (ChatGPT) and its platform (API).

**6. Deterministic Agent Architecture:**
OpenAI favors "deterministic, node-based" agent workflows over free-roaming systems — especially for regulated industries requiring explainability.

**7. From Prompt Engineering to Context Design:**
Wu signaled a fundamental shift from "prompt engineering" toward "context design" — structuring data and context to guide model performance, not just crafting clever prompts.

**Takeaway:** This interview is the most comprehensive window into OpenAI's platform strategy. The shift from "one model" to "model portfolio" mirrors the broader industry trend. The RFT insight explains why OpenAI is investing heavily in fine-tuning APIs.

---

### 2.11 LinkedIn — "Forward Deployed Engineer" Post (January 2025)

**Source:** [LinkedIn Post](https://www.linkedin.com/posts/sherwinwu1_forward-deployed-engineer-openai-activity-7284751404848619522-xW8x)
**Date:** ~January 2025

Wu posted about OpenAI's push on Forward Deployed Engineering. Over the past 18 months, his team has been deploying OpenAI technology on the frontlines, inside real enterprises, alongside real teams. The FDE model — borrowed from Palantir — means OpenAI engineers embed directly with customers to solve implementation challenges.

**Takeaway:** This LinkedIn post preceded the BG2 podcast discussion of FDE by several months, showing it was already a core strategic initiative.

---

### 2.12 LinkedIn — "Engineering Manager, API Experience" Post (July 2023)

**Source:** [LinkedIn Post](https://www.linkedin.com/posts/sherwinwu1_engineering-manager-api-experience-activity-7082552188635402240-cUHA)
**Date:** ~July 2023

Wu posted about hiring for the API Experience team — indicating his early role was Engineering Manager level before becoming Head of Engineering.

---

### 2.13 Assistants API Quickstart Open Source Tweet (~2024)

**Source:** X/Twitter
**Date:** ~2024

> "We've open sourced a new quickstart to help you build with the Assistants API and Next.js. It comes with sample code for creating a chat interface with streaming, and using tools like function calling, code interpreter, and the new file search."

**Takeaway:** Wu's team treats developer onboarding as product — open-sourcing quickstarts, not just documentation.

---

### 2.14 Lenny's Podcast — "Engineers are Becoming Sorcerers" (February 2026)

**Source:** [Lenny's Newsletter](https://www.lennysnewsletter.com/p/engineers-are-becoming-sorcerers) | [Spotify](https://open.spotify.com/episode/3EPDh5GqC9PEdyQBEpmdrQ) | [Apple Podcasts](https://podcasts.apple.com/us/podcast/engineers-are-becoming-sorcerers-the-future-of/id1627920305?i=1000749436380) | [Podwise](https://podwise.ai/dashboard/episodes/7171472)
**Date:** February 2026
**Host:** Lenny Rachitsky

This is Wu's most comprehensive and widely-shared interview. The title alone — "Engineers are becoming sorcerers" — has become a meme in the AI engineering community.

**Key Statistics:**

| Metric | Value |
|--------|-------|
| OpenAI engineers using Codex daily | 95% |
| Parallel AI agents per engineer | 10-20 |
| PRs reviewed by Codex | 100% |
| More PRs opened by Codex users vs non-users | 70% |
| Code review time reduction | 10-15 min → 2-3 min |

**Core Thesis — "Engineers are Sorcerers":**
> "Engineers are becoming tech leads. They're managing fleets and fleets of agents."

The metaphor references *The Sorcerer's Apprentice*:
> "It literally feels like we're wizards casting all these spells and these spells are kind of like going out and doing things for you."

**The One-Person Billion-Dollar Startup:**
Wu is bearish on this narrative. Instead, he predicts a startup proliferation effect:
> "If it's easy for a person to create a one-person billion-dollar startup, it also means it's way easier for people to just create startups in general."

> "There's going to be a huge startup boom and small SMB-style boom, where anyone can build software for anything."

> "There might be a hundred million-dollar startups. There might be tens of thousands of $10 million startups... it's actually pretty great to have a $10 million business."

> "I think we might actually enter into a golden age of B2B SaaS."

**"Models Will Eat Your Scaffolding for Breakfast":**
Referencing Nicolas Bustamante's concept — the scaffolding/wrappers developers build around models today will be absorbed by the models themselves as they improve.

**Build for Where Models Are Going:**
> "Make sure you're building for where the models are going and not where they are today."

> "This is the worst the models will ever be." (quoting Kevin Whale)

**Management Transformation:**
- Spend 50%+ of time with top performers, empowering them like "surgeons" (Mythical Man-Month reference)
- AI adoption requires **both** top-down buy-in AND bottom-up enthusiasm
- Build internal "tiger teams" of AI evangelists
- Avoid top-down mandates without grassroots excitement

**Challenge of 100% AI-Written Codebases:**
Teams maintaining fully Codex-written codebases face unique challenges without manual fallbacks. Solution:
> "Adding documentation and encoding tribal knowledge into the codebase."

**Next 18 Months Predictions:**
- Agents running longer, increasing task coherence
- Significant improvements in multimodal audio
- Models handling multi-hour tasks coherently
- Business process automation as the most underrated opportunity

**Product Recommendations:**
- Books: *Structure and Interpretation of Computer Programs*, *The Mythical Man-Month*, *There Is No Antimemetics Division*, *Breakneck* (on China)
- Hardware: Ubiquiti routers/cameras ("Apple of home networking")
- Life motto: "Never feel sorry for yourself."

**Takeaway:** This interview is the definitive Sherwin Wu content piece. The "sorcerer" metaphor captures the fundamental shift: engineers are evolving from code writers to agent orchestrators. The productivity data (95% Codex adoption, 70% more PRs, 10-20 parallel agents) is the most concrete evidence of how AI is changing engineering at the frontier.

---

### 2.15 Benzinga / OfficeChai Coverage (February 2026)

**Source:** [Benzinga](https://www.benzinga.com/markets/tech/26/02/50632038/openais-engineering-lead-says-ai-era-will-create-thousands-of-niche-startups-its-hard-for-me-to-imagine) | [OfficeChai](https://officechai.com/ai/ai-could-cause-an-smb-startup-boom-openais-sherwin-wu/)
**Date:** February 2026

Media coverage of Wu's Lenny's Podcast insights about the startup boom. Key additional quotes:

> "You're starting to see this play out in the AI startup scene where software has become a lot more vertical-oriented, where creating some AI tool for some vertical tends to work quite well."

> "If you play out AI, there's no reason why you can't have a hundred times more of these startups."

---

## 3. Key Themes Across All Content

### Theme 1: Engineers as Orchestrators, Not Coders

Wu's central thesis has evolved over three years:
- **2023 (QCon):** Function calling extends AI capabilities — LLMs get "tools"
- **2024 (o1 tweet):** Reasoning models change the product design paradigm
- **2025 (BG2/a16z):** Enterprise adoption requires FDE + model customization
- **2026 (Lenny's):** Engineers are "sorcerers" managing fleets of 10-20 agents

The progression: **tools → reasoning → customization → orchestration**. Each layer builds on the previous. By 2026, the engineer's job is not to write code but to orchestrate AI systems.

### Theme 2: Model Portfolio Over God Model

Wu has consistently moved away from the "one model to rule them all" thesis:
- Specialized models for specific tasks
- Fine-tuning (especially RFT) as the mechanism for specialization
- Users care about which model they're using — models are "anti-disintermediation technology"

### Theme 3: Build for Where Models Are Going

Wu's most actionable advice for developers:
- "This is the worst the models will ever be"
- "Models will eat your scaffolding for breakfast"
- Don't over-invest in workarounds for current model limitations
- The scaffolding you build today will be absorbed by tomorrow's model

### Theme 4: Forward Deployed Engineering as Moat

OpenAI's enterprise strategy is deeply informed by Palantir's FDE model:
- Engineers embed with customers
- Customer problems inform model improvements (T-Mobile → GPT-5)
- The feedback loop between deployment and research is the real moat

### Theme 5: Developers as AGI Distribution Layer

Wu frames OpenAI's entire platform strategy around one idea:
- ChatGPT reaches consumers (800M WAU)
- But the API reaches *every vertical* through developers
- OpenAI can't build for every industry — developers carry AGI into niches
- This is why the platform investment matters as much as the models

### Theme 6: From Prompt Engineering to Context Design

The evolution of developer skills:
- **2023:** Prompt engineering (crafting clever prompts)
- **2024:** Function calling / tool use (connecting to external world)
- **2025:** Context engineering (structuring data for model consumption)
- **2026:** Agent orchestration (managing fleets of AI agents)

### Theme 7: The Golden Age of Small Startups

Wu's contrarian take on the "one-person billion-dollar company":
- He's bearish on solo unicorns
- Bullish on thousands of $10M-$100M niche businesses
- AI lowers the barrier to build software for any vertical
- "A golden age of B2B SaaS"

---

## Latest Updates (2026)

### OpenAI Platform Scale & Model Evolution

Since Wu's most recent public appearances, OpenAI's platform — the infrastructure his team leads — has undergone massive evolution:

- **ChatGPT surpassed 900 million weekly active users** by January 2026, up from the 800 million Wu cited in his November 2025 a16z interview. The platform is on track to cross 1 billion WAU in early 2026, validating Wu's thesis that ChatGPT serves as the consumer distribution layer while the API carries AGI into verticals.

- **GPT-5.1 and GPT-5.2 shipped to the API.** GPT-5.1 introduced adaptive reasoning (dynamically adjusting thinking depth based on task complexity), extended prompt caching (24-hour retention), and improved coding personality — running 2-3x faster than GPT-5 on simpler tasks. GPT-5.2 followed with significant improvements in long-context understanding, agentic tool-calling, and vision, priced at $1.75/1M input tokens. Both models are available through the Responses API that Wu's team built. ([GPT-5.1 announcement](https://openai.com/index/gpt-5-1-for-developers/) | [GPT-5.2 announcement](https://openai.com/index/introducing-gpt-5-2/))

- **GPT-5.2-Codex and GPT-5.3-Codex set new SWE-Bench records.** Wu's claim that 95% of OpenAI engineers use Codex daily is now backed by even more capable models. GPT-5.3-Codex runs 25% faster and extends beyond coding to the full software lifecycle — debugging, deploying, monitoring, writing PRDs, and more. The Codex desktop app launched in February 2026 as a "command center for agentic coding" with parallel agents working across projects. ([GPT-5.2-Codex](https://openai.com/index/introducing-gpt-5-2-codex/) | [GPT-5.3-Codex](https://openai.com/index/introducing-gpt-5-3-codex/))

### Platform Strategic Moves

- **Assistants API sunset announced (August 2026).** OpenAI formally deprecated the Assistants API in August 2025, with full removal scheduled for August 2026. All features are migrating to the Responses API — the same API Wu positioned as "a potential industry standard beyond just MCP" at Dev Day 2025. This confirms Wu's team is consolidating around a single, agent-native API primitive. ([Deprecation notice](https://platform.openai.com/docs/deprecations))

- **Frontier platform launched for enterprise AI agents.** OpenAI released Frontier, a platform for businesses to build, deploy, and manage AI agents with capabilities described as similar to human "coworkers." Early customers include Uber, Intuit, State Farm, and Oracle. This is the productization of the Forward Deployed Engineering model Wu described on BG2 — moving from bespoke FDE engagements to a scalable enterprise agent platform.

- **Peter Steinberger (OpenClaw creator) hired to build personal agents.** In February 2026, OpenAI acqui-hired Peter Steinberger, whose viral OpenClaw AI assistant promised to be "the AI that actually does things." Steinberger will "drive the next generation of personal agents." OpenClaw remains open source. This signals OpenAI's push beyond developer tools into consumer-facing agent experiences. ([TechCrunch](https://techcrunch.com/2026/02/15/openclaw-creator-peter-steinberger-joins-openai/))

- **ChatGPT Health launched (January 2026).** OpenAI introduced ChatGPT Health, allowing users to connect medical records and wellness apps (Apple Health, MyFitnessPal, etc.) for personalized health conversations. This is a concrete example of Wu's "developers carry AGI into verticals" thesis — healthcare being one of the most valuable verticals, OpenAI is entering it directly while the API enables third-party health applications. ([OpenAI announcement](https://openai.com/index/introducing-chatgpt-health/))

- **Codex IDE integration with Xcode 26.3.** Apple integrated OpenAI's Codex directly into Xcode 26.3 (February 2026) for autonomous agentic coding, expanding the agent's reach beyond OpenAI's own tools into the broader developer ecosystem. This aligns with Wu's "developers as AGI distribution layer" — now Apple is a distribution partner.

---

## 4. Actionable Takeaways

### For Developers

1. **Start managing agents now.** OpenAI's own engineers run 10-20 parallel Codex agents. If you're still writing code line-by-line, you're already behind. The "sorcerer" model — casting spells (agent tasks) and monitoring results — is the new workflow.

2. **Don't over-invest in scaffolding.** Whatever wrapper, RAG pipeline, or prompt chain you build today, the model will likely absorb it within 12-18 months. Build the simplest version that works. "Models will eat your scaffolding for breakfast."

3. **Learn context design, not just prompt engineering.** The shift is from "how to write a clever prompt" to "how to structure data so the model produces correct output." This includes system prompts, retrieval context, tool descriptions, and output schemas.

4. **Test with the latest model.** GPT-5's instruction following is so strong that old prompt hacks (begging, threatening, role-playing) now backfire. Each model generation requires prompt re-calibration.

5. **Encode tribal knowledge in the codebase.** As AI writes more code, the biggest risk is loss of institutional knowledge. Wu's team solves this by embedding documentation and context directly in the codebase.

### For Founders

1. **Go vertical, go niche.** Wu sees AI enabling "a hundred times more startups" — not bigger companies, but more specialized ones. The winning strategy is deep vertical expertise + AI leverage.

2. **$10M businesses are the new $1B vision.** Wu explicitly says "it's actually pretty great to have a $10 million business." The VC-scale unicorn isn't the only path.

3. **Adopt top-down + bottom-up.** AI transformation fails with mandates alone. Build "tiger teams" of enthusiasts, give them resources, and let adoption spread organically.

4. **Don't build AI tooling startups.** Wu is explicitly short on "eval platforms, vector stores" — the competitive landscape evolves too fast and OpenAI will absorb the primitives.

### For Engineering Leaders

1. **Spend 50%+ of time with top performers.** Wu's management philosophy: treat top performers like "surgeons" (The Mythical Man-Month). Give them leverage, not process.

2. **Measure AI adoption honestly.** 95% Codex adoption at OpenAI, 100% PR review by AI, 70% more PRs. These are real metrics — establish similar baselines for your org.

3. **Prepare for the "too good" problem.** As models improve, the challenge shifts from "getting the model to do what you want" to "the model doing exactly what you say, literally." This requires clearer, more precise communication — not more creative prompting.

4. **Use Codex for code review first.** Wu's team cut review time from 10-15 minutes to 2-3 minutes. Code review is the lowest-risk, highest-ROI starting point for AI adoption.

---

## References

### Profiles

- LinkedIn: https://www.linkedin.com/in/sherwinwu1/
- X/Twitter: https://x.com/sherwinwu
- Sancus Ventures: https://www.sancus.vc/team/sherwin-wu
- InfoQ Profile: https://www.infoq.com/profile/Sherwin-Wu/
- QCon Speaker Page: https://qconnewyork.com/speakers/sherwinwu
- Quora: https://www.quora.com/profile/Sherwin-Wu

### Podcasts

#### Lenny's Podcast — "Engineers are Becoming Sorcerers" (Feb 2026)
- Lenny's Newsletter (transcript): https://www.lennysnewsletter.com/p/engineers-are-becoming-sorcerers
- Spotify: https://open.spotify.com/episode/3EPDh5GqC9PEdyQBEpmdrQ
- Apple Podcasts: https://podcasts.apple.com/us/podcast/engineers-are-becoming-sorcerers-the-future-of/id1627920305?i=1000749436380
- Podwise Summary: https://podwise.ai/dashboard/episodes/7171472
- 小宇宙: https://www.xiaoyuzhoufm.com/episode/698dd96e2aaefd8defac0fbf

#### a16z Show — "How OpenAI Builds for 800 Million Weekly Users" (Nov 28, 2025)
- Spotify: https://open.spotify.com/episode/36DXaD9KdlWh2Ksn7OVAdc
- Apple Podcasts: https://podcasts.apple.com/hn/podcast/how-openai-builds-for-800-million-weekly-users-model/id842818711?i=1000738780411
- Podwise Summary: https://podwise.ai/dashboard/episodes/6063995
- Podscan (a16z): https://podscan.fm/podcasts/a16z-podcast/episodes/how-openai-builds-for-800-million-weekly-users-model-specialization-and-fine-tuning
- Podscan (a16z Show): https://podscan.fm/podcasts/the-a16z-show/episodes/how-openai-builds-for-800-million-weekly-users-model-specialization-and-fine-tuning

#### BG2 Pod — "Inside OpenAI Enterprise" (Sep 11, 2025)
- Spotify: https://open.spotify.com/episode/0HnYWQc9retjyFVSEAmFmQ
- Apple Podcasts: https://podcasts.apple.com/us/podcast/inside-openai-enterprise-forward-deployed-engineering/id1727278168?i=1000726402888
- Everand: https://www.everand.com/podcast/915708725/Inside-OpenAI-Enterprise-Forward-Deployed-Engineering-GPT-5-and-More-BG2-Guest-Interview
- Chain of Thought Summary: https://podcasts.chainofthought.xyz/podcast-summaries/inside-openai-enterprise-forward-deployed-engineering-gpt-5-and-more-bg2-guest-interview
- InfoCaptor Summary/Transcript: https://my.infocaptor.com/hub/summaries/bg2-pod/inside-openai-enterprise-forward-deployed-engineering-gpt-5-and-more-%7C-bg2-guest-interview-yLTSqBzKG2s

#### Latent Space — "Developers as the Distribution Layer of AGI" (Oct 6, 2025)
- Latent Space: https://www.latent.space/p/devday-2025

### Conference Talks

#### QCon New York 2023 — "A Bicycle for the (AI) Mind: GPT-4 + Tools" (Jun 14, 2023)
- InfoQ Recording: https://www.infoq.com/presentations/bicycle-ai-gpt-4-tools/
- QCon Listing: https://qconnewyork.com/presentation/jun2023/bicycle-ai-mind-gpt-4-tools

#### QCon New York 2023 — Panel: "Navigating the Future: LLM in Production" (Jun 2023)
- InfoQ Panel Coverage: https://www.infoq.com/news/2023/06/qcon-ny-llm-panel/
- QCon Listing: https://qconnewyork.com/presentation/jun2023/panel-navigating-future-llm-production
- Day Two Recap: https://www.infoq.com/news/2023/06/day-two-qcon-ny-2023/

### X/Twitter Notable Tweets

- OpenAI o1 Paradigm Shift (Sep 12, 2024): https://x.com/sherwinwu/status/1834374228529086555
- GPT-4 Turbo with Vision GA (Apr 9, 2024): https://x.com/sherwinwu/status/1777882109086159100
- GPT-5 Instruction Following PSA (Aug 7, 2025): https://x.com/sherwinwu/status/1953520864592375933
- Tiktoken Model-to-Encoding Mapping (~Feb 2023): https://x.com/sherwinwu/status/1621918059203072001
- Lenny Rachitsky Promotion Tweet (Feb 2026): https://x.com/lennysan/status/2021971151862595672
- Martin Casado Endorsement (Nov 2025): https://x.com/martin_casado/status/1995893428375478579

### LinkedIn Posts

- Forward Deployed Engineer (~Jan 2025): https://www.linkedin.com/posts/sherwinwu1_forward-deployed-engineer-openai-activity-7284751404848619522-xW8x
- Engineering Manager, API Experience (~Jul 2023): https://www.linkedin.com/posts/sherwinwu1_engineering-manager-api-experience-activity-7082552188635402240-cUHA
- Lenny's Podcast Promotion by Lenny Rachitsky (Feb 2026): https://www.linkedin.com/posts/lennyrachitsky_engineers-are-becoming-sorcerers-sherwin-activity-7427736676929753088-91oU

### Articles & Analysis

- 01Cloud Engineering Blog — "How OpenAI Scales to 800 Million Weekly Active Users" (Dec 22, 2025): https://engineering.01cloud.com/2025/12/22/how-openai-scales-to-800-million-weekly-active-users-insights-from-sherwin-wu-on-model-specialization-and-fine-tuning/
- Benzinga — "OpenAI's Engineering Lead Says AI Era Will Create Thousands of Niche Startups" (Feb 2026): https://www.benzinga.com/markets/tech/26/02/50632038/openais-engineering-lead-says-ai-era-will-create-thousands-of-niche-startups-its-hard-for-me-to-imagine
- OfficeChai — "AI Could Cause An SMB Startup Boom" (Feb 2026): https://officechai.com/ai/ai-could-cause-an-smb-startup-boom-openais-sherwin-wu/
- StartupHub.ai — "How OpenAI Builds for 800 Million Weekly Users" (2025): https://www.startuphub.ai/ai-news/ai-video/2025/how-openai-builds-for-800-million-weekly-users-model-specialization-and-fine-tuning/
- Scripod — Episode Transcript (2025): https://scripod.com/episode/lgf925trdpphwtv4nw8rt1hz

### Video / Short-Form

- BG2 Clips — "OpenAI's first product wasn't ChatGPT" (TikTok): https://www.tiktok.com/@bg2.clips/video/7563697343414078728

### Related OpenAI Pages

- OpenAI DevDay 2025: https://openai.com/devday/
- OpenAI for Developers in 2025: https://developers.openai.com/blog/openai-for-developers-2025/

### 2026 Updates Sources

- GPT-5.1 for Developers: https://openai.com/index/gpt-5-1-for-developers/
- GPT-5.2 Introduction: https://openai.com/index/introducing-gpt-5-2/
- GPT-5.2-Codex: https://openai.com/index/introducing-gpt-5-2-codex/
- GPT-5.3-Codex: https://openai.com/index/introducing-gpt-5-3-codex/
- ChatGPT Health: https://openai.com/index/introducing-chatgpt-health/
- OpenClaw / Peter Steinberger joins OpenAI: https://techcrunch.com/2026/02/15/openclaw-creator-peter-steinberger-joins-openai/
- OpenAI API Deprecations: https://platform.openai.com/docs/deprecations
- ChatGPT Statistics (DemandSage): https://www.demandsage.com/chatgpt-statistics/
- OpenAI API Changelog: https://developers.openai.com/changelog/
