---
title: golang系列(32)--框架beego源码阅读(1)
date: 2021-01-14 14:39:02
tags:
- golang
categories:
- golang
---

`beego`作为国人的出品的`web`框架，工作中一直也没用过，能获得这么多`star`必然有过人之处，好好学习一下它的源码。


<!-- more -->

源码阅读从`bee`这个脚手架工具开始吧。[源码地址](https://github.com/beego/bee)


执行`go get github.com/beego/bee`，安装完成后，运行`bee`，

```shell
$ bee
Bee is a Fast and Flexible tool for managing your Beego Web Application.

USAGE
    bee command [arguments]

AVAILABLE COMMANDS

    version     Prints the current Bee version
    migrate     Runs database migrations
    api         Creates a Beego API application
    bale        Transforms non-Go files to Go source files
    fix         Fixes your application by making it compatible with newer versions of Beego
    pro         Source code generator
    dlv         Start a debugging session using Delve
    dockerize   Generates a Dockerfile for your Beego application
    generate    Source code generator
    hprose      Creates an RPC application based on Hprose and Beego frameworks
    new         Creates a Beego application
    pack        Compresses a Beego application into a single file
    rs          Run customized scripts
    run         Run the application by starting a local development server
    server      serving static content over HTTP on port
    update      Update Bee

Use bee help [command] for more information about a command.

ADDITIONAL HELP TOPICS


Use bee help [topic] for more information about that topic.
```

看下`main.go`，参数解析用的标准库`flag`实现。

```golang
func main() {
	utils.NoticeUpdateBee()
	flag.Usage = cmd.Usage
	flag.Parse()
	log.SetFlags(0)

	args := flag.Args()

	if len(args) < 1 {
		cmd.Usage()
		os.Exit(2)
		return
	}

	if args[0] == "help" {
		cmd.Help(args[1:])
		return
	}

    // 匹配命令运行
	for _, c := range commands.AvailableCommands {
		if c.Name() == args[0] && c.Run != nil {
			c.Flag.Usage = func() { c.Usage() }
			if c.CustomFlags {
				args = args[1:]
			} else {
				c.Flag.Parse(args[1:])
				args = c.Flag.Args()
			}

			if c.PreRun != nil {
				c.PreRun(c, args)
			}

			config.LoadConfig()
			os.Exit(c.Run(c, args))
			return
		}
	}

	utils.PrintErrorAndExit("Unknown subcommand", cmd.ErrorTemplate)
}
```

## command

`Command`定义了命令了需要具备的基本属性。

```golang
// Command is the unit of execution
type Command struct {
	// Run runs the command.
	// The args are the arguments after the command name.
	Run func(cmd *Command, args []string) int

	// PreRun performs an operation before running the command
	PreRun func(cmd *Command, args []string)

	// UsageLine is the one-line Usage message.
	// The first word in the line is taken to be the command name.
	UsageLine string

	// Short is the short description shown in the 'go help' output.
	Short string

	// Long is the long message shown in the 'go help <this-command>' output.
	Long string

	// Flag is a set of flags specific to this command.
	Flag flag.FlagSet

	// CustomFlags indicates that the command will do its own
	// flag parsing.
	CustomFlags bool

	// output out writer if set in SetOutput(w)
	output *io.Writer
}
```

创建`Command`也很简单。

```golang
var CmdFix = &commands.Command{
	UsageLine: "fix",
	Short:     "Fixes your application by making it compatible with newer versions of Beego",
	Long: `
  The command 'fix' will try to solve those issues by upgrading your code base
  to be compatible  with Beego old version
  -s source version
  -t target version

  example: bee fix -s 1 -t 2 means that upgrade Beego version from v1.x to v2.x
`,
}

var (
	source, target utils.DocValue
)

func init() {
	CmdFix.Run = runFix
	CmdFix.PreRun = func(cmd *commands.Command, args []string) { version.ShowShortVersionBanner() }
	CmdFix.Flag.Var(&source, "s", "source version")
	CmdFix.Flag.Var(&target, "t", "target version")
	commands.AvailableCommands = append(commands.AvailableCommands, CmdFix)
}

func runFix(cmd *commands.Command, args []string) int {
	t := target.String()
	if t == "" || t == "1.6" {
		return fixTo16(cmd, args)
	} else if strings.HasPrefix(t, "2") {
		// upgrade to v2
		return fix1To2()
	}

	beeLogger.Log.Info("The target is compatible version, do nothing")
	return 0
}
```


`commands`文件夹下实现了包括`migrate`在内的命令。

```shell
├── cmd
│   ├── bee.go
│   └── commands
│       ├── api
│       │   └── apiapp.go
│       ├── bale
│       │   └── bale.go
│       ├── beefix
│       │   ├── fix.go
│       │   ├── fix1To2.go
│       │   └── fixTo1.6.go
│       ├── beegopro
│       │   └── beegopro.go
│       ├── command.go
│       ├── dev
│       │   ├── cmd.go
│       │   └── githook.go
│       ├── dlv
│       │   ├── dlv.go
│       │   └── dlv_amd64.go
```

## generate

`generate`主要实现都基于模板的字符串替换`strings.Replace`。

```golang
func GenerateController(cname, currpath string) {
	w := colors.NewColorWriter(os.Stdout)

	p, f := path.Split(cname)
	controllerName := strings.Title(f)
	packageName := "controllers"

	if p != "" {
		i := strings.LastIndex(p[:len(p)-1], "/")
		packageName = p[i+1 : len(p)-1]
	}

	beeLogger.Log.Infof("Using '%s' as controller name", controllerName)
	beeLogger.Log.Infof("Using '%s' as package name", packageName)

	fp := path.Join(currpath, "controllers", p)
	if _, err := os.Stat(fp); os.IsNotExist(err) {
		// Create the controller's directory
		if err := os.MkdirAll(fp, 0777); err != nil {
			beeLogger.Log.Fatalf("Could not create controllers directory: %s", err)
		}
	}

	fpath := path.Join(fp, strings.ToLower(controllerName)+".go")
	if f, err := os.OpenFile(fpath, os.O_CREATE|os.O_EXCL|os.O_RDWR, 0666); err == nil {
		defer utils.CloseFile(f)

		modelPath := path.Join(currpath, "models", strings.ToLower(controllerName)+".go")

		var content string
		if _, err := os.Stat(modelPath); err == nil {
			beeLogger.Log.Infof("Using matching model '%s'", controllerName)
			content = strings.Replace(controllerModelTpl, "{{packageName}}", packageName, -1)
			pkgPath := getPackagePath(currpath)
			content = strings.Replace(content, "{{pkgPath}}", pkgPath, -1)
		} else {
			content = strings.Replace(controllerTpl, "{{packageName}}", packageName, -1)
		}

		content = strings.Replace(content, "{{controllerName}}", controllerName, -1)
		f.WriteString(content)

		// Run 'gofmt' on the generated source code
		utils.FormatSourceCode(fpath)
		fmt.Fprintf(w, "\t%s%screate%s\t %s%s\n", "\x1b[32m", "\x1b[1m", "\x1b[21m", fpath, "\x1b[0m")
	} else {
		beeLogger.Log.Fatalf("Could not create controller file: %s", err)
	}
}
```