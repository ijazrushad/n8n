📄 Guide: Building the AI Email Response & Approval Workflow
This guide provides a detailed walkthrough for setting up the two n8n workflows that power this AI email automation system.

⚙️ Part 1: The "Create & Approve" Workflow
This workflow is the primary engine. It triggers on a new email, generates a draft reply using AI, and sends an approval request to Microsoft Teams.

1. ➕ Create a New Workflow
Log in to your n8n instance (e.g., https://n8n.yourdomain.com).

Click "Add workflow" to start with a blank canvas.

2. 📧 The Trigger: On New Email
Add a Microsoft Outlook node.

Authentication: Connect your Microsoft account using the credentials you created in the Azure portal.

Event: Choose On New Email.

Folder: Select the folder you want to monitor, typically Inbox.

Fetch Test Event: Click this button to pull in a recent email as sample data. This is crucial for the next steps.

3. 🧠 The AI Brain: Generate Reply
Add a new AI node (e.g., Google Gemini or OpenAI).

Authentication: Connect your AI provider account by providing your API key.

Model: Select a powerful and cost-effective model, such as gemini-1.5-flash or gpt-4o.

Prompt: In the text field, write a clear prompt for the AI. Use expressions to pull data from the previous nodes.

Example Prompt:

You are a professional and helpful business assistant. Please read the following email and write a clear, concise, and polite draft reply.

Original Email Subject:
{{ $nodes["Microsoft Outlook Trigger"].json.subject }}

Original Email From:
{{ $nodes["Microsoft Outlook Trigger"].json.from.emailAddress.name }} <{{ $nodes["Microsoft Outlook Trigger"].json.from.emailAddress.address }}>

Original Email Content:

{{ $node["Get Email Body"].json.body.content }}

Draft your HTML reply below:

4. ✍️ The Action: Create a Draft Email
Add an HTTP Request node to use the Microsoft Graph API.

Method: POST

URL:

https://graph.microsoft.com/v1.0/me/messages/{{$node["Microsoft Outlook Trigger"].json.id}}/createReply

Body: Configure a JSON body to set the contentType to HTML and the content to the output from your AI node.

5. ✅ The Final Step: Send Approval Request to Teams
Add a Microsoft Teams node.

Resource: Adaptive Card.

JSON: Paste the Adaptive Card JSON from the main README.md file. This card will contain the draft and the interactive approval buttons.

Finally, Activate and Save your workflow!

🚀 Part 2: The "Send Approved Email" Workflow
This workflow is much simpler. It waits for the "Accept & Send" button to be clicked in Teams and executes the final send action.

1. ➕ Create a Second Workflow
Go back to the main n8n dashboard and click "Add workflow".

2. 👂 The Trigger: Webhook
Add a Webhook node. This will create a unique, permanent URL.

This URL is what the "Accept & Send" button in Teams will call. Copy the Production URL to use in the Adaptive Card of the first workflow.

3. 📤 The Action: Send the Draft
Add an HTTP Request node after the trigger.

Authentication: Use the same Microsoft credentials as the first workflow.

Method: POST.

URL: Use an expression to target the send endpoint with the Draft ID received by the webhook.

https://graph.microsoft.com/v1.0/me/messages/{{$json.body.draftId}}/send

Finally, Activate and Save this workflow. Your two-part automation is now complete!