---
title: Anbox 实现分析 1：程序入口
date: 2018-01-08 21:05:49
categories: 虚拟化
tags:
- 虚拟化
---

Anbox 的总体架构如 [运行 Anbox](https://www.wolfcstech.com/2017/11/28/run_anbox/) 一文的相关内容所述，其运行时主要由两个分开的实例构成，容器管理器和会话管理器。anbox 用同一个可执行文件，在启动时通过不同的参数实现运行时执行两块完全不同的逻辑，完成容器管理和会话管理的任务。
<!--more-->
在命令行中，为 `anbox` 可执行文件提供不同的 `command` 参数来确定具体执行什么样的实例。Anbox 通过同一个可执行文件，将多个功能完全不同的逻辑粘合起来。查看 `anbox` 的 help 信息，内容如下：
```
$ anbox help
NAME:
    anbox - anbox

USAGE:
    anbox [command options] [arguments...]

COMMANDS:
    help                 prints a short help message                                                                         
    system-info          Print various information about the system we're running on                                         
    version              print the version of the daemon                                                                     
    session-manager      Run the the anbox session manager                                                                   
    launch               Launch an Activity by sending an intent
```

`anbox` 可执行文件支持的 `command` 参数除了容器管理器的 `container-manager` 和会话管理器的 `session-manager`，还包括 `help`，`system-info`，`version`，`launch` 等。

`anbox` 应用程序的 `main()` 函数（位于 `anbox/src/main.cpp`）如下：
```
int main(int argc, char **argv) {
  anbox::Daemon daemon;
  return daemon.Run(anbox::utils::collect_arguments(argc, argv));
}
```

在 `main()` 函数中，创建了 `anbox::Daemon` 对象，通过 `anbox::utils::collect_arguments()` 函数将 C 风格的命令行参数字符串数组，转为命令行参数的 `std::string` 数组表示。`anbox::utils::collect_arguments()` 定义（位于 `anbox/src/anbox/utils.cpp` 文件中）如下：
```
std::vector<std::string> collect_arguments(int argc, char **argv) {
  std::vector<std::string> result;
  for (int i = 1; i < argc; i++) result.push_back(argv[i]);
  return result;
}
```

`main()` 函数完成一个简单的命令行参数转发，实际的应用程序入口位于 `anbox::Daemon` 类，该类定义（位于 `anbox/src/anbox/daemon.h`）如下：
```
namespace anbox {
class Daemon : public DoNotCopyOrMove {
 public:
  Daemon();

  int Run(const std::vector<std::string> &arguments);

 private:
  cli::CommandWithSubcommands cmd;
};
}  // namespace anbox
```

这个类只有一个类型为 `cli::CommandWithSubcommands` 的成员变量 `cmd`，用于组织 Anbox 支持的所有命令。

`anbox::Daemon` 类的实现（位于 `anbox/src/anbox/daemon.cpp`）如下：
```
namespace anbox {
Daemon::Daemon()
    : cmd{cli::Name{"anbox"}, cli::Usage{"anbox"},
          cli::Description{"The Android in a Box runtime"}} {
  cmd.command(std::make_shared<cmds::Version>())
     .command(std::make_shared<cmds::SessionManager>())
     .command(std::make_shared<cmds::Launch>())
     .command(std::make_shared<cmds::ContainerManager>())
     .command(std::make_shared<cmds::SystemInfo>());


  Log().Init(anbox::Logger::Severity::kWarning);

  const auto log_level = utils::get_env_value("ANBOX_LOG_LEVEL", "");
  if (!log_level.empty() && !Log().SetSeverityFromString(log_level))
    WARNING("Failed to set logging severity to '%s'", log_level);
}

int Daemon::Run(const std::vector<std::string> &arguments) try {
  auto argv = arguments;
  if (arguments.size() == 0) argv = {"run"};
  return cmd.run({std::cin, std::cout, argv});
} catch (std::exception &err) {
  ERROR("%s", err.what());

  return EXIT_FAILURE;
}
}  // namespace anbox
```

在 `anbox::Daemon` 类的构造函数中，收集支持的所有命令，并设置全局的日志等级，在 `Run()` 函数中，由标准输入流，标准输出流和参数数组构建 `cli::Command::Context` 传给 `cli::CommandWithSubcommands` 类的 `run()` 函数。`cli::CommandWithSubcommands` 类的 `run()` 函数是在应用的整个声明周期中永不结束的函数，`anbox::Daemon::Run()` 函数也一样，因而为 `cli::CommandWithSubcommands` 类的 `run()` 函数传递在栈上临时构造的 `cli::Command::Context` 对象的引用不会产生问题。

Anbox 的设计通过组合模式来组织各个命令，相关各个类的类图如下：

![Anbox Command Class Diagram](https://www.wolfcstech.com/images/1315506-8c796ad9f159f629.png)

Anbox 的这些 `Command` 类的基类 `anbox::cli::Command` 定义（位于 `anbox/src/anbox/cli.h`）如下：
```
template <std::size_t max>
class SizeConstrainedString {
 public:
  SizeConstrainedString(const std::string& s) : s{s} {
    if (s.size() > max)
      throw std::logic_error{"Max size exceeded " + std::to_string(max)};
  }

  const std::string& as_string() const { return s; }

  operator std::string() const { return s; }

 private:
  std::string s;
};
. . . . . .
// We are imposing size constraints to ensure a consistent CLI layout.
typedef SizeConstrainedString<20> Name;
typedef SizeConstrainedString<60> Usage;
typedef SizeConstrainedString<100> Description;
. . . . . .
/// @brief Command abstracts an individual command available from the daemon.
class Command : public DoNotCopyOrMove {
 public:
  // Safe us some typing
  typedef std::shared_ptr<Command> Ptr;

  /// @brief FlagsMissing is thrown if at least one required flag is missing.
  struct FlagsMissing : public std::runtime_error {
    /// @brief FlagsMissing initializes a new instance.
    FlagsMissing();
  };

  /// @brief FlagsWithWrongValue is thrown if a value passed on the command line
  /// is invalid.
  struct FlagsWithInvalidValue : public std::runtime_error {
    /// @brief FlagsWithInvalidValue initializes a new instance.
    FlagsWithInvalidValue();
  };

  /// @brief Context bundles information passed to Command::run invocations.
  struct Context {
    std::istream& cin;              ///< The std::istream that should be used for reading.
    std::ostream& cout;             ///< The std::ostream that should be used for writing.
    std::vector<std::string> args;  ///< The command line args.
  };

  /// @brief name returns the Name of the command.
  virtual Name name() const;

  /// @brief usage returns a short usage string for the command.
  virtual Usage usage() const;

  /// @brief description returns a longer string explaining the command.
  virtual Description description() const;

  /// @brief hidden returns if the command is hidden from the user or not.
  virtual bool hidden() const;

  /// @brief run puts the command to execution.
  virtual int run(const Context& context) = 0;

  /// @brief help prints information about a command to out.
  virtual void help(std::ostream& out) = 0;

 protected:
  /// @brief Command initializes a new instance with the given name, usage and
  /// description.
  Command(const Name& name, const Usage& usage, const Description& description, bool hidden = false);

  /// @brief name adjusts the name of the command to n.
  // virtual void name(const Name& n);
  /// @brief usage adjusts the usage string of the comand to u.
  // virtual void usage(const Usage& u);
  /// @brief description adjusts the description string of the command to d.
  // virtual void description(const Description& d);

 private:
  Name name_;
  Usage usage_;
  Description description_;
  bool hidden_;
};
```

`Name`、`Usage` 和 `Description` 都是长度受限的字符串的封装。`anbox::cli::Command` 类实现（位于 `anbox/src/anbox/cli.cpp`）如下
```
cli::Name cli::Command::name() const { return name_; }

cli::Usage cli::Command::usage() const { return usage_; }

cli::Description cli::Command::description() const { return description_; }

bool cli::Command::hidden() const { return hidden_; }

cli::Command::Command(const cli::Name& name, const cli::Usage& usage,
                      const cli::Description& description, bool hidden)
    : name_(name), usage_(usage), description_(description), hidden_(hidden) {}
```

`anbox::cli::Command` 类本身的实现主要是构造函数和几个 Getter 函数。`run()` 函数是命令执行的主体，也是 `anbox::cli::Command` 类最为重要的成员函数，其实现会交给其子类来完成。

`anbox::cli::Command` 类的子类 `anbox::cli::CommandWithSubcommands` 是 `anbox::cli::Command` 的容器，它集合了 Anbox 支持的所有命令，在执行时根据参数选择具体的 `anbox::cli::Command` 子类执行。`anbox::cli::CommandWithSubcommands` 类定义（位于 `anbox/src/anbox/cli.h`）如下：
```
/// @brief CommandWithSubcommands implements Command, selecting one of a set of
/// actions.
class CommandWithSubcommands : public Command {
 public:
  typedef std::shared_ptr<CommandWithSubcommands> Ptr;
  typedef std::function<int(const Context&)> Action;

  /// @brief CommandWithSubcommands initializes a new instance with the given
  /// name, usage and description.
  CommandWithSubcommands(const Name& name, const Usage& usage,
                         const Description& description);

  /// @brief command adds the given command to the set of known commands.
  CommandWithSubcommands& command(const Command::Ptr& command);

  /// @brief flag adds the given flag to the set of known flags.
  CommandWithSubcommands& flag(const Flag::Ptr& flag);

  // From Command
  int run(const Context& context) override;
  void help(std::ostream& out) override;

 private:
  std::unordered_map<std::string, Command::Ptr> commands_;
  std::set<Flag::Ptr> flags_;
};
```

`anbox::cli::CommandWithSubcommands` 类用一个 `std::unordered_map` 保存它维护的所有的 `anbox::cli::Command` 具体子类。`anbox::cli::CommandWithSubcommands` 类的实现（位于 `anbox/src/anbox/cli.cpp`）如下：
```
namespace {
namespace pattern {
static constexpr const char* help_for_command_with_subcommands =
    "NAME:\n"
    "    %1% - %2%\n"
    "\n"
    "USAGE:\n"
    "    %3% [command options] [arguments...]";

static constexpr const char* commands = "COMMANDS:";
static constexpr const char* command = "    %1% %2%";

static constexpr const char* options = "OPTIONS:";
static constexpr const char* option = "    --%1% %2%";
}
. . . . . .
cli::CommandWithSubcommands::CommandWithSubcommands(
    const Name& name, const Usage& usage, const Description& description)
    : Command{name, usage, description} {
  command(std::make_shared<cmd::Help>(*this));
}

cli::CommandWithSubcommands& cli::CommandWithSubcommands::command(
    const Command::Ptr& command) {
  commands_[command->name().as_string()] = command;
  return *this;
}

cli::CommandWithSubcommands& cli::CommandWithSubcommands::flag(
    const Flag::Ptr& flag) {
  flags_.insert(flag);
  return *this;
}

void cli::CommandWithSubcommands::help(std::ostream& out) {
  out << boost::format(pattern::help_for_command_with_subcommands) %
             name().as_string() % usage().as_string() % name().as_string()
      << std::endl;

  if (flags_.size() > 0) {
    out << std::endl
        << pattern::options << std::endl;
    for (const auto& flag : flags_)
      out << boost::format(pattern::option) % flag->name() % flag->description()
          << std::endl;
  }

  if (commands_.size() > 0) {
    out << std::endl
        << pattern::commands << std::endl;
    for (const auto& cmd : commands_) {
      if (cmd.second && !cmd.second->hidden())
        out << boost::format(pattern::command) % cmd.second->name() %
                   cmd.second->description()
            << std::endl;
    }
  }
}

int cli::CommandWithSubcommands::run(const cli::Command::Context& ctxt) {
  po::positional_options_description pdesc;
  pdesc.add("command", 1);

  po::options_description desc("Options");
  desc.add_options()("command", po::value<std::string>()->required(),
                     "the command to be executed");

  add_to_desc_for_flags(desc, flags_);

  try {
    po::variables_map vm;
    auto parsed = po::command_line_parser(ctxt.args)
                      .options(desc)
                      .positional(pdesc)
                      .style(po::command_line_style::unix_style)
                      .allow_unregistered()
                      .run();

    po::store(parsed, vm);
    po::notify(vm);

    auto cmd = commands_[vm["command"].as<std::string>()];
    if (!cmd) {
      ctxt.cout << "Unknown command '" << vm["command"].as<std::string>() << "'"
                << std::endl;
      help(ctxt.cout);
      return EXIT_FAILURE;
    }

    return cmd->run(cli::Command::Context{
        ctxt.cin, ctxt.cout,
        po::collect_unrecognized(parsed.options, po::include_positional)});
  } catch (const po::error& e) {
    ctxt.cout << e.what() << std::endl;
    help(ctxt.cout);
    return EXIT_FAILURE;
  }

  return EXIT_FAILURE;
}
```

`anbox::cli::CommandWithSubcommands` 类的 `command()` 函数主要用于添加 `Command` 元素，`flag()` 函数用于添加 `Flag` 元素。`help()` 函数用于输出帮助信息，它主要是根据格式字符串，将 `CommandWithSubcommands` 及所有的子命令的名字、描述等内容格式化并输出。

`run()` 函数解析命令行参数，选择适当的具体 `Command` 并执行。

`anbox::cli::Command` 类的子类 `anbox::cli::CommandWithFlagsAndAction` 用于描述可以带一些参数选项的具体的 `Command`，如容器管理器，会话管理器等。Anbox 的具体 `Command` 的定制行为，不是通过 override 该类的 `run()` 函数，而是通过定义一个 `std::function<int(const Context&)>` Action 函数来实现的。Anbox 的具体 `Command` 通过 `action()` 函数将定制了行为的 Action 提交给 `anbox::cli::CommandWithFlagsAndAction`。

`anbox::cli::CommandWithFlagsAndAction` 定义（位于 `anbox/src/anbox/cli.h`）如下：
```
/// @brief CommandWithFlagsAction implements Command, executing an Action after
/// handling
class CommandWithFlagsAndAction : public Command {
 public:
  typedef std::shared_ptr<CommandWithFlagsAndAction> Ptr;
  typedef std::function<int(const Context&)> Action;

  /// @brief CommandWithFlagsAndAction initializes a new instance with the given
  /// name, usage and description. Optionally the command can be marked as hidden.
  CommandWithFlagsAndAction(const Name& name, const Usage& usage,
                            const Description& description, bool hidden = false);

  /// @brief flag adds the given flag to the set of known flags.
  CommandWithFlagsAndAction& flag(const Flag::Ptr& flag);

  /// @brief action installs the given action.
  CommandWithFlagsAndAction& action(const Action& action);

  // From Command
  int run(const Context& context) override;
  void help(std::ostream& out) override;

 private:
  std::set<Flag::Ptr> flags_;
  Action action_;
};
```

Anbox 用 `Flag` 表示命令行参数选项，boost 可以辅助解析命令行参数并设置一些类型为 `std::string` 或 `bool` 之类的状态。通过 `flag()` 函数可以为具体 `Command` 添加一个命令行参数选项。

`anbox::cli::CommandWithFlagsAndAction` 的实现（位于 `anbox/src/anbox/cli.cpp`）如下：
```
void add_to_desc_for_flags(po::options_description& desc,
                           const std::set<cli::Flag::Ptr>& flags) {
  for (auto flag : flags) {
    po::value_semantic *spec = nullptr;
    flag->specify_option(spec);
    if (!spec) continue;
    desc.add_options()(flag->name().as_string().c_str(), spec,
                       flag->description().as_string().c_str());
  }
}
}
. . . . . .
cli::CommandWithFlagsAndAction::CommandWithFlagsAndAction(
    const Name& name, const Usage& usage, const Description& description, bool hidden)
    : Command{name, usage, description, hidden} {}

cli::CommandWithFlagsAndAction& cli::CommandWithFlagsAndAction::flag(
    const Flag::Ptr& flag) {
  flags_.insert(flag);
  return *this;
}

cli::CommandWithFlagsAndAction& cli::CommandWithFlagsAndAction::action(
    const Action& action) {
  action_ = action;
  return *this;
}

int cli::CommandWithFlagsAndAction::run(const Context& ctxt) {
  po::options_description cd(name().as_string());

  bool help_requested{false};
  cd.add_options()("help", po::bool_switch(&help_requested),
                   "produces a help message");

  add_to_desc_for_flags(cd, flags_);

  try {
    po::variables_map vm;
    auto parsed = po::command_line_parser(ctxt.args)
                      .options(cd)
                      .style(po::command_line_style::unix_style)
                      .allow_unregistered()
                      .run();
    po::store(parsed, vm);
    po::notify(vm);

    if (help_requested) {
      help(ctxt.cout);
      return EXIT_SUCCESS;
    }

    return action_(cli::Command::Context{
        ctxt.cin, ctxt.cout,
        po::collect_unrecognized(parsed.options, po::include_positional)});
  } catch (const po::error& e) {
    ctxt.cout << e.what() << std::endl;
    help(ctxt.cout);
    return EXIT_FAILURE;
  }

  return EXIT_FAILURE;
}

void cli::CommandWithFlagsAndAction::help(std::ostream& out) {
  out << boost::format(pattern::help_for_command_with_subcommands) %
             name().as_string() % description().as_string() % name().as_string()
      << std::endl;

  if (flags_.size() > 0) {
    out << std::endl
        << boost::format(pattern::options) << std::endl;
    for (const auto& flag : flags_)
      out << boost::format(pattern::option) % flag->name() % flag->description()
          << std::endl;
  }
}
```

`add_to_desc_for_flags()` 函数将 `flags_` 添加进 `po::options_description`，在后面通过 boost 的 `command_line_parser` 解析命令行参数时，与特定命令行参数选项相关联的状态会得到适当的更新。

`cli::CommandWithFlagsAndAction::run(const Context& ctxt)` 解析命令行参数并执行 Action。`cli::CommandWithFlagsAndAction::help(std::ostream& out)` 函数与 `cli::CommandWithSubcommands` 的相同函数的实现类似，它根据格式字符串，将命令行参数选项格式化并输出。

经过上面对 Anbox 的 `Command` 类结构体系的分析，我们获得了一个分析 Anbox 中如 `SessionManager` 和 `ContainerManager` 这样的具体 `Command` 实现的框架：
通过 `flag()` 函数可以提交一个 `Flag`，即一个命令行参数选项的描述及其关联的状态，该状态将在 `Command` 的 `run()` 函数执行初期通过解析命令行参数来更新；通过 `action()` 函数可以提交一个函数，作为 `Command` 行为的主体，该函数将会在 `Command` 的 `run()` 函数的最后执行。

无论是对哪个 `cli::CommandWithFlagsAndAction` 的子类的分析，我们都可以把它分成两部分来看：一是通过 `flag()` 函数提交 `Flag`，二是通过 `action()` 提交的函数。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。
