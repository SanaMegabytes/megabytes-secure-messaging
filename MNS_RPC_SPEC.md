# MNS Messaging – RPC Commands Overview

> These RPC commands are provided for educational and developer testing purposes.  
> A dedicated graphical user interface (GUI) will later handle all messaging features  
> for end users. Direct RPC usage will not be required in the final product.

Below is the complete list of RPC commands implemented for OFF-CHAIN encrypted messaging in the Megabytes wallet.


## 1. mns_encrypt
Encrypt a plaintext message for a recipient (via their MNS nickname).

Usage:  
``` mns_encrypt "<recipient nickname>" "<message>" ```

Returns:  
An encrypted ciphertext payload (MMSG) ready to be transmitted through the P2P network.

---

## 2. mns_send
Send a previously encrypted MMSG payload across the P2P network.

Usage:  
``` mns_send "<hex_ciphertext>" ```

Note:  
The plaintext message is never sent through this RPC.  
Encryption must be performed beforehand via mns_encrypt.

---

## 3. mns_decrypt
Decrypt an incoming MMSG payload and store it in the wallet inbox.

Usage:  
``` mns_decrypt "<hex_ciphertext>" ```

Behavior:  
- Attempts decryption using the wallet’s private keys  
- On success: stores a MNSStoredMessage entry locally  
- Returns the plaintext message and metadata

---

## 4. mns_inbox
List stored OFF-CHAIN decrypted messages in the wallet.

Supports filtering, searching, and pagination.

Usage:  
``` mns_inbox [direction] [unread_only] [search] [limit] [offset] [read_filter] ```

Parameters:  
- direction → "all" (default), "incoming", "sent"  
- unread_only → true / false  
- search → substring filter (case-insensitive)  
- limit → max results (default 50, 0 = unlimited)  
- offset → skip N results (pagination)  
- read_filter → "all" (default), "unread", "read"

Examples:  
``` mns_inbox ```

``` mns_inbox incoming true "" 20 0 ```  

``` mns_inbox all false "hello" 50 0 read ```

---

## 5. mns_get_message
Return a single stored message by its msgid.

Usage:  
``` mns_get_message "<msgid>" ```

Returns:  
Full metadata including read status, timestamp, plaintext, ciphertext, and keys.

---

## 6. mns_mark_read
Mark a message as read or unread.

Usage:  
``` mns_mark_read "<msgid>" true ``` 

``` mns_mark_read "<msgid>" false ```

Effects:  
Updates the stored message entry and immediately affects inbox filters & statistics.

---

## 7. mns_delete
Remove a message from the wallet’s local inbox.

Usage:  
``` mns_delete "<msgid>" ```

Behavior:  
Irreversibly deletes the stored message from the wallet database.

---

## 8. mns_inbox_stats
Return aggregate statistics about the local MNS inbox.

Usage:  
``` mns_inbox_stats ```

Returns:  
- total  
- read  
- unread  
- last_message_time  
- incoming: { total, read, unread, last_message_time }  
- sent: { total, read, unread, last_message_time }

---
