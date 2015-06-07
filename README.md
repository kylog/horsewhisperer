# Horse Whisperer

## Wait, what?

The HorseWhisperer is a command line parser and chained execution framework written in C++ (11).

(The Horse Whisperer should not be considered as a replacement for a commandline flags processing
library like [gflags](https://code.google.com/p/gflags/), but instead be used in the
special case where your application will be performing consecutive (and independent)
actions and context sensitive command line arguments are desired.)

## How do I even?

Using The Horse Whisperer comes down to 6 steps

 * Optionally configuring the global context
 * Defining global flags
 * Defining a set of actions
 * Defining action specific flags
 * Parsing and validating the commandline (global flags, actions, action flags, and arguments)
 * Executing actions

### Before we begin

(A picture is sometimes worth up to as many as a thousand words so if the HOWTO doesn't make sense the first time around it is worth taking a look at the content of the examples directory.)

The Horse Whisperer can be used in your code by simply including the header file.

    #include "horse_whisperer.h"

It is implemented as a singleton object and in most cases it is expected to exist for the
duration of of the program invoking it.

When first referencing the singleton three flags will be created for you by default.

 * --help
 * --version
 * --verbose (and many levels of -v)

`--help` is context sensitive and at the global context will display a banner and the list of
all the defined flags and actions.

    $ myprog --help
      --help                      Shows this message
      --verbose                   Set verbose output
      --version                   Display version information

In an action context it will display action specific help which will be explained in the *Defining a set of actions*
section.

The `--version` flag will display version information about the application if any has been specified.

The `--verbose` flag doesn't output anything but does set two internal values which can be useful when logging
information at various levels. These two values are *verbose* and *vlevel*.

When the `--verbose` flag is set *verbose* is internally set to true and *vlevel* set to 1.

The *vlevel* value can also be set by using multiples of `-v`

    $ myprog -vvv

    vlevel == 3

At any point after declaring a flag it's value can be set or looked up by the following templated functions.

    template <typename FlagType>
    FlagType GetFlag(std::string flag_name)

    template <typename FlagType>
    bool SetFlag(std::string flag_name, FlagType value)

HorseWhisperer will throw an "undefined_flag_error" exception when trying to apply GetFlag or
SetFlag to an undefined flag.

The vlevel value can be accessed by calling GetFlag:

    GetFlag<int>("vlevel")

### Configuring the global context

The banner message displayed by the `--help` flag can be set with the `SetHelpBanner` function.

    // void SetHelpBanner(std::string banner);
    HorseWhisperer::SetHelpBanner("Usage: myprog [global options] <action> [options]");

will change the banner message to

    $ myprog --help
    Usage: myprog [global options] <action> [options]
      --help                      Shows this message
      --verbose                   Set verbose output
      --version                   Display version information


The version message displayed by the `--version` flag can be set with the `SetVersion` function.

    // void SetVersion(std::string version);
    HorseWhisperer::SetVersion("MyProg version - 0.1.0\n");

will change the version message to

    $ myprog --version
    MyProg version - 0.1.0

Finally, a word on delimiters. By default Horse Whisperer has no delimiting symbols and chaining commands
is done simply by following up one command with another. There are certain cases where this behaviour is not optimal, so it is possible to define delimiters with the *SetDelimiters* function. Note that it is not advised to use a dash as
a delimeter; dashes are used to identify flags.

    // void SetDelimiters(std::vector<std::string> delimiters)
    SetDelimiters(std::vector<std::string>{"+", ";", "_then"});

### Defining global flags

Global context flags can be defined with the `DefineGlobalFlag` templated function. The parameter list is long and worth looking at in depth before using it.

    template <typename FlagType>
    void DefineGlobalFlag(std::string aliases,
                          std::string description,
                          FlagType default_value,
                          std::function<void(FlagType)> flag_callback)

**aliases:** Combination of short and long names, space separated, which can be used to set and look up the flag.

**description:** Short description which will be displayed when the --help flag is used.

**default_value:** Default value of the flag which is set.

**flag_callback:** This is the flag validation callback and it will be called when the flag is set. This can be any function
you provide and it doesn't necessarily have to be used for validation. Be aware that `flag_validation_error` exceptions will be propagated whereas all other exceptions will be converted to such class.

    // validation function
    bool validate(int x) {
        // Max value we accept is 5
        if (x > 5) {
            throw flag_validation_error { "You have assigned too many ponies!" };
        }
    }

    ...

    HorseWhisperer::DefineGlobalFlag<int>("ponies", "all the ponies", 1, validation);

Defining this global flag will change the output of `--help`

    $ myprog --help
    Usage: myprog [global options] <action> [options]
      --help                      Shows this message.
      --verbose                   Set verbose output
      --version                   Display version information
      --ponies                    all the ponies

### Defining a set of actions

Actions can be defined with the `DefineAction` function. As with defining global flags, the parameter list
can be long so it is worth looking at it in detail.

    static void DefineAction(std::string action_name,
                             int arity,
                             bool chainable,
                             std::string description,
                             std::string help_string,
                             std::function<int(const std::vector<std::string>& args)> action_callback,
                             std::function<void(const std::vector<std::string>& args)> arguments_callback = nullptr)

**action_name:** The name of the action. This name will always be used to refer to the action during the life of your
application.

**arity:** Arity is the amount of parameters expected by the action. Arity can either be a positive or negative integer, both being interpreted slightly differently. A positive integer means that the action expects *exactly* that many parameters (if arity is passed as 3 then we will attempt to pass exactly 3 parameters to the action.) If arity is negative it means that the action expects *at least* that many parameters (if arity is passed as -3 it means that we will attempt to pass at least 3 parameters to the action, but all parameters will be processed and passed until either **the end of ARGV** or **a delimiting symbol** is reached.)

**chainable:** This boolean value defines whether an action can be chained with other actions or not.

**description:** Short description which will be displayed when the --help flag is used.

**help_string:** If `--help` is set in the context of an action instead of the global context the help string
function will be displayed.

**action_callback:** Here we have the meat of the action. This is the function that will be called when the action has
been parsed from the commandline and invoked by running the HorseWhisperer::Start() function. The action
callback is expected to return an int (like a main function would) and will be passed a `std::vector<std::string>` parameter
which will contain the arguments passed to the action.

    // gallop action callback
    int gallop(std::vector<std::string> arguments) {
        for (int i = 0; i < GetFlag<int>("ponies"); i++) {
            std::cout << "Galloping into the night!" << std::endl;
        }
        return 0;
    }

**arguments_callback (optional):** This optional callback will be executed by the HorseWhisperer::Parse() function, once the parsing operation is completed. Here you can process the arguments of a given action and validate them. The arguments are passed as a vector of strings, as for the above
action_callback. As for the flag validation callback, `action_validation_error` exceptions will be propagated; all other exceptions will be converted to such class.

    // trot arguments callback
    void trotArgumentsCallback(const std::vector<std::string>& arguments) {
        for (std::string arg : arguments) {
            if (arg.find("mode", 0) == std::string::npos) {
                throw action_validation_error { "unknown trot mode '" + arg + "'" };
            }
        }
    }


Here's how we define actions:

    HorseWhisperer::DefineAction("gallop", 0, true, "make the ponies gallop",
                                 "The horses, they be a galloping", gallop);
    HorseWhisperer::DefineAction("trot", -2, true, "make the ponies trot in some way",
                                 "The horses, they be trotting in some 'mode ...'",
                                 trot, trotArgumentsCallback);

This will change `--help` output to

    $ myprog --help
    Usage: myprog [global options] <action> [options]
      --help                      Shows this message
      --verbose                   Set verbose output
      --version                   Display version information
      --ponies                    all the ponies

    Actions:

      gallop                      make the ponies gallop
      trot                        make the ponies trot in some way

Action specific help is now available...

    $ myprog gallop --help
    The horses, they be a galloping

    $ myprog trot --help
    The horses, they be trotting in some 'mode ...'

and actions can be called

    $ myprog gallop
    Galloping into the night!

    $ myprog trot 'mode bullet' 'mode rocket'
    Trotting like a bullet
    Trotting like a rocket

### Defining action specific flags

When you've defined an action you are able to define action specific flags. These flags are
context sensitive and will **only** be successfully parsed when they are in the context of an action.
The function is very similar to `DefineGlobalFlag` but it is worth taking an in depth look at its parameters.

    template <typename FlagType>
    void DefineActionFlag(std::string action_name,
                          std::string aliases,
                          std::string description,
                          FlagType default_value,
                          std::function<bool(FlagType)> flag_callback)

**action_name:** The name of the action to bind this flag to.

**aliases:** Combination of short and long names, space separated, which can be used to set and look up the flag.

**description:** Short description which will be displayed when the --help flag is used.

**default_value:** Default value of the flag which is set.

**flag_callback:** The validation callback will be called when the flag is set. This can be any function
you provide and it doesn't necessarily have to be used for validation. Be aware that the function will be expected
to return a boolean value. If false is returned the flag's value will not be changed.

    // Flag defined with no validation function
    HorseWhisperer::DefineActionFlag<bool>("gallop", "tired", "are the horses tired?", false, nullptr);

Then when looking at `--help`

    $ myprog --help
    Usage: myprog [global options] <action> [options]
      --help                      Shows this message.
      --verbose                   Set verbose output
      --version                   Display version information
      --ponies                    all the ponies

    Actions:

      gallop                      make the ponies gallop

    copy action options:

      --tired                     are the horses tired?

To use the --tired flag, we add some logic in the action callback:

    // action callback
    int gallop(std::vector<std::string> arguments) {
        for (int i = 0; i < GetFlag<int>("ponies"); i++) {
            if (!GetFlag<int>("tired")) {
                std::cout << "Galloping into the night!" << std::endl;
            } else {
                std::cout << "The pony is too tired to gallop." << std::endl;
            }
        }
        return 0;
    }

When trying to set `--tired` outside of the gallop action's context a failure will occur.

    $ myprog --tired
    Unknown Flag: tired

But when set inside the context of `gallop`

    $ myprog gallop --tired
    The pony is too tired to gallop.

### Parsing commandline: global flags, actions, action flags, and action arguments

When all flags and actions have been defined we are ready to parse the commandline and build
a chain of contexts. The contexts contain the global flags, a list of actions and the order
they should be called in as well as the flags used only by them.

    // ParseResult Parse(int argc, char** argv)
    Parse(argc, argv); // The same argv and argc passed to main()

The HorseWhisperer::Parse function will return a value from the HorseWhisperer::ParseResult enum,  indicating the operation outcome.
The possible return values are listed below.

    ParseResult::OK;            // success
    ParseResult::HELP;          // help request
    ParseResult::VERSION;       // version request
    ParseResult::ERROR;         // failed to parse (e.g. missing argument)
    ParseResult::INVALID_FLAG;  // invalid flag (e.g. string instead of an integer)

When parsing a given flag value, the relevant flag validation callback will be executed and a `flag_validation_error` may be thrown.
Once the parsing operation is complete, the action validation callbacks will be executed to validate the action arguments; an `action_validation_error` may be thrown.

### Displaying the help message

If the HorseWhisperer::Parse function returns ParseResult::HELP, you can simply call
HorseWhisperer::ShowHelp to display the requested help message (global or action-specific).

### Executing actions

When the commandline has been parsed starting your chain of action is as simple as calling the Start() function. Actions will continue to be executed until either the end of the list is reached or an action fails. When finished it will return the result of all the executed actions and'ed together (a return value of 0 means everything succeeded, 1 means the last attempt at executing an action failed).
In case a previous HorseWhisperer::Parse call did not return ParseResult::OK, the start function will
immediately return 1, without executing any action callback.

    // int Start();
    return Start();
