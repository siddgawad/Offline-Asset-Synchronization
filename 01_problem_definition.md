🎯 Your Problem Definition (Locked In)
I'm going to state what you've defined. Confirm or correct:

Parameter	Value
Primary Entity	Containers with Goods inside
Data Fields	container_id, type, location, status, docs; goods_id, description, qty, weight, value, status, docs
Conflict Model	Multi-version storage (siblings). User chooses winner on next access.
Connectivity	Satellite (design for 128kbps, $10/MB)
Offline Duration	Up to 21 days (transpacific worst case)
Fleet Size	100 vessels
Containers per Vessel	2,000 average
Records per Day	~500 status changes per vessel
Estimated Daily Data	~50MB per vessel (before compression)
Critical Failure	Lost customs docs = $50K fine + cargo held



❓ One Question Before Phase 2
You said user chooses which version wins. That means:

When the Captain syncs in port and sees: "Shore says container DELIVERED, you said container DAMAGED"—they see a dialog and pick one?

If yes → That's manual conflict resolution. Works for low-conflict scenarios.

If no → You need an automatic rule (like "vessel data wins for physical status, shore data wins for customs clearance").

Which is it?

es we apply rule that since captain is on vessel he has more authority on physical status of container, shore manager has more authority on customs clearance and other documents, so we store both values as tuple and let user choose which one to keep, but we also have a default value that is set based on the authority of the user
Undo changes up to this point

