# Questions

What if an update in one system should lead to a delete in another?
For instance if you want all "gold members" to be in a particular system and someone is downgraded from "gold" to "silver", he might need to be removed?
If we just set a filter and compare the current content (e.g. silver), we won't see that.

If you have a filter, we need the previous version as well. If your filter applies to the previous version but no longer to the current version, you are likely still interested in that change.






Data can be provided by pure masters that do not subscribe to changes but only publish them.
Subscriptions can be done by pure slaves that do not provide data but only consume it.

In a utility based approach, a single system can have multiple publishers for a single CDM.
This does assume however a standardization within the CDMs that allows for such matching subviews to be created.
One could imagine a "contacts" enricher that can apply to multiple business object cdms, assuming the cdms have the same position for the contacts array.
It becomes even more unlikely however when we include the need to target the correct entities which also requires a standardization in the external ids of those cdms etc.
But it is possible so we have to keep this in mind.

In the same vein, it is possible to have multiple subscribers for a given system.

Assumption is that if a single system has multiple subscriptions, they do NOT provide overlapping information from the same source system.


There are three major update paths:

## Webhook

In a webhook scenario we assume a single system has the capability of reporting a change near-realtime.

Our logic looks like this:

- fetch last known CDM instance
- get all subscriptions for that particular system
- diff (and patch) all changes to the core cdm
- get all OTHER system subscriptions that do not originate from that system
- map the newly patched core cdm to all the appropriate subtypes
- diff that subtype to the new core to see if the changes are relevant for that target system based on the hash of the resulting content

- if the hash differs, first do a get from the target system to get the latest data (it might have pending changes!)
	- do a diff against the previously known CDM (so without applied changes from the webhook system)
	- if there is a diff, we consider this another webhook update and publish that to be processed later
- we apply the original webhook diff to the fetched result
- we push the result and save the new hash

Consequences:

- if other systems (with no or delayed webhooks) has pending changes, they will be picked up ONLY if the change affects them
- if other systems have changes to the _same_ fields, they will be changed to the webhook value. parallel change should not occur as it is always hard to resolve "correctly"

## Full sync

In a full sync scenario we assume changes are waiting in systems but they do not have webhook capabilities. This is usually done nightly.

- we run through all the subscriber systems in order of priority (lowest to highest)
- we get the latest data and diff it against the "current" instance
- we immediately apply our changeset to the "current" instance so the next system diffs against that
- in the end we get a new resulting instance that may or may not have differences for each system
- we run through all subscriptions, diff against their last known version (possibly updated through the last pull) and see if we have any pending changes
- we apply pending changes and push it

Consequences:

- changes waiting in parallel systems are resolved through priority, when conflicts arise, the highest priority wins, again parallel changes are highly discouraged

## External modification

we modify it