# `run-scripts` and `run-commands` args issue in `NX`

This repo documents a bug with `run-scripts` and `run-commands` where stringification of arguments leads to unexpected results. Specifically, `--verbose` translated to `--verbose=true` for most CLIs will end in breakage. `cp` is used in these examples as a basic example. Other CLIs, such as `tsc` also fail.

## Table of Contents

* [Setup](#setup)
* [`run-scripts` issue](#run-scripts-issue)
* [`run-commands` issue](#run-commands-issue)
* [`run-many` issue](#run-many-issue)


## Setup
1. Clone repo
2. yarn install


## `run-scripts` issue

```bash
yarn nx run foo:build-script --verbose
```

Produces:
```bash
> nx run foo:build-script --verbose

> foo@1.0.0 test-script
> cp ./src/end.js ./src/start.js "--verbose=true"

usage: cp [-R [-H | -L | -P]] [-fi | -n] [-apvXc] source_file target_file
       cp [-R [-H | -L | -P]] [-fi | -n] [-apvXc] source_file ... target_directory

 ————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

 >  NX   Running target "foo:build-script" failed

   Failed tasks:
   
   - foo:build-script
```

Because internally, `nx` transforms `verbose` to `verbose=true`

See related code: https://github.com/nrwl/nx/blob/91ca67ad6b7b711d003980193fb8b3973cdf60f7/packages/workspace/src/executors/run-script/run-script.impl.ts#L19-L21

Problem is that this code adds `=true` to the `--verbose` command which many CLIs do not like:
```ts
const args = [];
Object.keys(options).forEach((r) => {
  args.push(`--${r}=${options[r]}`);
});
```

## `run-commands` issue

```bash
yarn nx run foo:build-commands --verbose
```

Produces:

```bash
> nx run foo:build-commands --verbose

...

> foo@1.0.0 test-commands
> cp ./src/start.js ./src/end.js "true"

usage: cp [-R [-H | -L | -P]] [-fi | -n] [-apvXc] source_file target_file
       cp [-R [-H | -L | -P]] [-fi | -n] [-apvXc] source_file ... target_directory
npm timing command:run-script Completed in 19ms
npm verb exit 64
npm timing npm Completed in 183ms
npm verb code 64
Warning: @nrwl/run-commands command "npm run test-commands --prefix ./packages/foo --verbose=true" exited with non-zero status code
 ——————————————————————

 >  NX   Running target "foo:build-commands" failed
```

Likely due to stringification here:
https://github.com/nrwl/nx/blob/91ca67ad6b7b711d003980193fb8b3973cdf60f7/packages/workspace/src/executors/run-commands/run-commands.impl.ts#L242-L262

```ts
function transformCommand(
  command: string,
  args: { [key: string]: string },
  forwardAllArgs: boolean
) {
  if (command.indexOf('{args.') > -1) {
    const regex = /{args\.([^}]+)}/g;
    return command.replace(regex, (_, group: string) => args[camelCase(group)]);
  } else if (Object.keys(args).length > 0 && forwardAllArgs) {
    const stringifiedArgs = Object.keys(args)
      .map((a) =>
        typeof args[a] === 'string' && args[a].includes(' ')
          ? `--${a}="${args[a].replace(/"/g, '"')}"`
          : `--${a}=${args[a]}`
      )
      .join(' ');
    return `${command} ${stringifiedArgs}`;
  } else {
    return command;
  }
}
```

## `run-many` issue

```bash
yarn nx run-many --target=build-script --all --verbose
```

Will cause this too since `--verbose` is forwarded along