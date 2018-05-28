

The following is an elegant piece of code which implements removing a file or directory.  Should the directory not be
empty then it will remove that directories' contents before removing the directory itself.

This code is elegant as:

- It is asynchronous returning a `promise` to remove the file, and
- It removes a directories' contents in parallel

``` JavaScript
const removeAll = name =>
    isDirectory(name)
        .then(exists =>
            exists
                ? readdir(name)
                    .then(dirs => Promise
                        .all(dirs.map(n => Path.resolve(name, n)).map(removeAll))
                        .then(() => rmdir(name)))
                : unlink(name));
```

Part of this code's simplicity is that it is supported by a number of libraries containing pre-built functions:

- `isDirectory`, `readdir`, `rmdir` and `unlink` have all been built and are available in a promises style,
- `Path` to compose and destruct director and file names,
- lists have the method `map`, and
- promises have the methods `then` and `all`

To rewrite this in an `Actor I` style it is necessary to first define a number of supporting types and actors which
can then be composed together to create this specific function.

``` haskell
import Actors
import Actors.Result
import System.IO.FileSystem as FS


removeAll : String -> Actors.Result FS.Error () -> Actors.Msg
removeAll name recipient =
	FS.isDirectory name
    [|
      apply self state isDirectory =
        if isDirectory then
          ( state
          , FS.readdir name
              [|
                okay self state dirs =
                  ( state
                  , Actors.Result.all (List.map removeAll dirs) (Actors.Result.then (\() -> FS.rmdir name) recipient)
                  )

                error self state error =
                  ( state
                  , recipient.error error)
              |]
        else
          ( state
          , FS.unlink name recipient
          )
    |]
```
