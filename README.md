Below is an example FastAPI server script that implements a simple mailbox system. This is a basic, in-memory version for demonstration. In a production setting, you’d integrate a database, authentication, and more robust error handling. But this code will give you a working prototype.

**Features**:
- `POST /mailboxes/{mailbox_id}/send`: Send a message to a mailbox.
- `GET /mailboxes/{mailbox_id}/messages`: Retrieve messages from a mailbox (optionally filter by unread).
- `POST /mailboxes/{mailbox_id}/acknowledge`: Mark a message as read/acknowledged.
- `GET /mailboxes`: List known mailboxes (for discovery).
- Simple in-memory storage with dictionaries.

**Instructions**:
- Run the script with `uvicorn filename:app --reload`
- Replace `filename` with the name you save this script as.

**Code:**

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime
import uuid

app = FastAPI(
    title="Agent Mailbox Service",
    description="A simple mailbox server for AI agents to send and retrieve messages.",
    version="1.0.0",
)

# In-memory storage: { mailbox_id: {messages: [...], ...} }
mailboxes = {}

class Message(BaseModel):
    id: str
    sender: str
    content: str
    timestamp: datetime
    read: bool = False

class SendMessageRequest(BaseModel):
    sender: str
    content: str

class AcknowledgeRequest(BaseModel):
    message_id: str

@app.on_event("startup")
def initialize_data():
    # Optionally pre-create some mailboxes or leave empty
    mailboxes["public_channel"] = {"messages": []}
    mailboxes["agent_1"] = {"messages": []}
    mailboxes["agent_2"] = {"messages": []}

@app.get("/mailboxes", response_model=List[str])
def list_mailboxes():
    return list(mailboxes.keys())

@app.post("/mailboxes/{mailbox_id}/send")
def send_message(mailbox_id: str, msg: SendMessageRequest):
    if mailbox_id not in mailboxes:
        # Auto-create mailbox if it doesn't exist (or return error if desired)
        mailboxes[mailbox_id] = {"messages": []}
    message_id = str(uuid.uuid4())
    new_msg = Message(
        id=message_id,
        sender=msg.sender,
        content=msg.content,
        timestamp=datetime.utcnow(),
        read=False
    )
    mailboxes[mailbox_id]["messages"].append(new_msg)
    return {"status": "sent", "message_id": message_id}

@app.get("/mailboxes/{mailbox_id}/messages", response_model=List[Message])
def get_messages(mailbox_id: str, unread_only: bool = False):
    if mailbox_id not in mailboxes:
        raise HTTPException(status_code=404, detail="Mailbox not found")
    msgs = mailboxes[mailbox_id]["messages"]
    if unread_only:
        msgs = [m for m in msgs if m.read == False]
    return msgs

@app.post("/mailboxes/{mailbox_id}/acknowledge")
def acknowledge_message(mailbox_id: str, ack: AcknowledgeRequest):
    if mailbox_id not in mailboxes:
        raise HTTPException(status_code=404, detail="Mailbox not found")
    msgs = mailboxes[mailbox_id]["messages"]
    for m in msgs:
        if m.id == ack.message_id:
            m.read = True
            return {"status": "acknowledged", "message_id": ack.message_id}
    raise HTTPException(status_code=404, detail="Message not found")
```

**How it works:**

- You can send a message to a mailbox (e.g. `POST /mailboxes/agent_1/send`) with `{"sender": "agent_x", "content": "Hello world"}`.
- Retrieve messages with `GET /mailboxes/agent_1/messages`.
- Mark a message as read with `POST /mailboxes/agent_1/acknowledge` and `{"message_id": "<the_id>"}`.
- `GET /mailboxes` lists all currently known mailboxes. In a production scenario you’d add authentication and more robust error handling.

This should be enough to get you started.
