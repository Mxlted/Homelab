# Proxmox → Discord Webhook Integration

Send Proxmox alerts and notifications directly to a Discord channel using a webhook.

---

## Setup Steps

### 1. Create a Discord Webhook
1. In your Discord server, open **Server Settings → Integrations → Webhooks**.  
2. Click **New Webhook**, name it (for example, `Proxmox`), and copy the URL.  
   It should look like this:

   https://discord.com/api/webhooks/<webhook.id>/<webhook.token>

---

### 2. Add a Webhook in Proxmox
In the Proxmox web interface:
1. Go to **Datacenter → Notifications → Webhooks → Add**.  
2. Fill out the form as follows:

| Field | Value |
|-------|--------|
| Endpoint Name | Discord |
| Enable | Checked |
| Method | POST |
| URL | https://discord.com/api/webhooks/{{ secrets.token }} |

---

### 3. Headers
Add a new header:

Key: Content-Type  
Value: application/json

---

### 4. Body
In the **Body** field, enter the following JSON:

{ "content": "``` {{ escape message }}```" }

This formats Proxmox’s message output inside a code block for readability in Discord.

---

### 5. Secrets
Add a new secret:

Key: token  
Value: <webhook.id>/<webhook.token>

Important:  
Do not include https://discord.com/api/webhooks/ in the secret.  
Only include the part after /webhooks/.

Example:

REDACTED/REDACTED

---

### 6. Comment (optional)
Add a short comment such as:

Send Proxmox alerts to Discord channel

---

## Testing
1. After saving, select the new `Discord` webhook in the list.  
2. Click **Test → Send Test Notification**.  
3. You should see a message appear in your Discord channel that says:

Test notification from Proxmox

---

## Example Output
VM 101 started successfully

or  

Backup job failed on node pve1

---

## Notes
- You can customize the message in the **Body** field.  
  Example:

  { "content": "**{{ node }} ({{ severity }}):** {{ message }}" }

- To limit which events trigger notifications, configure **Notification Targets → Rules** in Proxmox.

---

## Result
Proxmox will now send alerts and status messages directly to your Discord channel using your webhook.
