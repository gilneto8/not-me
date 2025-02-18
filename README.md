# Not-Me

> Easy and type-safe validation

## Features:

- Planned out to be used and shared **between backend and frontend**
- **Powerful type inference**: no need to write types manually. Even _Discriminated Union types_ are guessed from your schemas.
  - Example of a _Discriminated Union Type_:
  ```typescript
  type UnionType =
    | {
        discriminator: "has-a";
        a: string;
      }
    | {
        discriminator: "has-b";
        b: boolean;
      };
  ```
- Simple **DSL-like** API
- **No need for _try/catches_**: the final result will always be returned as the transformed input, or a tree of error messages. This way, there is no need to rearrange the flow to accomodate the _try/catch_, while reminding you to deal with validation errors
- **Easy to extend**. You can create a new schema just by extending the classes from the ones provided here
- **Implementation that is easy to read and change**. If you really need so, you can fork this library and change it without much hassle
- **API inspired** by `yup` and `joi`

## Quick links:

- [CONTRIBUTING](CONTRIBUTING.md)
- [LICENSE](LICENSE.md)

## Motivation behind this package

This package was built in order to fix some shortcomings (specially with _type safety_) that many validation libraries have. Most validation libraries try to do a lot, and their code starts getting confusing and with a lot of back and forths. As consequence, unsolved Github issues start pilling up, improving or extending the libraries ourselves becomes hard since there are too many flows with some history behind them, and a lot of broad types (like `any` or `object`) start surfacing and undermining the _type safety_ of a project.

Our take on validation with `not-me` is almost to provide you with enough boilerplate for you to build your own validations schemas. This way, it's easy to maintain and improve this package.

## How to use it

### Installing:

```
$ npm install not-me
```

### Imports:

> Most IDEs and Javascript text editors (like _Visual Studio Code_) import modules automatically just by starting to write the name of the value you want to use.

Keeping an app's code splitted into small lazy-loaded chunks is a priority in frontend development. Since legacy systems and some bundlers, like React Native's _Metro_, do not have tree-shaking, this package does not provide a single `index.js` import with all the code bundled in it. Instead, you are encouraged to import what you need from within the directories the package has. For example, the schemas are inside the `lib/schemas` directory, so if you want to import a schema for an object type, you need to import it like this `import { object } from 'not-me/lib/schemas/object/object-schema`

### Building schemas:

This library offers the following basic types for you to build more complex validation schemas:

- `array(elementsSchema)`
- `boolean()`
- `date()`
- `equals([...allowed values])`
  - use `as const` for when you want the types to be the exact value literals. Example: `equals([2, 'hello'])` validated value will be typed as `number | string` but `equals([2, 'hello'] as const)` validated value will be typed as `2 | 'hello'`
- `number()`
- `object({ property: schemaForTheProperty })`
- `objectOf(schemaForAllProperties)` - same as `object()` but for objects whose keys can be any string
- `string()`
- `or([...schemas])` - the value is filtered by multiple schemas till one matches. It's the equivalent to an _union type_

With these basic blocks, you can build more complex validations, by chaining...

- `test((v) => <condition>)` - will validate if your value matches a condition
- `transform((v) => <transform input value into any other value>)` - will allow you to modify the input value
- `defined()` - sets the schema to reject `undefined` values
- `nullable()` - sets the schema to accept `null` values

The methods above are all inherited from the `base()` schema. Other schemas might provide their own helpful methods, like `string()` provides `string().filled()`, a method that makes sure the field is filled not just with blank spaces.

Typescript will guide you in the recommended order by which you should chain validations. But if you use pure Javascript, we recommend you to chain schemas in this order:

- Type validations (like `number()`, `string()`, etc...)
- Schema specific validations (like `filled()`, `union()`, etc...)
- Nullability validations (`defined()` and `nullable()`)
- Test and Transform validations (`test()` and `transform()`)

If you follow what auto-complete presents to you, you should be fine.

### Error messages:

Most of these schemas and their methods (except `transform`) have a last parameter that allows you to set a customized error message for when the value fails to meet the conditions.

You can also customize the default error messages by using the `DefaultErrorMessagesManager` in `error-messages/default-messages/default-error-messages-manager`.

### Union typed schemas:

```typescript
/*
  schema will output
  { common: string } & ({ a: "a"; c: number } | { a: "b"; d: boolean })

  `as const` statements are needed to infer the literal value (like 'a' | 'b')
  instead of a generic value like `string`
*/
const schema = object({
  common: equals(["common"]).defined(),
  a: equals(["a", "b"] as const).defined(),
})
  .union((v) => {
    if (v.a === "a") {
      return {
        a: equals(["a"] as const).defined(),
        c: equals([0]).defined(),
      };
    } else {
      return {
        a: equals(["b"] as const).defined(),
        d: equals([false]).defined(),
      };
    }
  })
  .defined();
```

### Type utilities (at `not-me/lib/schemas/schema`):

- **`InferType<typeof schema>`**: get the output type of a schema
- **`Schema<T>`**: dictates that a value is a schema that has an output type of `T`

### Creating a schema of my own:

Just extend the class of the closest schema there is for your type of value, and call the `transform()` and `test()` methods in your new schema to setup the validation logic that will be run. Can be either in it's _constructor_, or you can add new methods to your schema.

- Here's how an Integer Schema could be implemented:

```typescript
import { NumberSchema } from "not-me/lib/schemas/number/number-schema";

class IntegerSchema extends NumberSchema {
  constructor(message?: string) {
    super();

    this.test(
      (input) => Number.isInteger(input),
      message || "Input is not an integer"
    );
  }
}

/*
  Just a wrapper function so you don't have to write `new IntegerSchema()`.
  It's more readable if you just call `integer()` inside a complex schema.
*/
export function integer(message?: string) {
  return new IntegerSchema(message);
}
```

### Library resolvers / adapters:

This package includes validation resolvers to work with the following libraries / frameworks:

- [Formik](#formik)
- [Nest](#nest)

#### <a name="formik"></a> Formik

```
$ npm install not-me-resolver-formik
```

```tsx
import { formikResolver } from "not-me-resolver-formik";

// (...)

<Formik /* (...) */ validate={formikResolver(notMeSchema)}>
  {/* (...) */}
</Formik>;
```

If you plan on doing a custom `validate` function, `not-me-resolver-formik` also exports `messagesTreeToFormikErrors`, which transforms _Not-Me_ error message trees into _Formik_ errors.

#### <a name="nest"></a> Nest

By integrating this resolver with your NestJS project, parameters annotated with `@Param`, `@Query` and `@Body` will be validated by `not-me`. The parameters need to be typed as **ES6 classes** and annotated with the `@ValidationSchema` decorator, in order to get the validation schema working throught reflection.

```
$ npm install not-me-resolver-nestjs
```

- `app.ts`

  ```typescript
  import { Module, OnApplicationShutdown } from "@nestjs/common";
  import { APP_PIPE } from "@nestjs/core";
  import { NotMeValidationPipe } from "not-me-resolver-nestjs";

  @Module({
    providers: [
      {
        provide: APP_PIPE,
        useClass: NotMeValidationPipe,
      },
    ],
  })
  export class AppModule {}
  ```

- `any-data.dto.ts`

  > Tie both the DTO class and the schema type by either **typing the schema with the DTO class** or **implementing the schema's inferred type**

  - **Typing the schema with the DTO class**

    > **(RECOMMENDED) The schema is allowed to have more properties** than the class. This method guarantees that all values specified in the class are present the final validated object. When using this method, **the class should be the first place you go to add, change or remove properties**.

    ```typescript
    import { object } from "not-me/lib/schemas/object/object-schema";
    import { string } from "not-me/lib/schemas/string/string-schema";
    import { Schema } from "not-me/lib/schemas/schema";
    import { ValidationSchema } from "not-me-resolver-nestjs";

    const schema: Schema<AnyDataDTO> = object({
      field: string().defined(),
    });

    @ValidationSchema(schema)
    export class AnyDataDTO {
      field: string;
    }
    ```

  - **Implementing the schema's inferred type**

    > Some use cases might require **the class to have more properties** than the schema. In those cases, we recommend **implementing the schema's inferred type**. When using this method, **the schema should be the first place you go to add, change or remove properties**.

    ```typescript
    import { object } from "not-me/lib/schemas/object/object-schema";
    import { string } from "not-me/lib/schemas/string/string-schema";
    import { InferType } from "not-me/lib/schemas/schema";
    import { ValidationSchema } from "not-me-resolver-nestjs";

    const schema = object({
      field: string().defined(),
    });

    @ValidationSchema(schema)
    export class AnyDataDTO implements InferType<typeof schema> {
      field: string;
    }
    ```

### How it works under the hood:

When you set up a schema, you're just pilling up filter functions that will test and transform your initial value. These are the types of filters that are called during validation, by this order:

- **Nullability checks** will check if the value is `undefined` or `null` and whether or not they're allowed
- **Type filter** will validate if your input is in a specific type (example: a number, an object, an array, etc...)
- **Shape filters** will validate the fields in your value. This only applies to object and array values
- **Test and Transform filters** will run basic _true_ or _false_ checks on your value, or transform your value.
