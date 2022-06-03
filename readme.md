# ts-configuration-modules

## goal

Create something similar to the NixOS module system, but for TypeScript and in a type-safe manner.

## why?

The way that the NixOS module system works and makes it easy to configure an entire operating system is brillant!
But without type-safety and auto-completion from the editor, it's also very frustrating.
There are efforts in the Nix-Community to create a statically-typed language to change this. It's called Nickel.
But to me, TypeScript seems to be what they basically are looking for. So I tried to verify, if it could be used to create a type-safe version of the system.

## how does it work

Like with the NixOS module system, modules define options and use the values that are passed into these options to pass values to options of other modules.
So for example:

- the "system"-module would define, that "app" is enabled.
- the "app"-module would define, that "nginx" is enabled, and that it should serve files from /path/to/app on http://my.app
- the "nginx"-module would define, that there's a /etc/nginx/nginx.conf file, containing the configuration needed to serve the apps files
- the "files"-module would then evaulate to all the files that need to be written to disk

## defining a module

A `Module` consists of the following things:

- the type of the options the module takes
- the default value for that type
- a list of `ModuleEffect`s this module has on other modules

A `ModuleEffect` consists of the following:

- the target-module, it has an effect on
- a function, transforming the target-modules current configuration into a new configuration, using the options passed to the parent module

So, assuming theres a low-level "files" module, which we can use to define the files and their contents on the system, in the following format:

```ts
{
    "/path/to/file1":"content of file 1",
    "/path/to/file2":"content of file 2"
}
```

And we want to create a module, that takes a list of users, and transforms it into a /etc/passwd file, it would look like the following:

```ts
import { files } from "files-module";
import { module } from "ts-configuration-modules";

type User = {
  name: string;
  id: string;
  groupId: string;
};

export const users = module<User[]>({
  // between <...> is the type that the module takes as options
  name: "users",
  initialInput: [], // by default, there are no users. Must satisfy the type specified above.
  effects: [
    mixinEffect(
      // creates an effect, that merges the object returned by the function below into the configuration of the files module
      files, // the target module, this effect has an effect on
      (
        users // all the users, after all parent module's effects on this module were evaluated
      ) =>
        users.length > 0 && {
          // only create the file if we have at least one user
          "/etc/passwd": users //build the content of the /etc/passwd files from the user array
            .map(
              ({ name, id, groupId }) =>
                `${name}::${id}:${groupId}::/home/${name}/:/bin/bash`
            )
            .join("\n"),
        }
    ),
  ],
});
```
