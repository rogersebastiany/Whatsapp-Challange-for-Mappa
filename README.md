The first thing that called my attention about the current architecture is the fact that it is indeed a "black box", in which the recruitment team is unable to understand what is going on. So, my idea is to evolve the current architecture into a modular, event-driven system to work alongside the existing Dispatcher Server and AI Agent to solve the three core business needs: observing the funnel, intervening with at-risk candidates and allowing operator control.

The idea is to treat every candidate interaction as a discrete event. These events are then published to a central message bus. Different services can then subscribe to this stream of events to perform specific tasks. One service for analytics, another for notifications, and a new dashboard for recruiters.

---
## Solution Overview

**Observability**: An Analytics Service will listen to all events, process them, and store them in a database optimized for analytical queries. A new Recruiter Dashboard will query this database to visualize the candidate funnel in near-real time, showing drop-off rates at each stage.

**Predict and Notify**: A Scheduler Service will run periodically (e.g. every hour) to query the existing database for candidates who have not produced an "activity" event in the last 24 hours. When an "at-risk" candidate is found, this service will trigger a Notification Service to send a detailed alert to the recruitment team's Slack channel.

**Control (Operator Takeover)**: A new Message Router service will be introduced as the primary entry point for all incoming Whatsapp messages. This router will check a State Management Service to see if a conversation is in **BOT** or **HUMAN** mode.
	If in **BOT** mode, it forwards the message to your existing Dispatcher Server.
	If in **HUMAN** mode, it forwards the message to the Recruitment Dashboard, allowing the recruiter to chat directly with the candidate. The recruiter can then switch the conversation back to **BOT** mode when wanted.

---

### Workflow: Bot Mode

This is the default operational path, designed for end-to-end automation. Every new conversation begins in this mode.

1. **Ingestion**: A candidate's message is received by the **Meta API**, which triggers a configured webhook, sending the message payload to our central **Message Router**.
    
2. **Routing Decision**: The **Message Router** performs a real-time lookup against the **State Management** service. In this default path, the state is confirmed as `BOT`.
    
3. **Bot Activation**: The message is forwarded to the **Recruitment Bot**. The bot queries the **Primary DB** for the candidate's context and history, then engages its **AI Agent** to process the message and formulate a reply.
    
4. **Event Publication**: Having processed the interaction, the **Recruitment Bot** generates a structured event (e.g., `CANDIDATE_REPLIED`) and publishes it to the **Event Stream**. This action is decoupled from the main response flow.
    
5. **Analytics Pipeline (Asynchronous)**: The **Analytics Service**, a dedicated subscriber to the event stream, consumes this event, transforms the data, and persists it in the **Analytics DB**. This process populates the Recruiter Dashboard with near-real-time funnel data.
    
6. **Response**: The **Recruitment Bot** sends its generated reply to the candidate via the **Meta API**, completing the interaction loop.
    

---

### Workflow: Human Mode

This path is activated exclusively when a recruiter initiates an operator takeover, ensuring human intervention is a deliberate and controlled action.

1. **Ingestion & Routing**: The flow begins identically, with the **Message Router** receiving an inbound message. However, the query to the **State Management** service now returns a `HUMAN` state.
    
2. **Recruiter Notification**: Based on this state, the router bypasses the bot entirely and pushes the message directly to the **Recruiter Dashboard**, typically via a WebSocket for instantaneous delivery.
    
3. **Operator Action**: The message appears in the recruiter's inbox. They can now engage in a direct, live conversation with the candidate.
    
4. **Response**: When the recruiter sends a reply from the dashboard, its backend service calls the **Meta API** directly to deliver the message to the candidate's WhatsApp.
    

Throughout this workflow, the **Recruitment Bot** is dormant, ensuring a clean separation between automated and manual conversation states. The recruiter retains full control until they choose to revert the state back to `BOT` mode via the dashboard.

---

### Technology Choices & Trade-offs

| Component            | Recommendation           | Rationale                                                                            | Key Trade-off                                                                                                                   |
| :------------------- | :----------------------- | :----------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------ |
| **Event Stream**     | **QStash**               | Serverless, zero management overhead, and simple to integrate via HTTP.              | Functions as a task queue, not a replayable event log like Kafka. Better for simplicity than for advanced, log-based analytics. |
| **State Management** | **Redis**                | Provides a fully managed, serverless Redis instance with pay-as-you-go pricing.      | It's an external dependency, but the operational savings heavily outweigh the benefits of self-hosting for this use case.       |
| **Core Services**    | **Node.js (TypeScript)** | Excellent for I/O-heavy tasks, high developer productivity, and a massive ecosystem. | Go offers superior raw performance and concurrency, but with a steeper learning curve for teams unfamiliar with it.             |

## Implementation Plan

The project should be implemented in three distinct phases, ordered to deliver the most critical business value first. The priority is to gain visibility, then add proactive notifications, and finally implement full operator control.

---

### Phase 1: Foundational Visibility

The initial focus is to eliminate the "black box" and provide immediate insights to the recruitment team.

- **Instrument Bot:** Modify the existing bot to publish key events (e.g., `STAGE_COMPLETED`, `RESPONSE_RECEIVED`) to an event stream.
    
- **Build Analytics Pipeline:** Create a simple analytics service to listen to the stream and save the event data.
    
- **Launch V1 Dashboard:** Develop a read-only dashboard that visualizes the candidate funnel and drop-off points.

### Phase 2: Proactive Notifications

With visibility established, the next step is to build a system for proactive intervention.

- **Develop Scheduler:** Create a scheduled job that runs periodically to identify inactive candidates by querying the primary database.
    
- **Integrate Notifications:** Connect the scheduler to the Slack API to send automated alerts for "at-risk" candidates to the recruitment team.

### Phase 3: Operator Control

This final phase introduces the full real-time intervention and control capabilities.

- **Deploy Core Services:** Roll out the `Message Router` and `State Management` (Upstash) services.
    
- **Reroute Webhook:** Update the Meta API configuration to point the incoming message webhook to the new `Message Router`.
    
- **Enhance Dashboard:** Add the real-time chat inbox and state-control buttons ("Takeover" and "Resume Bot") to the Recruiter Dashboard.
    
- **Activate Full Routing:** Enable the final logic in the `Message Router` to direct messages based on the `HUMAN`/`BOT` state.

Of course. Here are a few further considerations for the project.

## Further Considerations

---

### Scalability

The proposed event-driven architecture is inherently scalable. Each service (e.g., Message Router, Recruitment Bot, Analytics Service) is decoupled, allowing it to be scaled independently to meet demand. This ensures the system remains responsive and cost-effective, even under a high volume of candidates.

### Data Privacy & Security

Handling candidate data requires strict adherence to privacy regulations like LGPD.

- **Privacy:** Personally Identifiable Information (PII) within conversation logs and analytics must be handled securely. We should implement data retention policies and consider anonymizing data used for funnel analysis to protect candidate privacy.
    
- **Security:** Key measures include validating all incoming webhooks from the Meta API to prevent spoofing and implementing robust authentication and role-based access control (RBAC) on the Recruiter Dashboard to ensure only authorized personnel can access sensitive candidate information.

### Potential Future Features

This architecture provides a strong foundation for future enhancements. The event and analytics data we collect can power more advanced capabilities, such as:

- **AI-Powered Risk Prediction:** Training a model to proactively identify candidates at high risk of dropping off based on their behavior patterns.
    
- **Sentiment Analysis:** Analyzing messages in real-time to flag candidates who may be confused or frustrated, allowing for quicker intervention.
    
- **Recruiter Macros:** Adding pre-written, templated responses to the dashboard to handle common questions more efficiently.
