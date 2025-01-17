# todo

- use a separate type for pulling and pushing. pulling _MUST_ be a subtype of pushing (or equal to)

# Onboarding new systems

When you start a new cdm instance alltogether, we attempt to gather as much information as possible from all systems involved.
This means we ignore master definitions allowing systems to provide information beyond their daily master scope. Additionally we don't unset values so if a lower tier (in priority) system has information which a higher tier does not, that information will be retained.
In a normal sync the higher prio system would win, throwing away the information.

However, when you have an existing cdm instance and want to onboard a new system, we assume that you have spent time cleaning up the data in the CDM so it should be more correct in cases of conflicting data.
If we do nothing special however, all data from the new system would be flagged as "new" and overwrite your cleaned up CDM instance.

Instead for onboarding we have a similar special handling: if we see that CDM has never pushed the instance to your system, we assume your data is less valid which in turn means we ignore any changes to _existing_ data in the cdm instance.
However, the newly onboarded system CAN go beyond the master (NOT the initial master which is set up specifically for this purpose of onboarding but requires manual preparation). It can add information that it assuming the current value is "null" in cdm.

## Data overlap

If the system you are onboarding has a dataset that has a large overlap with the existing dataset, it is best to run a full batch sync because you will need all your system information to be as up to date as possible.
However, in some exotic cases you want to onboard a system that has a dataset that has little to no overlap with the existing data set.

For small overlaps "point syncs" are acceptable (pulling specific data from the remote systems) which is the default behavior if you sync without doing a batch pull first.

# The single sync

Because CDM is often triggered only on change, it is imperative that we sync _all_ changes to _all_ systems in a single go. Otherwise you may need to wait to the next change to push some latent changes.

The "trigger" and "create" are two examples where a source system performs a "pull" and the target systems perform a "push". However the "push" in the target systems might add data (especially metadata like relations & identifiers) that might not be picked up otherwise.

There are two ways to look at this:

- we optimize all the interactions so cascading changes are pushed to all systems (A triggers B which changes something that should be reported to A and C etc).
- we do a periodic check to see if there are unreported changes (e.g. last modified that is greater than the push date)

However, in the latter case we have to be careful with push rejections which culminate in never ending attempts to push the data.

# Indexes and constraints

alter table cdm_instance_external_ids add constraint uniq_name_identifier unique(name, identifier);

# Which diff to apply

You might assume that CDM diffs the current CDM instance with what we know from the target system and applies any changes. However it does not do that.
Instead, for each system it will keep track of the last "successfully" pushed cdm instance version.
When we push again, we diff the current cdm version with that last version and apply only the changes.

The advantage is that any unreported changes in the target system are not accidently rolled back (they would be flagged as a difference when comparing cdm to instance).
The disadvantage is that if a system were to get out of sync for some reason, it would not automatically restore itself unless that particular field would happen to have changed.

You can however force that behavior (pushing the last version of cdm regardless of current state of the target system) by using the "full push" feature.

# Arrays

Arrays are a tricky proposition. You can't simply conclude that yes or not a specific value is the same.

There are two types of arrays:

- simple type
- complex type

An array of simple types is still pretty easy to diff: you check which values differ between both arrays and you end up with a set of inserts and/or deletes

When it comes to complex values however, it becomes more difficult. If you have a list of contact persons and you want to update the first name of a contact person, how do you match the updated contact person to the original?
If you don't match them, it will seem like two different records and you will delete the original and insert the updated rather than "correctly" flagging the first name as an update.
This is suboptimal as you can't correctly trace the history of these values but it would still work so it might be acceptable.

However, another problem is that suppose you have 3 contact persons and you delete the first. If you simply assume the indexes match (so record 0 in the original must equate to record 0 in the new version), this will trigger a cascade of deletes and inserts because the system concludes that every record has changed.

You can prevent this cascade by having an identity to each record which can be matched. The diffing algorithm supports this if you mark a field with "primary key".

However if you don't set a primary key at this time, it will assume the index is the key which can trigger a delete cascade.

In the future we may want to tweak the algorithm in case there is no obvious primary key:

- serialize each entry (e.g. json) (we assume that you are using defined type which means the keys will be in the same order every time)
- the serialized version is the key, we still can't properly log updates to nested fields but we can at least prevent delete cascade

(recurse for nested arrays)

## Referential deletes

Suppose we again have 3 contact persons and in one system you delete the first and in the second system you delete the second.

If we diff the first system against the CDM, we conclude "delete [0]". However, we don't apply that just yet, instead we diff the second system against the cdm and conclude "delete [1]".

If we now sequentially run these two diffs, we actually end up with an array that only contains the second element. Inserts happen at the back but could also affect this given enough systems with differences.

The solution here is to:

- accept only one system updates arrays in any given push (AS IS!)
- use the primary key (if present) to delete the correct one
- use the stringification-as-key approach to ensure (before actually applying the delete) that you are deleting the correct one

If it becomes tricky to access the original version to redetermine the correct key to delete, we might want to consider hashing the stringified version and using that as the delete key, for instance instead of "delete [0]" we say "delete [hash]".

# The initial problem

Suppose a few scenario's:

- you are starting a new CDM and onboarding 5 systems immediately
- you are starting a new CDM with 4 systems and onboard a fifth system on day 2
- you are starting a new CDM with 4 systems, and after a year you want to onboard a fifth system

The question becomes: which data should win? When 5 systems are simultaneously merged for the first time, the priority level will dictate which values "win" out.
If you onboard a system after onto an already existing cdm instance, potentially all "stale" data of the new system will be flagged as a delta from the cdm. And because the other systems are not reporting any deltas, the priority concept does not kick in and all the "stale" data wins by default.

The reason the second and third scenario are different is because on day 2 you probably haven't cleaned up all your data, so the data in the cdm and the new system are equally likely to be correct.
In the third scenario you are onboarding a system with stale data onto a cdm that has been running for a while and likely contains data that is more correct than the outlier system.

We can't really use the "master" document definition because this is intended to highlight where mutation can take place, it does not reflect the initial data quality that is present.

We could use a "initial master" document structure that highlights the fields that are of high quality. This does not really solve the priority issues that arise when a new system is onboarded into an existing CDM setup, but it does limit the impact on the amount of fields.

The downside here is also that is it not "intelligent", it can not follow different rules based on the instance itself. If you are merging say 50.000 customer objects with a newly onboarded system, you might be able to automate 49.000 of them but still need to review 1000 merges.
You don't want to arbitrarily start human tasks for each merge, but it requires actual logic to determine which merges should be reviewed.

In the future we might add "merge" service support which can be used to onboard a system to an existing CDM instance.

# Resolving paths

Suppose you have 4 systems A through D.

A knows an identifier for B, B for C, C for D.

If you create an organisation based on an existing entry in A, you can cascade this to B, then C, then D.

However if you create an organisation from D, it becomes tricky.

Step 1:
D: create

Step 2:
A: unknown
B: unknown
C: link to C

Step 3:
A: unknown
B: link to B

Step 4:
A: link to A

Depending on the depth of the linkage based on the initial data, we may need to loop A-B-C-D multiple times to get the corresponding entries everywhere.

We can assume that A does not actually interact with the remote system in step1, step2 and step3 because it has no relevant identifier to do so.
This means we can keep poking at A until it returns something without incurring overhead. If of course there is something to return.

Knowing whether A can not actually resolve anything useful because there isn't anything or there isn't enough information is hard though.

We could add a "resolution" priority which would indicate the path to follow when resolving a new entity. However that path is likely determined by your starting point.
We would need more insight into how systems are correlated to extract a relevant path for resolution.

Currently we are limited to 1-deep resolving based on a good trigger system that has a substantial amount of identifiers.

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

### Standard batch sync

In the most basis batch sync principle we loop over our own items during a periodic run and for each instance perform this:

- pull from each system and incoporate changes into the CDM
- push to each system with the diff

The timing interval between pull and push is short enough because we are doing this per instance. No concurrent changes are expected and this type of sync could even presumably run during business hours.

This sync has a number of drawbacks:

- we can't use it to pick up "new" items from remote systems
- it will always scan all systems, even if those systems have more intelligent ways of syncing changes like last modified syncing
- it performs a lot more calls to each system as we do an instance get rather than a list for every item

### Custom batch sync

If no webhooks are available, a custom batch sync is likely the best way to detect newly created items.
However you don't want to first loop over all entities in an external system just to detect new ones, just to turn on the standard batch sync and pull all entities _again_ with a get.

Additionally depending on how clever the remote system is, you can do optimized batch syncs where you can only fetch any modified data.

For this reason there is also a pullAll service which will send you the date that the last batch sync was done and a configurable batch window so you can use paging.
If you set the "optimizedPullAll" boolean to true, once the batch process is done, the system will assume everything that was not modified is actually "up to date" and mark them as such with the timestamp of the START of the custom batch. That is the latest point in time we know for certain that there was no change.

If at least one system has a custom batch sync, the system will follow a slightly different logic: it will first start all batch syncs in separate tasks. Then once those are done it will proceed with a standard batch sync.
IMPORTANT: to prevent the standard batch sync from still pulling data from a system that performed a custom batch sync, you _must_ configure a pullInterval that is representative of how long the whole process takes. Note that it is not only the runtime of your own sync, but potentially other custom batch syncs as well and whether or not they run in parallel.

### Design choice

In the first iteration the idea was just to offer you tools so you could push data into the system which meant externalizing the entire custom batch process.
However it was added to this framework because of a few reasons:

- the system now actually knows about the custom batch processes and can wait for them to finish, ensuring that the data is up to date. Otherwise you would have to rely on "smart scheduling"
- for "optimized" batch syncs you usually need a last run which you would need to manage somewhere anyway
- we would prefer a centralized overview of _all_ the moving parts involved in the CDM syncs, this also means centralized logging to figure out what went wrong (if anything)
- we can internalize logic like "optimized batch pull" through configuration rather than you having to actually code those things into your solution

## Relations

A relation is unidirectional which means if a system actually has a bidirectional relation between two entities, two relation entries must be created: one in each direction.

The reason for this is that a lot of relations are 1-*. If we sync the * we will usually get the correct relation to the 1, but when we are syncing the 1, we usually won't be getting the correct relations to the *.
This leaves us with a problem when syncing the one: is the relation no longer needed? Is it simply unknown from that end of the relation?

The system expects the pull services to always send all relations that are known, any relation that is not explicitly reinforced to exist will be removed upon pull.

We provide you with the existing relations 

## Deletes

Deletes are picked up as we try to "pull" from a system but it returns nothing where there was something before.
If we do a push with a forced pull, we will also notes this particular discrepency and set it to decoupled.

Note however in a custom batch sync scenario it is a lot harder to pick this up. Most remote systems will not be able to report this and the batch sync will simply not report on it.
Especially in the optimized custom batch sync where you use the last modified, it is hard to distinguish between "it wasn't updated" or "it was deleted".
And the custom batch is specifically meant to prevent case-by-case pull later on so the push will bypass any additional pull and simply try to push the data.
The subscriber will likely throw an exception which will cascade into a rejection.

For this reason (and also to know the actual state of the remote system after rejection) an additional provide is done on reject, this can pick up deleted data.

### Combine with external looping

We can run a batch service that pulls all items from a remote system and pushes them to the CDM (this counts as a pull though). The push logic can do a few things:

- if it does not exist yet, create it
- the cdm might do a check (pulled vs pushed date) that all involved systems have received new data since their last push, if so, it can start a syncing of that item early (needs locking though to ensure no double syncs) -> perhaps not doable because of next comment

If your batch service has an optimized loop (e.g. last modified), it can run an additional service to mark all items from its core system for a particular cdm as being "in sync". It is unclear how this would be combined with the instance level check though, especially with locking.

You can run your batch sync processes for multiple systems in parallel, they are just feeding data into the CDM. Once they are all done, the CDM can start a syncing round. To prevent the CDM from _again_ retrieving the data, we have added a pull interval. If the CDM sync routine starts within that timeframe, it will not refetch the data.

Because such batch services can take a while to finish though, there can be quite a time interval between fetching the data and updating it. During this time any change in the remote system would be reset once the CDM sync kicks in.

## System Data

When we look at external systems we need to ask ourselves two questions:

- which data do they _have_? Our CDM is the union of all data pertaining to a particular subject in an enterprise. Each system however is likely only interested in a part of it.
- once we've established which data they have, we need to determine which data they are allowed to update and which they are only allowed to receive updates about

Ideally, even for the data that they are not allowed to update, we would like to know which data a system has. This will allow us to optimize synchronization strategies (e.g. only push if there is an actual difference). Additionally it allows us to check for discrepancies. For instance if the business decides that a particular system should not be able to update a particular field, we can check that it indeed doesn't.

We craft a derivative CDM for each system that reflects the fields that they actually have. 






## Pull vs push

We can pull data from a system and we can push data to it.

A system can have read and/or write permissions on a subset (or all) of the fields of a CDM.
In most cases a system will have fewer write permissions than it does read permissions, this means if we were to look at exact data definitions for push and pull, the "pull" would have much less information than the "push".
It is assumes that every field in the "pull" is also available in the "push" because even though you might have master permissions on a particular field, so might someone else.

When it comes to "concurrent changes" in different systems, it only relates to the fields available in the pull, not to any additional fields that might be available in the push. They are _intended_ to overwrite concurrent changes happening in fields that it should not be meddling with.

When pushing data we actually have two secondary goals:

- prevent overwriting changes that have not been reported yet with historic values
- don't push data if the system is already in sync, this helps reduce the load on the system

For the first we only care _if_ the fields in question are available in the "pull", otherwise you are not a master and are not allowed to have your own values.
Assuming no unplanned meddling in the target system, diffing our latest pushed data with the data we are about to push can prevent pushing the same data twice.

Because all data in the pull _has_ to be present in the push, saving the last "pushed" content is the safer path.

A push without pull can exist (the system is purely a slave), a pull without push should not exist as the system should at the very least consider that other systems might also be master of some overlapping fields.
If you were to set up a pull without a push however we don't need to optimize away the amount of push calls so we don't need a content to diff against.

All that said means that the content in the instance subscription table is that of the last _pushed_.

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