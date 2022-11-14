# ExUnit under the hood

How many times a day do we type the command `mix test`? How much time do we spend leaning back in our chairs, watching how lazily (they are always lazy!) our tests perform? Nowadays, covering code with tests is an important part of software development. Development without testing is like developing a hazardous chemical compound without a gas mask and a protective suit.

Have you ever wondered how ExUnit runs tests? A bit of a rhetorical question, because we always perceive ExUnit as a tool for dirty work: it should just work. Most of the time we don't even keep our code clean in tests.
<!-- The phrase "curiosity killed the cat" means "be careful with curiosity". I think the opposite applies here. -->
But anyway, curiosity killed the cat, so I decided to take a look at how it works under the hood.

## What tests are

What do we know about tests in ExUnit? 

1. There are usually a lot of test files. Of course, you can keep all of your tests inside one file. But we are on the Light Side, right?
2. Tests can run asynchronously. There are nuances here. Sometimes async tests start interfering with each other.
3. Tests can run sequentially. 
4. Tests finish with a status: successful or not.
5. Tests print out a report when they're done with their work.
6. We have to start tests.

Doesn’t it remind you of something? We have a certain set of scripted tasks, and we must process them as quickly as possible. Each task must report on the results of its work, and as a result, a general report must be generated. Yep, this is a pool of workers. Honestly, I myself was surprised at how simply everything is arranged.

Let’s find out how ExUnit launches, runs, and finishes.

## Launch 🚀

In 99 percent of cases, we start tests via the `mix test` command. This script looks for test files in the test directories of the project, pre-launches the necessary applications, compiles the script files (a lot of important things happen at this step), and starts ExUnit itself.

Surprisingly, ExUnit starts with the following function:

```elixir
ExUnit.start()
```

In the project, this command is usually scripted in the *test/test_helper.exs* file.

Because ExUnit is an OTP Application, it's started with the `start/1` callback as:

```elixir
{:ok, _} = Application.ensure_all_started(:ex_unit)
```

ExUnit starts the following process tree under `ExUnit.Supervisor`:

- `ExUnit.Server` – The GenServer module works as a test case storage for async and sync tests separately. Each test case is added to `ExUnit.Server` using the `add_async_module/1` or `add_sync_module/1` functions during the compilation of the test module in the `__**after_compile__`** callback.
- `ExUnit.CaptureServer` – Module for working with I/O devices. It is needed when we're testing logging.
- `ExUnit.OnExitHandler` – The agent module responsible for executing the `setup_all` and `on_exit/2` callbacks if they were added to the test case. These callbacks are always executed in a separate process outside of the test process.

Now everything is ready to work!

## Run, ExUnit, run… 😣

There is a program (Beam VM) exit event handler created during the start of ExUnit. Inside this handler, the `ExUnit.Runner.run/2` function is called. And this function does all the work.

```elixir
System.at_exit(fn
        0 ->
          time = ExUnit.Server.modules_loaded(false)
          options = persist_defaults(configuration())
          %{failures: failures} = ExUnit.Runner.run(options, time)

          if failures > 0 do
            System.at_exit(fn _ -> exit({:shutdown, Keyword.fetch!(options, :exit_status)}) end)
          end

        _ ->
          :ok
      end)
```

Then I asked myself: why are the tests running inside this `System.at_exit/1` handler? What is the use of it?
As it turns out, it is created for the following case:

```elixir
defmodule Foo do
  # some code
end

ExUnit.start()

defmodule FooTest do
  use ExUnit.Case
  # tests
end
```

In this case, the test that is declared after the ExUnit start will be processed anyway. [Thanks to José Valim for the clarification.](https://elixirforum.com/t/why-exunit-does-the-job-in-system-at-exit-1/51641/2)

Keep in mind, this behaviour is correct when the `:ex_unit, :autorun` config is set to true, which is the default value. You can set it to false, and run tests manually using `ExUnit.run/0`.

The `ExUnit.Runner` module fills the “workers pool” up with async (if you remember this is the test with the `async: true` option) test cases taken from `ExUnit.Server`, and creates a separate process for each test case with `spawn_monitor/1`. The size of the pool is equal to `System.schedulers_online * 2` by default. But you can set any other value here.

When there are no more async test cases to execute, it executes sync tests sequentially inside a simple `for module <- sync_modules do` cycle.

### Running of tests in the test case

Each test case is defined as a separate module.

If there is a `setup_all` callback declared in the module, then `ExUnit.OnExitHandler` spawns a process to execute this callback and keeps the process alive until the tests are complete to keep access to resources created in the callback.
Then tests tagged as `:excluded` are filtered from the test case.
<!-- I don't know exactly, what you mean by this next sentence: -->
After the tests list was shuffled.

Finally, the remaining tests are executed inside `Enum.reduce_while/2`, and a separate process is created for each test.
If there are `setup` callbacks for the tests, they are executed before the tests are run. A callback receives the `context` as an argument. These callbacks are executed inside the test process.

The test itself is called in a very simple way:

```elixir
apply(module, name, [context])
```

Where the third argument `[context]` is the context returned by the `setup_all` and `setup` callbacks.

After all the tests inside the test case are complete, all the declared `on_exit/2` callbacks are called in the reserved order. Then the `:exit` signal is sent to the process where the `setup_all` and `on_exit/2` callbacks were called.

## Finishing the work 😮‍💨

Each test finishes with one of the following statuses:

1. `:test_finished` - The test is complete, stopping the monitoring of the test process. Completion of the test doesn’t mean that test was successful. If the test was unsuccessful then the `max_failures_reached?` check starts. If the counter of unsuccessful tests reached the number set in configs as `:ex_unit, :max_failures`, then execution of tests stops with the fail status. By default, the `:ex_unit, :max_failures` value is :infinity.
2. The test failed with some errors - In this case, failures counter increments and the `max_failures_reached?` check starts also.
3. The execution of the test takes longer than the `:ex_unit, :timeout` value set in the configuration - The default value is 60 seconds. In this case, the failure-counter is incremented and `max_failures_reached?` is checked.

In the end, `ExUnit.Runner.run/2` returns a tuple containing statistics of the finished tests.

## Printing on the screen 🖥️

There is `ExUnit.EventManager` which publishes events like `:suite_started`, `:case_started`, and `:test_started`. `ExUnit.CLIFormatter` listens to these events. For example when the `:test_finished` event happens, it prints out the well-known dot:

```elixir
IO.write(success(".", config))
```

The module prints out the other events in the same way.

## Summary

As you can see even with low-level process management ExUnit has fairy simple concepts under the hood. There are many more things beyond the scope of this article, but I hope it gave you some idea of how ExUnit works.
