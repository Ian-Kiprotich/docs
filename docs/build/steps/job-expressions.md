---
title: Write Job expressions
sidebar_label: Write Jobs
---

To define the business logic and data transformation rules or logic for
individual `Steps` in your workflow, you will need to write a `Job`. This
article will provide a basic overview of Job expressions & writing tips.

:::tip

For example Jobs written by the OpenFn core team and other users, check out the
[Library](/adaptors/library) or other project repositories under
[Github.com/OpenFn](https://github.com/OpenFn).

:::

## About Jobs

A `Job` is evaluated as a JavaScript expression and primarily defines the
specific series of [Operations](/docs/build/steps/operations.md) (think: tasks,
database actions, custom functions) to be performed in a specific Workflow Step.

In most cases, a Job is a series of `create` or `upsert` operations that are
triggered by a webhook event (e.g., form submission forwarded from ODK mobile
data collection app) or cron (e.g., daily @ 12:00). See this basic example:

```js
create(
  'Patient__c',
  fields(
    field('Name', dataValue('form.surname')),
    field('Other Names', dataValue('form.firstName')),
    field('Age__c', dataValue('form.ageInYears')),
    field('Is_Enrolled__c', true),
    field('Enrollment_Status__c', 3)
  )
);
```

This would create a new `Patient__c` record in the connected app. The patient's
`Name` will be mapped from the form submission forwarded to OpenFn from the
source mobile data collection app via a webhook request. Then, because we assume
that all Patients registered in the mobile app are always considered "enrolled",
we hard code the patient enrollment status as true - see the mapping for
`Is_Enrolled__c`.

The functions see above is OpenFn's own syntax, and you've got access to dozens
of common "helper functions" like `dataValue(path)` and destination specific
functions like `create(object, attributes)`. You can choose to either write jobs
using this OpenFn syntax or write your own custom, anonymous functions in
JavaScript to do whatever your heart desires. Remember that **Steps are
evaluated as JavaScript**.

### dataValue

The most commonly used "helper function" is `dataValue(...)`. This function
takes a single argument—the _path_ to some data that you're trying to access
inside the message that has triggered a particular run. In the above example,
you'll notice that `Is_Enrolled__c` is _always_ set to `true`, but `Name` will
change for each message that triggers the running of this step. It's set to
`dataValue('form.surname')` which means it will set `Name` to whatever value is
present at `state.data.form.surname` for the triggering webhook request. It
might be Bob for one message, and Alice for another.

:::note

Note that for message-triggered steps, `state` will always have it's `data` key
(i.e., `state.data`) set to the body of the triggering webhook request (aka HTTP
request).

I.e., `dataValue('some.path') === state.data.some.path`, as evaluated at the
time that the operation (`create` in the above expression) is executed.

:::

### A Job with custom JavaScript

To write your own custom JavaScript functions, simply add an `fn(...)` block to
your code as below.

```js
fn(state => {
  //write your own function to manipulate/transform state
  return state;
});
```

Alternatively, you can add custom JavaScript code in-line any Adaptor-specific
functions. See example job below where JavaScript was added to transform the
data value outputted for `Name`.

```js
create(
  'Patient__c',
  fields(
    field('Name', state => {
      console.log('Manipulate state to get your desired output.');
      return Array.apply(null, state.data.form.names).join(', ');
    }),
    field('Age__c', 7)
  )
);
```

Here, the patient's name will be a comma separated concatenation of all the
values in the `patient_names` array from our source message.

## Available JavaScript Globals

For security reasons, users start with access to the following standard
JavaScript globals, and can request more by opening an issue on Github:

- [`Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [`console`](https://nodejs.org/api/console.html)
- [`JSON`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON)
- [`Number`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)
- [`Promise`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [`String`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)

## Examples of adaptor-specific functions

**N.B.: This is just a sample.** There are lots more available in the
[Adaptors docs](/adaptors/) and
[repository](https://github.com/OpenFn/adaptors).

### language-common

These are available functions "common" across Adaptors. You can use any of these
when writing Job expressions.

- `field('destination_field_name__c', 'value')` Returns a key, value pair in an
  array.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L364)
- `fields(list_of_fields)` zips key value pairs into an object.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L377)
- `dataValue('JSON_path')` Picks out a single value from source data.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L146)
- `each(JSON_path, operation(...))` Scopes an array of data based on a JSONPath
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L262).
  See beta.each when using multiple each()'s in an expression.
- `each(merge(dataPath("CHILD_ARRAY[*]"),fields(field("metaId", dataValue("*meta-instance-id*")),field("parentId", lastReferenceValue("id")))), create(...))`
  merges data into an array then creates for each item in the array
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L396)
- `lastReferenceValue('id')` gets the sfID of the last item created
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/common/src/Adaptor.js#L175)
- `fn(state){return state.references[state.references.length-N].id})` gets the
  sfID of the nth item created

#### each()

For more on the `each(...)` operation, see
[this page](/documentation/next/build/steps/each) and the below example.

```js
each(
  dataPath('csvData[*]'),
  upsertTEI(
    'aX5hD4qUpRW', //piirs uid
    {
      trackedEntityType: 'bsDL4dvl2ni',
      orgUnit: dataValue('OrgUnit'),
      attributes: [
        {
          attribute: 'aX5hD4qUpRW',
          value: dataValue('aX5hD4qUpRW'),
        },
        {
          attribute: 'MxQPuS9G7hh',
          value: dataValue('MxQPuS9G7hh'),
        },
      ],
    },
    { strict: false }
  )
);
```

### Salesforce

See below for some example functions for the Salesforce Adaptor. See the
[Adaptor](/adaptors/) for more.

- `create("DEST_OBJECT_NAME__C", fields(...))` Create a new object. Takes 2
  parameters: An object and attributes.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/salesforce/src/Adaptor.js#L466-L480)
- `upsert("DEST_OBJECT_NAME__C", "DEST_OBJECT_EXTERNAL_ID__C", fields(...))`
  Creates or updates an object. Takes 3 paraneters: An object, an ID field and
  attributes.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/salesforce/src/Adaptor.js#L539-L560)
- `relationship("DEST_RELATIONSHIP_NAME__r", "EXTERNAL_ID_ON_RELATED_OBJECT__C", "SOURCE_DATA_OR_VALUE")`
  Adds a lookup or 'dome insert' to a record.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/salesforce/src/Adaptor.js#L49-L56)

### dhis2

See below for some example functions for the DHIS2 Adaptor. See the
[Adaptor](/adaptors/) for more.

- `create('events', {..})` Creates an event.
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/dhis2/src/Adaptor.js#L137-L142)
- `create('dataValueSet', {..})` Send data values using the dataValueSets
  resource
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/dhis2/src/Adaptor.js#L185-L191)

### OpenMRS

See below for some example functions for the OpenMRS Adaptor. See the
[Adaptor](/adaptors/) for more.

- `create('person',{..})` Takes a payload of data to create a person
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/openmrs/src/Adaptor.js#L292-L311)
- `createPatient(...)` Takes a payload of data to create a patient
  [(source)](https://github.com/OpenFn/adaptors/blob/main/packages/openmrs/src/Adaptor.js#L292-L311)

## Snippets and samples

Below you can find some examples of block code for different functions and data
handling contexts.

<details>
<summary><b>Step expression (for commcare to SF)</b></summary>

The following step expression will take a matching receipt and use data from
that receipt to upsert a `Patient__c` record in Salesforce and create multiple
new `Patient_Visit__c` (child to Patient) records.

```js
upsert(
  'Patient__c',
  'Patient_Id__c',
  fields(
    field('Patient_Id__c', dataValue('form.patient_ID')),
    relationship('Nurse__r', 'Nurse_ID_code__c', dataValue('form.staff_id')),
    field('Phone_Number__c', dataValue('form.mobile_phone'))
  )
),
  each(
    join('$.data.form.visits[*]', '$.references[0].id', 'Id'),
    create(
      'Visit__c',
      fields(
        field('Patient__c', dataValue('Id')),
        field('Date__c', dataValue('date')),
        field('Reason__c', dataValue('why_did_they_see_doctor'))
      )
    )
  );
```

</details>

<details>
<summary><b>Accessing the "data array" in Open Data Kit submissions</b></summary>

Notice how we use "each" to get data from each item inside the "data array" in
ODK.

```js
each(
  '$.data.data[*]',
  create(
    'ODK_Submission__c',
    fields(
      field('Site_School_ID_Number__c', dataValue('school')),
      field('Date_Completed__c', dataValue('date')),
      field('comments__c', dataValue('comments')),
      field('ODK_Key__c', dataValue('*meta-instance-id*'))
    )
  )
);
```

</details>

<details>
<summary><b>ODK to Salesforce: create parent record with many children from parent data</b></summary>

Here, the user brings `time_end` and `parentId` onto the line items from the
parent object.

```js
each(
  dataPath('data[*]'),
  combine(
    create(
      'transaction__c',
      fields(
        field('Transaction_Date__c', dataValue('today')),
        relationship(
          'Person_Responsible__r',
          'Staff_ID_Code__c',
          dataValue('person_code')
        ),
        field('metainstanceid__c', dataValue('*meta-instance-id*'))
      )
    ),
    each(
      merge(
        dataPath('line_items[*]'),
        fields(
          field('end', dataValue('time_end')),
          field('parentId', lastReferenceValue('id'))
        )
      ),
      create(
        'line_item__c',
        fields(
          field('transaction__c', dataValue('parentId')),
          field('Barcode__c', dataValue('product_barcode')),
          field('ODK_Form_Completed__c', dataValue('end'))
        )
      )
    )
  )
);
```

</details>

<details>
<summary><b>Salesforce: perform an update</b></summary>

```js
update("Patient__c", fields(
  field("Id", dataValue("pathToSalesforceId")),
  field("Name__c", dataValue("patient.first_name")),
  field(...)
));
```

</details>

<details>
<summary><b>Salesforce: Set record type using 'relationship(...)'</b></summary>

```js
create(
  'custom_obj__c',
  fields(
    relationship(
      'RecordType',
      'name',
      dataValue('submission_type'),
      field('name', dataValue('Name'))
    )
  )
);
```

</details>

<details>
<summary><b>Salesforce: Set record type using record Type ID</b></summary>

```js
each(
  '$.data.data[*]',
  create(
    'fancy_object__c',
    fields(
      field('RecordTypeId', '012110000008s19'),
      field('site_size', dataValue('size'))
    )
  )
);
```

</details>

<details>
<summary><b>Telerivet: Send SMS based on Salesforce workflow alert</b></summary>

```js
send(
  fields(
    field(
      'to_number',
      dataValue(
        'Envelope.Body.notifications.Notification.sObject.phone_number__c'
      )
    ),
    field('message_type', 'sms'),
    field('route_id', ''),
    field('content', function (state) {
      return 'Hey there. Your name is '.concat(
        dataValue('Envelope.Body.notifications.Notification.sObject.name__c')(
          state
        ),
        '.'
      );
    })
  )
);
```

</details>

<details>
<summary><b>Sample DHIS2 events API step</b></summary>

```js
event(
  fields(
    field('program', 'eBAyeGv0exc'),
    field('orgUnit', 'DiszpKrYNg8'),
    field('eventDate', dataValue('properties.date')),
    field('status', 'COMPLETED'),
    field('storedBy', 'admin'),
    field('coordinate', {
      latitude: '59.8',
      longitude: '10.9',
    }),
    field('dataValues', function (state) {
      return [
        {
          dataElement: 'qrur9Dvnyt5',
          value: dataValue('properties.prop_a')(state),
        },
        {
          dataElement: 'oZg33kd9taw',
          value: dataValue('properties.prop_b')(state),
        },
        {
          dataElement: 'msodh3rEMJa',
          value: dataValue('properties.prop_c')(state),
        },
      ];
    })
  )
);
```

</details>

<details>
<summary><b>merge many values into a child path</b></summary>

```js
each(
  merge(
    dataPath("CHILD_ARRAY[*]"),
    fields(
      field("metaId", dataValue("*meta-instance-id*")),
      field("parentId", lastReferenceValue("id"))
    )
  ),
  create(...)
)
```

</details>

<details>
<summary><b>arrayToString</b></summary>

```js
arrayToString(arr, separator_string);
```

</details>

<details>
<summary><b>access an image URL from an ODK submission</b></summary>

```js
// In ODK the image URL is inside an image object...
field("Photo_URL_text__c", dataValue("image.url")),
```

</details>

<details>
<summary><b>Use external ID fields for relationships during a bulk load in Salesforce</b></summary>

```js
fn(state) => {
  const patients = array.map(item => {
    return {
      Patient_Name__c: item.fullName,
      'Account.Account_External_ID__c': item.account
      'Clinic__r.Unique_Clinic_Identifier__c': item.clinicId,
      'RecordType.Name': item.type,
    };
  });

  return {...state, patients}
}
```

</details>

<details>
<summary><b>Bulk upsert with an external ID in salesforce</b></summary>

```js
bulk(
  'Visit_new__c',
  'upsert',
  {
    extIdField: 'commcare_case_id__c',
    failOnError: true,
    allowNoOp: true,
  },
  dataValue('patients')
);
```

</details>

## Anonymous Functions

Different to [Named Functions](#examples-of-adaptor-specific-functions),
Anonymous functions are generic pieces of JavaScript which you can write to suit
your needs. Read on for some examples of these custom functions.

### Custom replacer

```js
field('destination__c', state => {
  console.log(something);
  return dataValue('path_to_data')(state).toString().replace('cats', 'dogs');
});
```

This will replace all "cats" with "dogs" in the string that lives at
`path_to_data`.

> **NOTE:** The JavaScript `replace()` function only replaces the first instance
> of whatever argument you specify. If you're looking for a way to replace all
> instances, we suggest you use a regex like we did in the
> [example](#custom-concatenation-of-null-values) below.

### Custom arrayToString

```js
field("target_specie_list__c", function(state) {
  return Array.apply(
    null, sourceValue("$.data.target_specie_list")(state)
  ).join(', ')
}),
```

It will take an array, and concatenate each item into a string with a ", "
separator.

### Custom concatenation

```js
field('ODK_Key__c', function (state) {
  return dataValue('metaId')(state).concat('(', dataValue('index')(state), ')');
});
```

This will concatenate two values.

### Concatenation of null values

This will concatenate many values, even if one or more are null, writing them to
a field called Main_Office_City_c.

```js
...
  field("Main_Office_City__c", function(state) {
    return arrayToString([
      dataValue("Main_Office_City_a")(state) === null ? "" : dataValue("Main_Office_City_a")(state).toString().replace(/-/g, " "),
      dataValue("Main_Office_City_b")(state) === null ? "" : dataValue("Main_Office_City_b")(state).toString().replace(/-/g, " "),
      dataValue("Main_Office_City_c")(state) === null ? "" : dataValue("Main_Office_City_c")(state).toString().replace(/-/g, " "),
      dataValue("Main_Office_City_d")(state) === null ? "" : dataValue("Main_Office_City_d")(state).toString().replace(/-/g, " "),
    ].filter(Boolean), ',')
  })
```

> Notice how this custom function makes use of the **regex** `/-/g` to ensure
> that all instances are accounted for (g = global search).

### Custom Nth reference ID

If you ever want to retrieve the FIRST object you created, or the SECOND, or the
Nth, for that matter, a function like this will do the trick.

```js
field('parent__c', function (state) {
  return state.references[state.references.length - 1].id;
});
```

See how instead of taking the id of the "last" thing that was created in
Salesforce, you're taking the id of the 1st thing, or 2nd thing if you replace
"length-1" with "length-2".

### Convert date string to standard ISO date for Salesforce

```js
field('Payment_Date__c', function (state) {
  return new Date(dataValue('payment_date')(state)).toISOString();
});
```

> **NOTE**: The output of this function will always be formatted according to
> GMT time-zone.