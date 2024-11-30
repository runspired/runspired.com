---
title: Adventures in WarpDrive | Cascade on Delete
published: true
---

# Adventures in WarpDrive | Cascade on Delete

Recently at [AuditBoard](https://www.auditboard.com/) we had a case come up where we needed to perform some additional cleanup whenever certain records were deleted.

For instance: imagine you have both a `user` and one or more `search-result` resources, where `search-result` contains a link to the full user and a few fields related to a search query or used as a table row. When `user:1` is deleted, you want to ensure that any `search-result:X` related to `user:1` is also deleted, because their existence no longer makes sense.

This could be achieved by writing a function `deleteUser` that you use anywhere a user is deleted that handles deleting both the user and iterating available search-results and deleting any that pointed at the user, or by manually handling this logic in each location in the code that requires it.

This approach doesn't scale well. Performance falls off the more kinds of search-results you might need to handle, and the more data you have in cache. In large systems, this can also become brittle: a developer may forget to handle the extra deletions or minor variations in the code that does so might create divergent expectations over time.

Faced with this problem recently I recognized it as a variant of "cascade on delete", a feature some users of EmberData/WarpDrive have wanted for ages. I also recognized this was doable now over some of the newer public APIs, and could be made performant by using some of the (still private but iterating towards public) Graph APIs.

## Step 1: Extending our Store with the new behavior

I figured `ResourcePolicy` was a good name for the primitive I wanted (to mirror `CachePolicy`)
and set out to write one. I started with scaffolding the shape of the policy and integrating it
with our store.

For this, I wanted to support both an upfront static config and the ability to dynamically
add to the config overtime (we deliver configuration from our API alongside schema information
for the types of records we typically care about this feature for).

*./resource-policy.ts*
```ts
import type { Store } from './store';

type ResourcePolicyConfig = {
  onDeleteAssociated: Record<string, Set<string>>;
};

/**
 * The ResourcePolicy manages rules for cleanup whenever a record is deleted,
 * allowing for more advanced behaviors like cascading or associated deletes.
 */
export class ResourcePolicy {
  store: Store;
  policy: ResourcePolicyConfig;

  constructor(store: Store, policy: ResourcePolicyConfig) {
    this.store = store;
    this.policy = policy;
  }

    /**
   * Adds a rule to attempt deletion of records of the `associatedType` when
   * a record of the `type` is deleted. This will only work for 1:none relationships
   * where the associated record has a one-way relationship to the primary type.
   *
   * Thus this is a very limited feature and should be used with caution, it is
   * primarily intended for use as a cascade delete on implicit relationships of
   * dynamically generated records.
   */
  addAssociatedDelete(type: string, associatedType: string) {
    this.policy.onDeleteAssociated[type] = this.policy.onDeleteAssociated[type] ?? new Set();
    this.policy.onDeleteAssociated[type].add(associatedType);
  }
}
```

*./store.ts*
```ts
import Store, { recordIdentifierFor } from '@ember-data/store';
import { ResourcePolicy } from './resource-policy.ts';

export class AppStore extends Store {
  /**
   * The ResourcePolicy manages rules for cleanup whenever a record is deleted,
   * 
   * You should not need to interact with this policy directly.
   */
  resourcePolicy = new ResourcePolicy(this, {
    // Note: this can be dynamically populared via handler based on request response meta
    onDeleteAssociated: {},
  });

  // .. other store config
}
```

## Subscribing to Cache Updates

One of the newer features in EmberData/WarpDrive is the [NotificationManager](https://api.emberjs.com/ember-data/release/classes/NotificationManager). By "newer" this feature has existed since the mid-3.x series, but its capabilities have expanded with time and it is not a feature that has generally been surfaced for general use (we should probably change that, consider this post your introduction).

The NotificationManager is actually how WarpDrive manages intelligent reactivity. Each UI Object that the store creates for the application (records, record arrays, documents etc.) uses this API to subscribe to the cache for updates. When an update occurs, the UI Object dirties any reactive signals for state that has changed.

This is also how the EmberInspector currently integrates with the store to watch for changes to the cache for its own use.

In addition to being able to subscribe to the changes to a specific document or resource, the NotificationManager allows subscribing to changes to `'added' | 'removed' | 'updated' | 'state'` for any resource or document. We're going to make use of that for this feature:

First, I updated the constructor to give us somewhere to store information about data that has
been recently removed and method call to kickoff our subscription handling.

```diff
+import type { StableRecordIdentifier } from '@warp-drive/core-types';
+import type { CacheOperation } from '@ember-data/store';
 import type { Store } from './store';

 type ResourcePolicyConfig = {
   onDeleteAssociated: Record<string, Set<string>>;
 };

 /**
  * The ResourcePolicy manages rules for cleanup whenever a record is deleted,
  * allowing for more advanced behaviors like cascading or associated deletes.
  */
 export class ResourcePolicy {
   store: Store;
   policy: ResourcePolicyConfig;
+  recentlyRemoved: WeakSet<StableRecordIdentifier>;

   constructor(store: Store, policy: ResourcePolicyConfig) {
     this.store = store;
     this.policy = policy;
+    this.recentlyRemoved = new WeakSet();

+    this._setup();
   }

   // ... more below
 }
```

Then I setup our subscriptions:

```ts
export class ResourcePolicy {

   // .. more between
  
  _setup() {
  const { notifications } = this.store;

    // any time a resource change occurs
  notifications.subscribe('resource', (identifier: StableRecordIdentifier, type: CacheOperation) => {
    // don't do any special handling for newly created, unsaved records
    if (!identifier.id) {
      return;
    }

    switch (type) {
    // if the change is a deletion, consider if we need to delete associated records
    case 'removed':
      void this._onDeleteAssociated(identifier);
      break;
    }
    });
    }

  // .. more below
}
```

## Performing the Cascade/Associated Delete

Ok so this part is going to get a little messy. Here's the full implementation of `_onDeleteAssociated` to get oriented with. It's annotated but afterwards I'll also walk through a few of the salient points:

```ts
import { assert } from '@ember/debug';

import type { GraphEdge, ImplicitEdge, ResourceEdge } from '@ember-data/graph/-private';
import { peekGraph } from '@ember-data/graph/-private';

// ... more between

export class ResourcePolicy {

  // ... more between

  _onDeleteAssociated(identifier: StableRecordIdentifier) {
    // This guards against multiple notifications for removal of the same
    // record, which occurs in (at least) 4.12 due to multiple parts of the
    // cache independently reporting the removal during cleanup.
    //
    if (this.recentlyRemoved.has(identifier)) return;
    
    this.recentlyRemoved.add(identifier);

    assert('identifier must have an id', identifier.id);
    const { store } = this;
    const { type } = identifier;
    const associated = this.policy.onDeleteAssociated?.[type];


    // if we have no rule for this type, no cleanup to attempt
    //
    if (!associated) return;


    // for our app, the 1:1 case is simple because our API endpoints
    // re-use the ID ala `query-result-user` and `user` share the same ID.
    // if we were to start using this logic for more than that case, we would
    // remove this optimization
    //
    if (associated.size === 1) {
      const associatedType = Array.from(associated)[0]!;
      const record = store.peekRecord(associatedType, identifier.id);
      if (record) {
        store.unloadRecord(record);
      }
      return;
    }

    // we need to find the implicitly related record
    // and then determine if all of its relationships are now empty
    // and only remove it if so: we use the graph to determine this.
    //
    const graph = peekGraph(store)!;
    const edgeStorage = graph?.identifiers.get(identifier);

    // If there are no edges, there are no relationships
    //
    if (!edgeStorage) return;

    
    // for our app's specific scenario, we only wanted to unload the record
    // if all associated relationships were now empty
    //
    const toUnload = [];
    for (const associatedType of associated) {

      // implicit keys match the pattern `implicit-${associatedType}:${inverseName}${randomNumber}`
      // gaining access to implicit keys via an explicit API is a feature we need to add when we
      // mark the Graph as a fully public API
      //
      const keys = Object.keys(edgeStorage).filter(
        (key) => key.startsWith(`implicit-${associatedType}:`)
      );
      const key = keys[0];
      assert('expected to find a key', key);
      assert(`expected to only find one key, found ${keys.length}`, keys.length === 1);

      const edge = edgeStorage[key];
      assert('expected to find an implicit edge', edge && isImplicitEdge(edge));
      const associatedIdentifers = edge.remoteMembers;

      // yup, that's a label. I hate me too but they are useful in this scenario.
      gc: for (const associatedIdentifier of associatedIdentifers) {
        // for each associated identifier, if all of it's own
        // relationships are empty (not including the one we're
        // deleting as it may not have been cleaned up yet),
        // then we can remove it.
        //
        const associatedStorage = graph.identifiers.get(associatedIdentifier);
        assert(
          `expected to find associated storage for ${associatedIdentifier.lid}`,
          associatedStorage
        );

        for (const assocKey of Object.keys(associatedStorage)) {
          const assocEdge: GraphEdge | undefined = associatedStorage[assocKey];
          assert('expected to find a belongsTo edge', assocEdge && isBelongsToEdge(assocEdge));

          if (assocEdge.remoteState !== null) {
            // if this edge is the edge that kicked off the deletion,
            // we treat it as removed even though the state is still
            // present in the graph.
            //
            if (assocEdge.remoteState === identifier)
              continue;


            // if we have remoteState that is not the originating
            // identifier, then this record cannot be removed, so
            // we break out both the inner and the outer loop.
            //
            break gc;
          }
        }

        // if we made here, then all of the associated record's
        // relationships are empty and we can remove the record.
        // 
        const record = store.peekRecord(associatedIdentifier);
        assert('expected to find a record', record);
        if (record) {
          toUnload.push(record);
        }
      }
    }

    if (toUnload.length) {
      for (const record of toUnload) {
        store.unloadRecord(record);
      }
    }
  }
}

function isBelongsToEdge(edge: GraphEdge): edge is ResourceEdge {
	return edge.definition.kind === 'belongsTo';
}

function isImplicitEdge(edge: GraphEdge): edge is ImplicitEdge {
	return edge.definition.isImplicit;
}
```

Ok, so in this implementation a few things were specific to AuditBoard's use cases:

- the associated records for which we cascade the delete only ever utilize belongsTo relationships, never hasMany relationships.
- the associated records sometimes (but not always) contain more than one belongsTo relationship, in the single case there's a simple fully public API to quickly perform the removal
- these belongsTo relationships are all implemented as one-directional (`inverse: null`)

All this means for you is that if you want to implement a similar concept in your app, its best to first understand the requirements your app has around cascading deletes. There are ways to handle hasMany relationships, cascade-on-delete for regular relationships, cascade-on-delete for completely unrelated records etc: you just need to know the rules and shape of the problem as it pertains to your app.

That is what is great about the NotificationManager, its a simple API but it lets us quickly write code that can do complex things within our specific app-domain logic.

The two questions you probably have come from the code handling what happens when one of our associated records relates to more than one type: what is the graph, and what are implicit edges.

## The Graph

The Graph is a library used by the JSON:API Cache which we built so that cache implementations (for any format) could delegate one of the hardest problems (managing relationships) to a primitive built and optimized for it.

We've kept the API for it private to this point because we expect to change the implementation of it quite a bit while adding support for paginated relationships, and there are some open questions around whether it should also become how documents are stored (the answer is probably "yes").

The Graph is just that, a graph: it stores the relationships (directional Edges) between resource CacheKeys (identifiers).

## Implicit Edges

When the Graph encounters a one-direction relationship e.g. something like a belongsTo or a hasMany with `inverse: null`, it creates an implicit inverse relationship for convenience and performance.

For instance, lets say a user has a car, the car gets into an accident and is totaled, and now the user does not have a car.

We might model this as a hasMany (`user.cars`) pointing at a belongsTo (`car.owner`), or we might model this with `inverse: null` (e.g. just `user.cars`, and no inverse on the car).

In the case there is no inverse, when the car is totaled and thus deleted, we need to know what relationships that car was in to remove its entry. That's where the implicit edge comes in, it tells us exactly what relationships currently point at the car, and so following them backwards we can quickly remove the car from all associated relationships.

This feature is one of the many small niceties of working with relationships in EmberData/WarpDrive, and it happens to sound exactly like the information we need for cascade-on-delete!

To be clear, we could implement this with public API by iterating all records of the associated type and checking the current values of their relationships, but that is a far more expensive operation as it requires iterating far more values, and potentially creating ui objects for them and their relationships.

So in our case with `user` and `search-result`, we reach in to the list of implicit edges in the graph for the user that got deleted, get the list, and for each implicit relationship if it matches our associated type (`search-result`) we perform the deletion check. Elegant really, though I wish we were closer to making this information publically and conveniently accessible. I don't like reaching into private APIs either 🙈

This exploration though lets me feel out what information should be public and what (future) public APIs are missing from the Graph in its current state.

---

This has been an Adventure in WarpDrive. Stay tuned for our next episode!
