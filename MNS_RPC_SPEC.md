# Megabytes MNS — Identity & Off-Chain Messaging RPC Overview

> These RPC commands are provided for educational and developer testing purposes.  
> A dedicated graphical user interface (GUI) will later handle all messaging features  
> for end users. Direct RPC usage will not be required in the final product.

Below is the complete list of RPC commands implemented for OFF-CHAIN encrypted messaging in the Megabytes wallet.

## 1. MNS Identity (On-Chain)

- **mns_register** — Register a new MNS identity on-chain  
- **mns_build** — Build a raw MNS registration transaction  
- **mns_list** — List all known MNS identity records  
- **mns_listmine** — List MNS identities belonging to the local wallet  
- **mns_resolve** — Resolve a human-readable MNS name into its identity pubkey, address, and metadata  


## 2. Encrypted Off-Chain Messaging (Wallet)

- **mns_encrypt** — Encrypt a message for a target MNS identity or address  
- **mns_decrypt** — Decrypt a received MMSG payload using the wallet’s private keys  
- **mns_inbox** — List stored MNS messages (incoming + sent) with filtering and search  
- **mns_get_message** — Retrieve a specific message by `msgid`  
- **mns_mark_read** — Mark a message as read or unread  
- **mns_delete** — Delete a message from the local inbox  
- **mns_inbox_stats** — Show inbox statistics (unread count, total messages, etc.)
- **mns_inbox_snapshot** - Try to decrypt wallet pending OFF-CHAIN MNS messages and move them into the wallet inbox.
- **mns_inbox_snapshot_delete** - Delete snapshot msgid - db level.
- **mns_inbox_pending_import** - Import ciphertext-only OFF-CHAIN MNS messages from the in-memory inbox into wallet pending storage.

Convenience RPC:

- **mns_send_offchain** — Resolve → encrypt → queue → store as “sent” (one-call end-to-end message sending)


## 3. Wallet Outbox (Message Queue)

- **mns_outbox_list** — List queued messages waiting for delivery  
- **mns_outbox_add** — Manually queue a message (recipient_key_id + payload_hex)  
- **mns_outbox_get** — Retrieve a specific queued entry by `msgid`  
- **mns_outbox_delete** — Remove a single pending message  
- **mns_outbox_clear** — Remove all queued messages
- **mns_outbox_markdelivered** — Mark Delivered


## 4. MNS Presence (Online Status Layer)

- **mns_presence_broadcast** — Wallet builds a CMNSPresence packet and signs it  
- **mns_presence_push** — Broadcast a serialized presence packet to all peers  
- **mns_presence_get** — Return the last known online status for an identity  
- **mns_presence_list** *(optional future)* — List presences for all known identities  

The P2P layer also listens for `MMSG_PRESENCE` and updates:

- Last-seen time  
- Peer who owns the identity  
- Presence timestamp (anti-replay)  


## 5. Raw P2P Control (Debug / Developer Tools)

- **mns_send** — Broadcast a raw encrypted MMSG payload to all peers  
- **mns_message_push** — Push an encrypted payload to a specific peer (NetMsgType::MMSG)  
- **mns_message_sendtoid** — Send MMSG to the peer currently owning a given identity  
- **mns_p2p_probe** — Developer tool to test INV, GETDATA, or direct MMSG push  
---
