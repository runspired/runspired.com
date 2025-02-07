---
title: Exploring transformed and derived values in @warp-drive/schema-record
published: true
---

# Exploring transformed and derived values in @warp-drive/schema-record

With [@warp-drive/schema-record](https://github.com/emberjs/data/tree/main/packages/schema-record#readme) approaching its first stable release, now felt like as good a time as any to start writing about some of the key differences from [@ember-data/model](https://github.com/emberjs/data/blob/main/packages/model/README.md) in its approach to reactive-data.

The first thing most will notice is that the authoring format has changed from javascript classes to json schemas. We could spend a whole article on just that shift and all the implications of it, but not today.

There are a lot of key behavioral differences between SchemaRecord and Model, from a shift to immutability, built-in change buffering, deeply reactive object and array fields, partials, to (still under construction) a whole new relationship paradigm. These are all also worthy of their own in-depth blog posts.

Today, I want to focus on transformed and derived values. I was motivated to write this post following [this discussion in the emberjs discord](https://discord.com/channels/480462759797063690/1337069566940942388/1337143861964968017).

In the world of Models, engineers could use the class to add additional behaviors and derived (computed or calculated) values in addition to the schema fields defined via decorator. When using SchemaRecord, the only fields allowed are those defined via schema. In other words, **SchemaRecord places a fairly massive new constraint on just how much you can do.**

In this post, I want to explore that new constraint. Why did we add it? How does it help developers fall into the pit-of-success? And most importantly, are there any alternatives when using SchemaRecord? (*spoiler alert yes*).

## The World According to Schema

> On the sixth day of the second month of the 19th year of our library data, in the evening we lifted our eyes and low under the night-shift of the monitor we looked upon the git respository and from it issued forth a decree that henceforth our records must respect the boundaries of the data they represent.

In the world of Schema, every behavior of a record is defined by its schema and derived from the data in the cache.

**every**. **behavior**.

If the property or method is not included in your schema for the resource, it doesn't exist. Every SchemaRecord begins as a completely clean slate.

You may (or more likely probably do not yet) know that SchemaRecord has a special "legacy" mode that allows it to emulate the default Model behaviors. Everything from props on the instance like `isDestroyed`, `isDestroying` and `isReloading` to default fields like `id`, the state-machine `currentState` and its friends `hasDirtyAttributes`, `isDeleted`, `isEmpty`, `isError`, `isLoaded`, `isNew`, `isSaving`, `isValid`, to the methods `reload`, `rollbackAttributes`, `save`, `serialize`, `unloadRecord`, `deleteRecord`, `destroyRecord`. We even emulate the private `_createSnapshot` method and `constructor.modelName`.

Every single one of these, yes *every single one* is implemented by adding a schema-field to the definition for the resource, the result of the `withDefaults` call below.

```ts
import { withDefaults } from '@ember-data/model/migration-support';

const User = withDefaults({
  type: 'user',
  fields: [
    { name: 'firstName', kind: 'field' }
  ]
})
```

Outside of a small special group referred to as `locals`, all of these fields are created by a `derivation`. Yes: that means that derivations can return or do all sorts of things, even methods!

Beyond this, there is a [proposal](https://github.com/emberjs/data/issues/9534#issue-2534328361) being floated to allow apps to define not just the fields on a resource, but even what kinds of fields are valid. Each field "kind" today (`alias`,`schema-object`,`field`,`derived`,`resource`,`collection` etc.) is roughly speaking implemented as a function following a nearly identical signature. Given this, it feels like a natural progression of the schema-verse to allow registering `kind` functions just like you can register `derivations` and `transformations` (capabilities we will dive into more below).

So you see, while you can't just quickly slap a getter or a method on a class like you could with Model, the world is your oyster! *(please, please pretend I didn't just say that.. lest you steer your app into a miserable place)*

As the saying goes: **just because you could, doesn't mean you should!**

## Why is Everything Schema?

To understand why you shouldn't just write a whole bunch of new field capabilities and derivations to keep on doing whatever, lets take a moment to understand why SchemaRecord isn't implemented as a class you can extend in the first-place.

Javascript applications, especially those that target browsers, need to balance a lot of factors to maintain great performance characteristics, a few of the big buckets:

- asset download time
- asset parse time
- JS eval time
- runtime memory overhead

The Model approach to reactive-data performed poorly in all three of these categories.

As apps grow, the number of Models requires and the size of their definitions grows, leading to larger and larger assets and thus larger download and parse times.

Since those Models contain the schema, they are often needed synchronously at unpredictable times, leading to them generally being eval'd early during an app-boot cycle.

Since Models are subclasses (of at least Model and thus also EmberObject and its chain) and often use Mixins and lots of defineProperty calls (from the decorators), parsing them, evaluating and instantiating them is particularly expensive from both a compute and memory perspective.

In short, using Models for reactive-data is expensive and doesn't scale well with the demands of your application.

### So is SchemaRecord just about performance?

No, actually. There are ways for us to optimize record instances in ways that outperform the current SchemaRecord implementation (and we may introduce those as special modes in the future, they have different tradeoffs, we are currently balanced in favor of program correctness and helping developers catch accidental mistakes).

SchemaRecord is equally about *flexibility*. By having our record instances consume JSON schema to derive their behaviors, we gain not only the ability to deliver smaller, easier to parse JSON payloads that scale better **but also** the ability to deliver the definitions only when we need them and from whatever source is most optimal.

For instance, embedding these JSON schemas in your JS bundle, having them be separate JSON files you load alongside your JS bundle – or just-in-time (JIT), or having them be part of response payloads from requests you make to your API are all equally valid ways of delivering schema. And these modalities can be mixed as needed for apps to tune themselves. *flexibility*.

*flexibility* is also about what schemas mean from the perspective of partial-data and typescript. In the Model world, the Models were typically used as the type. This leads to friction where in some contexts fields are optional or invalid (such as during a create), while in others they are readonly.

In the SchemaRecord world, types are the types. They vanish from your runtime, and can be tailored to the context of specific edit, create or partial-data scenarios. For more about this I recommend reading the [TypeScript Guide](https://github.com/emberjs/data/blob/main/guides/typescript/2-why-brands.md).

### So is SchemaRecord just about performance and flexibility?

No, actually. It is equally about program correctness (I hinted at this above). Over the years, we have had the opportunity to watch developers intentionally and unintentionally misuse models in ways that lead to frustrating application bugs.

One of the most basic mistakes is treating a record as a convenient storage location for local component state. For instance, we'll often see records in a list get mutated to add booleans like "isSelected" "isFocused" and "isExpanded". At first this works and feels easy: then later this creates confusing bugs when the record is used by a different component trying to add and stash its own state.

With SchemaRecord, if a property isn't in the schema, accessing (or worse attempting to set it) will immediately throw an error. This ensures you aren't leaking unintended side-effects elsewhere in your code.

The full set of ways SchemaRecord is helping to steer you towards program correctness and protect against wierd and spooky bugs is fairly vast, and probably left for a blog post of its own. As a teaser, the way it goes about immutability and mutation is also designed to guide you to write more correct programs. Suffice it to say though: its important to realize that this is one of its primary goals. And it is likely to be one of the goals that developers (you) struggle with the most.

As programmers we've been programmed to hack at things until they work. With Models, we often could just hack until it seemed to work. SchemaRecord demands that you step back and think through how the thing *should* work and where various behaviors and state truly belongs.

### The Pit of Success

A basic principle of SchemaRecord is that a little friction and the right constraints go a long way towards steering apps into patterns that are performant, scalable, and correct by default.

Removing the ability to easily and quickly extend a Model with new behaviors is a key part of that principle.

If you find yourself asking "why do I have to write so many custom derivations" or "why am I writing so many custom transformations" or "why does this feel so hard to do" there's a decent chance the answer is "its meant to be".

Equally though, just because something is hard does not mean "you should never". Our goal as a library is to steer you towards what is usually best. Your goal, as a developer, is to know when to steer against the current.

And so with that prolongued introduction, lets explore two categories of steering against the current: transformed and derived values.

## Exploring FieldSchemas

The original question which prompted this discussion was asking whether accessing services on records was still possible, and if so how to have a field on a SchemaRecord that changed based on a user selected language set in the `intl` service.

In the Model world, this was solved with the following setup:

```ts
import Model, { attr } from '@ember-data/model';
import { service } from '@ember/service';

export default class House extends Model {
  @service intl;

  @attr declare houseDescription: {
    en: 'Great House',
    es: 'Buena Casa'
  },
  
  get description() {
    const lang = this.intl.lang;
    return this.houseDescription[lang];
  }
}
```

Today, lets focus on three specific kinds of FieldSchemas exploring how each might be used to solve this use case:

- [transformed fields](https://github.com/emberjs/data/blob/33193bf9097a122c1e51a543ea4ebf6a1a2a74d4/packages/core-types/src/schema/fields.ts#L3-L36)
- [derived fields](https://github.com/emberjs/data/blob/33193bf9097a122c1e51a543ea4ebf6a1a2a74d4/packages/core-types/src/schema/fields.ts#L407-L457)
- [aliased fields](https://github.com/emberjs/data/blob/33193bf9097a122c1e51a543ea4ebf6a1a2a74d4/packages/core-types/src/schema/fields.ts#L38-L87)


### Transformed Fields

You may have heard of transformations before when using Models. If so, you understand the rough idea of what a transformation is, but transformed fields are very different from the transformations that could be defined via Model attributes.

Defining a transform on a Model looked like this:

```ts
class User extends Model {

  @attr('string') name;
  //      ^ 'string' is the transform
}
```

This exact field definition can be defined in JSON as:

```json
{
  "name": "name",
  "kind": "attribute",
  "type": "string",
  "options": null
}
```

It may surprise you to know that this basically did nothing in the Model world ... unless you happened to be extending from one of the serializers provided by the package `@ember-data/serializer`, in which case *as long as you did not override the wrong normalization or serialization hook* would use Ember's resolver to lookup the transform to help normalize or serialize the payload.

Key takeaways about legacy transforms:

- operated on data to/from the Cache and your API
- weren't guaranteed to operate at all
- are definitely not type info (despite how many have tried to treat them)

A common pitfall that developers hit with legacy transforms is that they don't run when you mutate a record.

For instance, say you use the `'date'` transform to convert string dates to `Date` instances. Your API sends down a string, the serializer transforms the field, and the value in the cache is now a `Date` instance.

Now, say you are creating a new record with `store.createRecord('user', { birthday })`. What do you pass for `birthday`, a string or a Date instance? The answer is a `Date`, though often folks will set a string instead.

This gets really pernicious with the boolean and number transforms, because while the values coming from the API are converted into the proper form, if you update the value by binding it to a text-input... the value in the cache will now be a string.

Enough about the faults of legacy transforms though (and there are many). SchemaRecord guides us towards correctness, and one of the ways it does so is by introducing a complete rework of transforms. We'll call the new transforms `Transformations`.

Transformations:

- operate data to/from the cache and your app code (they run when the value is accessed or set on the record)
- are guaranteed to operate
- could well be type info (but still aren't, use types for types)

To see how this works, lets create and register a Date transformation (note, heavily recommend something immutable like luxon for Date values instead of raw Date)

```ts
import type { Transformation } from '@warp-drive/schema-record/schema';
import { Type } from '@warp-drive/core-types/symbols';

const DateTransform: Transformation<string, Date> = {
  serialize(value: Date, _options, _record): string {
    return value.toUTCString();
  },
  hydrate(value: string, _options, _record): Date {
    return new Date(value);
  },,
  [Type]: 'date',
};

store.schema.registerTransformation(DateTransform);
```

We register the transformation so that there is no ember-resolver magic. Like schemas, Transformations can be registered Just-In-Time, which means that if desired you can fetch and load transformations asynchrously alongside schemas. As long as the Transformation is registered by the time you access the field on the record instance, you're good to go.

In addition to some of the common scenarios like Date and Enum, we expect due to their guarantee to run that some folks will choose to use them to write validation layers for fields used in forms.

This is explicitly allowed, though not necessarily sensible as often form validation errors are best handled with other patterns. Validation purposes aside, throwing errors from transforms (especially in dev mode) for malformed data can be an effective way to enforce good habits and prevent sneaky bugs from occurring like integers getting coerced into strings.

### Implementing Mapped Translations Using a Transformed Field

First, lets create a schema and a type to match the data we will have:

```ts
import { withDefaults } from '@warp-drive/schema-record/schema';

const House = withDefaults({
  type: 'house',
  fields: [
    {
      name: 'houseDescription',
      kind: 'field',
      // ^ using 'field' instead of 'attribute' ensures we use
      // the new transformations behavior and not the legacy one.
      type: 'mapped-translation',
      // ^ This declares what transformation to use
    }
  ]
});

type TranslationMap = {
  en?: string;
  es?: string;
};

type HouseRecord = {
  id: string;
  $type: 'house';
  houseDescription: string; // NOT TranslationMap!
};

store.schema.registerResource(House);
```

Now, for the `mapped-translation` implementation.

```ts
import type { Transformation } from '@warp-drive/schema-record/schema';
import { Type } from '@warp-drive/core-types/symbols';

const MappedTranslationTransform = {
  serialize(value: string, options: null, record: SchemaRecord): TranslationMap {
    const lang = getOwner(record).lookup('service:intl').lang ?? 'en';
    return { [lang]: value };
  },
  hydrate(value: TranslationMap, options: null, record: SchemaRecord): string {
    const lang = getOwner(record).lookup('service:intl').lang ?? 'en';
    return value[lang] ?? '';
  },
  [Type]: 'mapped-translation',
};

store.schema.registerTransformation(MappedTranslationTransform);
```

With the above, when we access the `houseDescription` property we get the
correct description for our current language. Whenever the current language changes,
or whenever the cache updates with a new value for houseDescription the value on
our record will recompute.

Whenever we set the property, we update the cache with the new value. However,
in this approach the mutation is dangerous:

```ts
return { [lang]: value };
```

This will mean that the mutated state in the cache will lose any other languages that
had values. In some cases, this may be desired, but if we wanted to patch just the one
language we'd need a bit more info.

The downside of the transformation approach is that we don't give the schema for the field
being operated on to the serialize or hydrate methods. This was by design to avoid folks
getting too creative inside of transformations, though in a scenario like this it might be useful.

Lets say the options arg gave you access to the field-schema instead of just fieldSchema.options. Then we could do a merge in the cache during serialization to avoid removing other languages. We could also do this by duplicating a small amount of field information in the schema definition.

```ts
import { Type } from '@warp-drive/core-types/symbols';

const MappedTranslationTransform = {
  serialize(value: string, field: FieldSchema, record: SchemaRecord): TranslationMap {
    const owner = getOwner(record);
    const cache = owner.lookup('service:store').cache;
    const identifier = recordIdentifierFor(record);
    const lang = owner.lookup('service:intl').lang ?? 'en';
    const currentValue = cache.getAttr(identifier, field.name);

    return Object.assign({}, currentValue, { [lang]: value });
  },
  hydrate(value: TranslationMap, field: FieldSchema, record: SchemaRecord): string {
    const lang = getOwner(record).lookup('service:intl').lang ?? 'en';
    return value[lang] ?? '';
  },
  [Type]: 'mapped-translation',
};

store.schema.registerTransformation(MappedTranslationTransform);
```

Perhaps with time and feedback this is a restriction we will lift. The primary reason this restriction was put in place is to try to prevent transformations that compute off of additional fields, as this can lead to difficult to reason about differences between what the record presents and what is in the cache.

A bit of friction to steer folks the right way by default ... but a high-friction work around via padding additional info into options if the correct course is to steer against the stream.

Ok, so now for the `alias` approach.

### Aliased Fields

An AliasField can be used to alias one key to another key present in the cache version of the resource.

Unlike DerivedField (which we will see next), an AliasField may write to its source when a record is in an editable mode.

AliasFields may utilize a transformation, specified by type, to pre/post process the field.

An AliasField may also specify a `kind` via options. `kind` may be any other valid field kind
other than:
 - `@hash`
 - `@id`
 - `@local`
 - `derived`

This allows an AliasField to rename any field in the cache.

Alias fields are generally intended to be used to support migrating between different schemas, though there are times where they are useful as a form of advanced derivation when used with a transform.

For instance, an AliasField could be used to expose both a string and a Date version of the
same field, with both being capable of being written to.

### Implementing Mapped Translations Using an Aliased Field

In the alias approach, you retain exposing two fields like the original Model had, and you still write the transformation described above. The primary advantage is retaining access to the original field.

Here is our new House schema and types.

```ts
import { withDefaults } from '@warp-drive/schema-record/schema';

const MappedTranslationObject = {
  type: 'mapped-translation-object',
  identity: null,
  // ^ resource schemas with no identity field are used to describe reusable data structures
  // without our primary resource types
  fields: [
    { name: 'en', kind: 'field' },
    { name: 'es', kind: 'field' }
  ]
};

const House = withDefaults({
  type: 'house',
  fields: [
    {
      name: 'houseDescription',
      kind: 'schema-object',
      type: 'mapped-translation-object',
      // ^ this means the resource-schema for this object is 'mapped-translation-object'
    },
    {
      name: 'description',
      kind: 'alias',
      type: null,
      options: {
        kind: 'field',
        name: 'houseDescription',
        // ^ means this field will source its data from the field right above
        type: 'mapped-translation',
        // ^ means this field will use the transform we defined before
      }
    }
  ]
});

type TranslationMap = {
  en?: string;
  es?: string;
};

type HouseRecord = {
  id: string;
  $type: 'house';
  houseDescription: TranslationMap;
  description: string;
};

store.schema.registerResources([House, MappedTranslationsObject]);
```

This works exactly the same as the transformation approach except now we use `record.description` to get the description in the currently active language and can still access and update `houseDescription` directly.

One advantage of this is that because `houseDescription` is a `schema-object`, mutating it instead of `description` is both deeply-reactive and granular (the cache knows how to perform and store deep changes to schema-objects).

Finally, the derivation approach:

### Derived Fields

A DerivedField is a field whose value is derived
from other fields in the schema.

The value is read-only, and is not stored in the cache,
nor is it sent to the server.

Usage of derived fields should be minimized to scenarios where the derivation is known to be safe. 

For instance, derivations that required fields that are not always loaded or that require access to related resources that may not be loaded should be avoided.

### Implementing Mapped Translations Using a Derived Field

```ts
const MappedTranslationObject = {
  type: 'mapped-translation-object',
  identity: null,
  fields: [
    { name: 'en', kind: 'field' },
    { name: 'es', kind: 'field' }
  ]
};

const House = withDefaults({
  type: 'house',
  fields: [
    {
      name: 'houseDescription',
      kind: 'schema-object',
      type: 'mapped-translation-object',
    },
    {
      name: 'description',
      kind: 'derived',
      type: 'mapped-translation'
      // ^ the name of our derivation
      // this is not our transformation from before,
      // we will define this below
      options: { field: 'houseDescription' }
  ]
});
```

You'll notice that the above looks a lot like the alias approach. The main difference is that alias fields can be mutated, derived fields can never be set. 

And here's what that derivation would look like:

```ts
import { Type } from '@warp-drive/core-types/symbols';

function mappedTranslation(
  record: SchemaRecord & { [key: string]: unknown },
  options: Record<string, unknown> | null,
  _prop: string
): string {
  if (!options?.field) throw new Error(`options.field is required`);

  const opts = options as { field: string };
  const lang = getOwner(record).lookup('service:intl').lang ?? 'en';

  return record[options.field][lang] ?? '';
}
mappedTranslation[Type] = 'mapped-translation';

store.schema.registerDerivation(mappedTranslation);
```

## Parting Thoughts

You can write a derivation that gives you access to services, but its generally something I'd avoid except for key-data-concerns. Key-data-concerns might be things like:

- the intl service for use by derivations or transforms
- a clock service for use by time based derivations or transforms

But I wouldn't do somthing like give access to the store service or a request service.

Transformations and Derivations are only annoying if you try to make heavy use of them. If you keep it to a few well-thought-out transformations and derivations you can go really far and fast, but if you try to put tons of unique computations onto your records, it is intentionally annoying.

The point is to guide you into putting these calculations into the correct spots. Translations like this are a good use case for Transforms and Derivations because they are useful to tons of fields and tons of records and relatively simple calcs: you write the function once, register it, and can from then on make use of it in any schema.

A side-effect of this pattern that is super valuable for some apps is that these functions and transformations follow a contract and pattern simple and descriptive enough that is also lets them cross the client/server boundary.

Lets say you have a setup such that your API returns your schemas. It follows since the API knows the shape of the data and the schema that if you wanted to make the same derivation on the API (say for creating a PDF report or CSV export) then you can either swap the transformation/derivation implementation out for one that works for your API context and/or potentially just share the same function both places from a common library.

Small primitives. Constrained. But powerful.
