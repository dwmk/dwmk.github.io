Just copy-paste the script into the "Console" part of browser dev tools:

## Method 1: Through APIs (requires more info, but faster)
```
(async () => {
    console.log("%c🔥 Redline API Deletion Started. Close tab to abort.", "color: #ff5500; font-weight: bold; font-size: 14px;");

    // --- CONFIGURATION ---
    const TOKEN = "YOUR_AUTHORIZATION_TOKEN_HERE"; // get by sending a message, goto Network tab, find message, open Header, find Authorization, copy it
    const CHANNEL_ID = "YOUR_CHANNEL_ID_HERE"; // get from URL or copy channel ID
    const AUTHOR_ID = "YOUR_USER_ID_HERE"; // get by copy ID
    // ---------------------

    const headers = { 'Authorization': TOKEN, 'Content-Type': 'application/json' };
    const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

    let lastMessageId = null;

    while (true) {
        // 1. Maximize batch retrieval to 100 (Discord's hard maximum)
        let fetchUrl = `https://discord.com/api/v9/channels/${CHANNEL_ID}/messages?limit=100`;
        if (lastMessageId) fetchUrl += `&before=${lastMessageId}`;

        const res = await fetch(fetchUrl, { headers });
        if (!res.ok) {
            console.error("Failed to fetch messages. Token or Channel ID might be wrong.");
            break;
        }

        const messages = await res.json();
        if (messages.length === 0) {
            console.log("✨ All clear. No more messages found.");
            break;
        }

        lastMessageId = messages[messages.length - 1].id;
        const myMessages = messages.filter(m => m.author.id === AUTHOR_ID);
        
        if (myMessages.length > 0) console.log(`Targeting ${myMessages.length} messages in this block...`);

        // 2. Execute Deletions
        for (let i = 0; i < myMessages.length; i++) {
            const msg = myMessages[i];
            let isDeleted = false;

            // Keep trying this specific message until it dies
            while (!isDeleted) {
                const deleteRes = await fetch(`https://discord.com/api/v9/channels/${CHANNEL_ID}/messages/${msg.id}`, {
                    method: 'DELETE',
                    headers
                });

                if (deleteRes.status === 204) {
                    console.log(`🗑️ Nuked: ${msg.id}`);
                    isDeleted = true;
                    
                    // The absolute bleeding edge of Discord's limit
                    await sleep(1050); 
                } 
                else if (deleteRes.status === 429) {
                    // We pushed too fast. Read the penalty time, wait, and loop again.
                    const data = await deleteRes.json();
                    const penaltyTime = (data.retry_after * 1000) + 100; // Add 100ms buffer
                    console.warn(`🚦 Redlined! Discord pushed back. Cooldown: ${penaltyTime}ms...`);
                    await sleep(penaltyTime);
                } 
                else {
                    console.error(`⚠️ Unhandled Error ${deleteRes.status} on message ${msg.id}. Skipping.`);
                    isDeleted = true; // Break the while loop to prevent getting stuck
                }
            }
        }
    }
})();
```

## Method 2: Through UI (easier, but slower)

```
(async () => {
    console.log("%c🧹 Clean Slate Automator Started. Refresh the page to stop.", "color: cyan; font-weight: bold;");

    const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

    // Safely dismiss Discord menus without crashing React or Floating-UI
    const hitEscape = () => {
        document.dispatchEvent(new KeyboardEvent('keydown', {
            key: 'Escape', code: 'Escape', keyCode: 27, bubbles: true, cancelable: true, view: window
        }));
    };

    // A simpler, safer click that passes the browser window context
    const safeClick = (element) => {
        if (!element) return;
        const box = element.getBoundingClientRect();
        const mouseEvent = new MouseEvent('mousedown', {
            bubbles: true, cancelable: true, view: window,
            clientX: box.left + (box.width / 2),
            clientY: box.top + (box.height / 2)
        });
        
        element.dispatchEvent(mouseEvent);
        element.dispatchEvent(new MouseEvent('mouseup', mouseEvent));
        element.click(); // Fallback native click
    };

    while (true) {
        // 1. Find all messages
        const messages = document.querySelectorAll('div[role="article"][data-list-item-id^="chat-messages"]');
        if (messages.length === 0) {
            console.log("Waiting for messages to appear...");
            await sleep(1000);
            continue;
        }

        const msg = messages[0];
        msg.scrollIntoView({ block: 'center', behavior: 'instant' });
        await sleep(150);

        // 2. Right-Click the message with a pristine, fully formed MouseEvent
        const box = msg.getBoundingClientRect();
        msg.dispatchEvent(new MouseEvent('contextmenu', {
            bubbles: true,
            cancelable: true,
            view: window, // CRITICAL: Fixes the 'Unable to determine render window' error
            clientX: box.left + 50,
            clientY: box.top + 20,
            buttons: 2
        }));

        await sleep(400); // Wait for context menu to open completely

        // 3. Find exact ID based on updated DOM logs
        const deleteMenuBtn = document.getElementById('message-delete');
        
        if (!deleteMenuBtn) {
            console.log("🚫 Not your message. Escaping menu and skipping.");
            hitEscape();
            msg.removeAttribute('data-list-item-id'); // Mark as done to avoid infinite loops
            await sleep(300);
            continue;
        }

        // 4. Click the Delete Context Menu Item
        safeClick(deleteMenuBtn);
        await sleep(450); // Wait for confirmation modal to render

        // 5. Find the final "Delete" button inside the modal
        const allElements = Array.from(document.querySelectorAll('span, div, button'));
        const confirmTextElement = allElements.find(el => el.textContent?.trim() === 'Delete' && el.closest('button'));
        
        if (confirmTextElement) {
            const finalButton = confirmTextElement.closest('button');
            safeClick(finalButton);
            console.log("✨ Deleted!");
        } else {
            console.log("⚠️ Modal missing or couldn't find delete button. Escaping.");
            hitEscape();
            msg.removeAttribute('data-list-item-id');
        }

        // Safe cooldown before grabbing the next message (increased to 1000ms to avoid rate limits)
        await sleep(1000); 
    }
})();
```
