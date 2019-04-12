Once again, it's been a while since I've posted anything here. One reason is just that I've spent
a lot of time in JavaScript / TypeScript land over the last year, so haven't had to deal much with
.net stuffs -- well, at least I haven't had to deal much with new things that I need done in that
realm, though I have been grateful for the PeanutButtery goodness which I could use in a dotnet core
asp.net project, namely:

- Utils still manages to make my workflow simpler, especially with functional-style extensions like `.ForEach` and `.ForEachAsync`, not to mention the useful `.SelectAsync`
- I was _so_ pleased to find that `PeanutButter.DuckTyping` delivered on my need to translate keys
  in an appsettings.json file into a read-only `IAppSettings` object I could pass around. I'd forgotten
  that I'd added the facility to interpret string values as enums on the desired interface, and had
  a real moment of joy when it "just worked". More often than not, when looking back at code I've
  written in the past, it's just something I can learn from -- I could do it so much better now, and
  often older code of mine just seems to be _lacking_. `PeanutButter.DuckTyping` was challenging (but
  rewarding) to build, and still continues to deliver value today. If you haven't already checked it
  out -- [check out the tests for examples of how to use it](https://github.com/fluffynuts/PeanutButter/tree/master/source/Utils/PeanutButter.DuckTyping.Tests).


Anyway, on with the story!

Not too long ago, I added a MySql variant to the `PeanutButter.TempDb` family. It literally spins up
a `mysqld` process on a high, random port, and, when disposed, chucks that away. You can tweak the
server settings before startup, create and switch to schemas, and data is put in your machine's
temp folder, so it's also (hopefully) chucked out at some point.

Now `PeanutButter.TempDb.MySql` had a `netstandard2.0` target framework set -- and the package was
produced with the `netstandard2.0`-targeting assembly, but I hadn't had much cause to test
with that particular target until recently.

Obviously one of the requirements is that that library has is to be able to find a
functional `mysqld` binary on your system. Whilst I allowed for specifying the binary directly,
on Windows, I was allowing zero-conf by querying installed services -- which worked out well for the
places it was being used.

The problem is that, of course, OSX and Linux does not have anywhere near the same service
framework as Windows (not a bad thing at all). It also turns out that the service control
classes available to `netstandard2.0`, even on Windows, are unable to give the same level of
information (specifically, the location of the service binary) as under `net452`
and higher. Something had to be done!

### Strategy #1: search the PATH environment variable
On the OSX machine that I had access to, `mysqld` exists in /usr/local/bin (symlinked to somewhere
deep in the bowels of homebrew territory), so if we could just find it in the `PATH`, we have another
zero-to-low-conf method for finding mysqld.

So the static class `Find` was introduced to `PeanutButter.Utils` -- at the moment, it just has the `InPath` method, which, when given a string like `mysql`, will search all folders mentioned in the
`PATH` variable for `mysql` -- and on Windows, will also search for all matches which would be
executable according to `PATHEXT` -- so `mysql.bat`, `mysql.com`, `mysql.exe`, etc. For example:

```csharp
var mysqld = Find.InPath("mysql");
// on windows, this should produce something like C:\MySql57\bin\mysqld.exe,
//    assuming C:\MySql57\bin were in the path
// on OSX, this gives /usr/local/bin/mysqld
```

On Windows, this _definitely_ gets you something executable. On OSX, I was afraid that a rogue `mysql`
file could end up in the path, and not be actually executable. Remote, sure, but possible. I needed
to get file permissions.

Allegedly there are library functions within `Mono` for this. Allegedly,
I can access ported variants from `netstandard2.0`. In practice, I could install
the relevant package, but only  namespaces were available to me.

"It would be really convenient", I thought,
"if I could just read the output from `ls -l /usr/local/bin/mysql`". We'll get back to
this idea in a bit...

### Strategy #2: there's a cli app for that!
Meanwhile, I found that on Windows, I could use the `sc` command to (eventually)
determine the location of the mysqld binary -- meaning I didn't have to rely on
whether or not the available .net libraries could tell, if
I could just do some good old piping from stdout.

So another useful class was born: `PeanutButter.Utils.ProcessIO`. It's an `IDisposable` wrapper around
a process and its stdout and stderr, which can be quite easily consumed:
```csharp
// query the sc utility to find the specific mysql service installed on the current machine
using (var io = nw ProcessIO("sc", "query"))
{
    if (!io.Started)
    {
        // process never started, we can find out why with the StartException
        Console.WriteLine("Process start fails!");
        Console.WriteLine(io.StartException.Message);
        return null; // nothing to report!
    }
    // `sc query` will produce blocks like:
    // SERVICE_NAME: MySQL57   <- this is the bit of interest right now
    // DISPLAY_NAME: MySQL57
    //     TYPE               : 10  WIN32_OWN_PROCESS
    //     STATE              : 4  RUNNING
    //                             (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
    //     WIN32_EXIT_CODE    : 0  (0x0)
    //     SERVICE_EXIT_CODE  : 0  (0x0)
    //     CHECKPOINT         : 0x0
    //     WAIT_HINT          : 0x0
    return io.StandardOutput
        .FirstOrDefault(
            line =>
            {

                var lower = line.ToLower();
                return lower.StartsWith("service_name") &&
                        lower.Contains("mysql")
            })?.Split(':').Last().Trim();
```

(the above is just the first step in ascertaining the binary being used to run a
mysql windows service, but it serves to illustrate the point)

What does `ProcessIO` grant you?
- `IEnumerable<string>` wrappers around stderr and stdout
- process cleanup after you're done
- using `yield return` on lines from `StandardOutput` and `StandardError`, you're only consuming the lines which have become available -- so if you're invoking a command with literally thousands of lines of output and have cause to exit early, chances are the entire output won't be created. I say "chances are", because the process should continue to produce output until the stdout buffer fills, and then become blocked, which may well exceed your read position. Still, when a
`ProcessIO` is disposed, if the associated process hasn't yet exited, it is forcefully killed.
- `ProcessIO` also exposes the original `Process` as a property, so you can
interact with it's `StandardInput` if that's on your radar.

So now you have literally _no excuse_ for not being a good, unix-ey consumer (: In addition, you can test against MySql on Windows, OSX, and even Linux with impunity.

Happy hacking (: