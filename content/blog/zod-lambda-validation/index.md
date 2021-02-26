---
title:  Validating typescript Lambda input with Zod
date:   2021-02-16 22:32:38 -0800
categories: zod lambda typescript validation
---
![Zod](./zod.svg)
It's common for lambdas to be triggered via API Gateway, but SNS, SQS, etc will all feed lambdas with strings. When you're writing lambdas that take in JSON string parameters, you're going to want to validate the input and convert it into a first class statically-typed object as soon as possible - after all, that's why we use typescript, right? 

Since typescript is a type-safe language (by definition), using real Typescript types is the way to go here. The best way to define your parameters is as first-class Types in Typescript, and then validate that the string you've been given matches the object type that you've defined. But how?

The way I've validated input like this in the past was via JSON schemas - I would define a schema and use a JSON schema validator like `ajv`. Maybe I'd wrap the lambda in some middleware that would take in the schema and the event, use Middy to do the validation, and have the middleware spit out a validated object (Onica's [sailplane](https://github.com/onicagroup/sailplane) made this easy). But would it be typed? No! Then I'd also have to define a Typescript Type or Typescript Interface with essentially the same information as was in the JSON schema, and cast the object to that type. This is not a great developer experience.

[Zod](https://github.com/colinhacks/zod) is a library designed to make this easy; it lets you define a schema using native Typescript types. You can then ask Zod to validate the input for you and convert it to a first-class Typescript object - the best part is that your IDE's Intellisense can understand it!  Let's look at an example.

Say I have an API Gateway method defined like this:

``` typescript
export const update:AsyncProxyHandler = async event => {
  let commandRequest:unknown = JSON.parse(event.body);
}
```

The problem with this is that we aren't validating the command object. It could be anything!  But then we'd also have to define a Typescript Type or Typescript Interface with essentially the same information. Or generate one from the other. This was not an ideal solution. Instead, we can use Zod to do both the validation _and_ define the type. Like so:

```typescript
import * as z from 'zod';
export const commandRequest = z.object({
    deviceId: z.string(),
    tenantId: z.string()
});
export type CommandRequest = z.infer<typeof commandRequest>;

export const update:AsyncProxyHandler = async event => {
  let json:unknown = JSON.parse(event.body);
  const command = commandRequest.safeParse(json); //command is of type CommandRequest
  if (!parsed.success) {
    console.log(parsed.error);
    return { statusCode: 500, body: { message: parsed.error }};
  }
  return {statusCode: 200};
}
```

Here we used Zod's `safeParse` function that doesn't immediately throw an error if it finds an object doesn't conform to the schema; instead, it returns an object containing the results of the parse attempt. If you just want a valid object of the right type or an exception can use zod's `parse` method instead.

But what if one of your object's fields is optional? No problem: define it as such, like so: `deviceId: z.string().optional()`.

This first example was pretty straight forward, but most real-world applications are not. How about a more interesting use case, where we can employ Zod's discriminated union functionality.

Let's say, instead of an API Gateway event handler, you're writing a handler for an SQS queue. This queue could see several different types of messages, and you want a validator that can handle all of them as first-class Typescript Types. For discussion purposes, let us imagine that your queue contains commands of different types: Create and Delete, which have mostly the same attributes but have a discriminator for the command string.

```typescript
export const baseCommand = z.object({
  deviceId: z.string(),
  tenantId: z.string()
});
export const updateCommand = z.object({
  commandType: z.literal('update');
}).merge(baseCommand);
export type UpdateCommand = z.infer<typeof updateCommand>;
export const deleteCommand = z.object({
  commandType: z.literal('delete');
}).merge(baseCommand);
export type DeleteCommand = z.infer<typeof deleteCommand>;

//Now create a discriminated union of the two commands
export const command = z.union([
  updateCommand,
  deleteCommand
])
export Command = z.infer<typeof command>

export const execute: SQSHandler = async event => {
  const commands = event.Records.map(r => {
    let json: unknown;
    try {
        json = JSON.parse(r.body);
    } catch (e) {
        LOG.error('Failed to parse message', e);
        return [];
    }
    const parsed = zodObject.safeParse(json);
    if(!parsed.success) {
      console.log(parsed.error);
      return;
    }
    return parsed.data;
  });
}
// Now you have a collection of objects that may be of type UpdateCommand or of type DeleteCommand
```

Someone's even created some [middleware](https://github.com/lechodiman/middy-zod-validator) integrating Zod, if you choose to go that route.

We've barely scratched the surface of what Zod is capable of, but I hope that for you this has sparked some possibilities.
