# todo

- merging of "new" identifiers needs to take into account unchanged values, in most cases you will simply push "all" identifiers

# Scenario's

## Webhook

The webhook approach is the simplest: a system reports something has been changed in near realtime. This has the following characteristics:

- targeted: we know exactly which items are impacted, no need to scan entire systems
- frequency: webhooks are called as they occur which means the frequency is higher on a per instance basis than for example a batch sync system: you might get multiple webhook calls for a single instance during the day versus a single batch sync at night. However, when looking at overall numbers of entities being synced back and forth the frequency is very low because you are only exchanging actually changed items rather than having to check everything for change
- concurrency: because webhooks happen near real time, they are by their very nature during a period of active usage of the systems. This means there is a higher likelyhood of concurrent changes that are not reported and processed yet.

The webhook approach follows this approach:

- we pull _only_ the system that reported the change and merge this into our version of the CDM
- we pull all other systems but do _not_ merge this into our version of the CDM (meaning the change can be picked up later). The reason we pull at all is to prevent accidently overwriting changes with historical data
- we merge the changes from the webhook into the separate target systems

If we do notice a change while pulling from a system that is not the source of the webhook, we do incorporate this into the update we push to that system but we do not report it yet to the system. This will have to wait until that system also performs a webhook or until a batch sync kicks in.
Because our CDM has not incorporated the change, the system will trigger the next time and register this as a change.

## Batch sync

In the most basis batch sync principle we loop over our own items during a periodic run and for each instance perform this:

- pull from each system and incoporate changes into the CDM
- push to each system with the diff

The timing interval between pull and push is short enough because we are doing this per instance. No concurrent changes are expected and this type of sync could even presumably run during business hours.

This sync has a number of drawbacks:

- we can't use it to pick up "new" items from remote systems
- it will always scan all systems, even if those systems have more intelligent ways of syncing changes like last modified syncing

### Combine with external looping

We can run a batch service that pulls all items from a remote system and pushes them to the CDM (this counts as a pull though). The push logic can do a few things:

- if it does not exist yet, create it
- the cdm might do a check (pulled vs pushed date) that all involved systems have received new data since their last push, if so, it can start a syncing of that item early (needs locking though to ensure no double syncs) -> perhaps not doable because of next comment

If your batch service has an optimized loop (e.g. last modified), it can run an additional service to mark all items from its core system for a particular cdm as being "in sync". It is unclear how this would be combined with the instance level check though, especially with locking.

You can run your batch sync processes for multiple systems in parallel, they are just feeding data into the CDM. Once they are all done, the CDM can start a syncing round. To prevent the CDM from _again_ retrieving the data, we have added a pull interval. If the CDM sync routine starts within that timeframe, it will not refetch the data.

Because such batch services can take a while to finish though, there can be quite a time interval between fetching the data and updating it. During this time any change in the remote system would be reset once the CDM sync kicks in.

# Push vs pull

The "pulled" date indicates when we last received an update of what the actual data in the remote system is.
The "pushed" date indicates when we last pushed our changes to the remote system.

If we push data, assuming no problems in the target systems, we can assume any pull done immediately after should return the exact same data.

# Questions

What if an update in one system should lead to a delete in another?
For instance if you want all "gold members" to be in a particular system and someone is downgraded from "gold" to "silver", he might need to be removed?
If we just set a filter and compare the current content (e.g. silver), we won't see that.

If you have a filter, we need the previous version as well. If your filter applies to the previous version but no longer to the current version, you are likely still interested in that change.




pull updates the remote data in our system but ALSO the cdm
push does a localized pull to update our data but NOT the cdm, this has to be done in a later pull
update pull to have a 





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