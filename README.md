# ts-formdata

[![npm](https://img.shields.io/npm/v/ts-formdata)](https://www.npmjs.com/package/ts-formdata)
![GitHub top language](https://img.shields.io/github/languages/top/lorefnon/ts-formdata)
![GitHub Workflow Status (with branch)](https://img.shields.io/github/actions/workflow/status/lorefnon/ts-formdata/main.yaml?branch=main)

[FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData) is a convenient way to extract data from forms - however it is not so convenient to use when you want the form payload to be something more complex than key-value pairs. In addition, the values are always strings- and you are responsible to converting them to numbers, dates, booleans etc.

Writing these converters are boring and error-prone - this library helps you by providing you a simple approach to encode nesting and type information in the field names so that they can be easily parsed into nested structures in a type-safe manner.

ts-formdata is framework agnostic - feel free to use it alongside Vanilla JS, React, Svelte etc. on client side or with [formdata-node](https://www.npmjs.com/package/formdata-node) on the server side.

## Installation

```bash
# npm
npm i ts-formdata

# pnpm

pnpm i ts-formdata
# yarn
yarn add ts-formdata
```

## Usage

Let's say our form data looks like this:

```ts
type UserPayload = {
    settings: {
        mode: 'auto' | 'light' | 'dark';
        theme: 'red' | 'green' | 'blue';
    };
    favouriteFrameworks: Array<{
        name: string;
        satisfaction: number;
    }>;
    profile: {
        firstname?: string;
        lastname?: string;
        image?: Blob;
    };
};
```

### Fields

Use the `fields` helper to create your input names:

```ts
import React from 'react';
import { fields, asStr, asNum } from 'ts-formdata';

const f = fields<MyForm>();

// We are using React for illustration here, but ts-formdata
// is framework agnostic
const UserPayloadForm = () => {
    return (
        <form
            onSubmit={(e) => {
                const formData = new FormData(e.currentTarget);
                const {
                    data, // files and fields
                    fields, // fields only
                    files, // files only
                } = extractFormData<MyForm>(formData);
                console.log('data: ', data);
            }}
        >
            <h2>Settings</h2>

            <label>
                <span>Mode</span>
                <select name={asStr(f.settings.mode)}>
                    <options>auto</options>
                    <options>light</options>
                    <options>dark</options>
                </select>
            </label>
            <label>
                <span>Theme</span>
                <select name={asStr(f.settings.theme)}>
                    <options>red</options>
                    <options>green</options>
                    <options>blue</options>
                </select>
            </label>

            <h2>3 favourite Frameworks</h2>

            <input type="text" name={asStr(f.favouriteFrameworks[0].name)} />
            <input
                type="number"
                name={asNum(f.favouriteFrameworks[0].satisfaction)}
            />

            <input type="text" name={asStr(f.favouriteFrameworks[1].name)} />
            <input
                type="number"
                name={asNum(f.favouriteFrameworks[1].satisfaction)}
            />

            <h2>User</h2>

            <input type="text" name={asStr(f.profile.firstname)} />
            <input type="text" name={asStr(f.profile.lastname)} />
            <input type="file" name={asStr(f.profile.image)} />
        </form>
    );
};
```

What is happening under the hood is:

-   `fields()` returns a proxy that will create a string
    -   So: `f.user.lastname.toString()` will be a string `"user.lastname"`
-   asStr, asNum etc. are encoders that append type information to this string
    -   So: `asStr(f.user.lastname)` resolves to a string `"user.lastname:string"`
-   We use this string as a name of the field.
-   Our `extractFormData` function is aware of this path syntax and type suffix and is able to derive the nested structure with appropriately coerced values from the key-value pairs we get in the FormData object.

**Type safety:**

-   Attempting to construct paths that don't match the keys in payload (eg. `f.setting.mode`) will result in a type error.
-   Attempting to use a type encoder with field of different type (eg. `asNum(f.user.firstname)`) is also a type error.

#### Nested paths

For nested objects simply chain the keys:

```ts
fields<MyForm>().user.firstname;
> 'user.firstname'
```

For arrays you have to call the function:

```ts
fields<MyForm>().favouriteFrameworks();
> 'favouriteFrameworks[]'
```

**Note:**
Only use primite values inside arrays if you don´t provide an index!

_Bad:_

```ts
fields<MyForm>().favouriteFrameworks().key;
> 'favouriteFrameworks[].key'
```

_Good:_

```ts
fields<MyForm>().favouriteFrameworks(1).key;
> 'favouriteFrameworks[1].key'
```

Simply pass in the index:

```ts
fields<MyForm>().favouriteFrameworks(2);
> 'favouriteFrameworks[2]'
```

It´s also possible to create objects in arrays

```ts
fields<MyForm>().favouriteFrameworks(2).satisfaction;
> 'favouriteFrameworks[2].satisfaction'
```

### extractFormData

`extractFormData` pulls out the data from a `FormData` object where `fields` are typed as string and `files` are typed as Blob:

**Note:**
Browsers send empty inputs as an empty string or file. `extractFormData` omits the values.

```ts
import { extractFormData } from 'ts-formdata';

export async function handlePost(request: Request) {
    const formData = await request.formData();

    const {
        data, // files and fields
        fields, // fields only
        files, // files only
    } = extractFormData<MyForm>(formData);

    // data:
    type Data = {
        settings?: {
            mode?: string;
            theme?: string;
        };
        favouriteFrameworks?: Array<{
            name?: string;
            satisfaction?: string;
        }>;
        user: {
            firstname?: string;
            lastname?: string;
            image?: Blob;
        };
    };

    // fields:
    type Fields = {
        settings?: {
            mode?: string;
            theme?: string;
        };
        favouriteFrameworks?: Array<{
            name?: string;
            satisfaction?: string;
        }>;
        user: {
            firstname?: string;
            lastname?: string;
        };
    };

    // files:
    type Files = {
        user: {
            image?: Blob;
        };
    };
}
```

### Custom Codecs

If the provided converters asStr, asNum etc. are not adequate for you, you can provide custom codecs to the library which will be used for encoding of names and extraction of values.

For example if you want boolean values to be represented as 1/0 values, you can implement a codec as follows:

```
import { BaseCodec } from "ts-formdata";

export class ShortBoolCodec extends BaseCodec<boolean> {
    constructor() {
        super('sbool') // <-- Type suffix used in name
    }
    decodeValue(value: FormDataEntryValue) {
        return value === '1';
    }
}

const shortBool = new ShortBoolCodec();
```

Now in your form:

```
<input type="text" pattern="(1|0)" name={shortBool.encode(f.mood.isHappy)} />
```

While extracting you need to pass this codec to `extractFormData`:

```
extractFormData(new FormData(e.currentTarget), [shortBool])
```

### Multistep wizards

You can merge data from multiple forms by providing an accumulator to extractFormData.

```ts
const extracted: Partial<Payload> = {};
extractFormData(formData, [], extracted);
// Later:
extractFormData(formData, [], extracted);
```

`extracted` will continue to accumulate the extracted fields in each `extractFormData` invocation.

This is convenient for example, in wizards where each step needs to be separately validated on step submission but data is saved to server only after the final step.

## Caveats

### Missing fields

If a certain field is missing in the rendered form, it will not be present in the form data. However we are not able to validate this at compile time.

This is not a big problem in practice because these errors are easily caught during preliminary testing. However, if you do want to safeguard against partial data you will need to use a validation library like zod to validate the extracted data.

You also need to take care to not use conditional rendering for form fields, as only the fields which are currently present in DOM will be extracted. So it is a better practice to structure dynamic forms such that any fields that are hidden from view are hidden through css rather than removed from DOM.

## License

[MIT](https://github.com/lorefnon/ts-formdata/blob/main/LICENSE)
