# 命令行工具cobra的使用

安装cobra

```vim
go get -v github.com/spf13/cobra/cobra
```

切换到GOPATH的src目录下并创建一个新文件夹：demo

```bash
cd $GOPATH/src
mkdir demo
cd demo
```

初始cobra

```livecodeserver
$  ../../bin/cobra 
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cobra [command]

Available Commands:
  add         Add a command to a Cobra Application
  help        Help about any command
  init        Initialize a Cobra Application

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -h, --help             help for cobra
  -l, --license string   name of license for the project
      --viper            use Viper for configuration (default true)

Use "cobra [command] --help" for more information about a command.
```



可以看到cobra支持两个命令，一个是init, 一个是add，其中init是初始化一个cobra工程，add是给工程中添加一个子命令

初始化该项目：

```awk
$ ../../bin/cobra init --pkg-name demo
```

执行完上述命令后会生成如下几个文件及文件夹

```maxima
$ tree 
.
├── cmd
│   └── root.go
├── LICENSE
└── main.go

1 directory, 3 files
```

main.go

```go
package main

import "imgctl/cmd"

func main() {
    cmd.Execute()
}
```

cmd/root.go

```go
package cmd

import (
  "fmt"
  "os"
  "github.com/spf13/cobra"

  homedir "github.com/mitchellh/go-homedir"
  "github.com/spf13/viper"

)


var cfgFile string


// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
  Use:   "cobra-demo",
  Short: "A brief description of your application",
  Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
  // Uncomment the following line if your bare application
  // has an action associated with it:
  //    Run: func(cmd *cobra.Command, args []string) { },
}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}

func init() {
  cobra.OnInitialize(initConfig)

  // Here you will define your flags and configuration settings.
  // Cobra supports persistent flags, which, if defined here,
  // will be global for your application.

  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra-demo.yaml)")


  // Cobra also supports local flags, which will only run
  // when this action is called directly.
  rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}


// initConfig reads in config file and ENV variables if set.
func initConfig() {
  if cfgFile != "" {
    // Use config file from the flag.
    viper.SetConfigFile(cfgFile)
  } else {
    // Find home directory.
    home, err := homedir.Dir()
    if err != nil {
      fmt.Println(err)
      os.Exit(1)
    }

    // Search config in home directory with name ".cobra-demo" (without extension).
    viper.AddConfigPath(home)
    viper.SetConfigName(".cobra-demo")
  }

  viper.AutomaticEnv() // read in environment variables that match

  // If a config file is found, read it in.
  if err := viper.ReadInConfig(); err == nil {
    fmt.Println("Using config file:", viper.ConfigFileUsed())
  }
}
```

如果要实现一个没有子命令的CLI工具，那么由cobra生成程序的操作就结束了，由于cobra的所有命令都是通过cobra.Command这个结构体实现的，因此这里就需要对root.go文件中的RootCmd进行修改：

cmd/root.go

```go
package cmd

import (
  "fmt"
  "os"
  "github.com/spf13/cobra"

  homedir "github.com/mitchellh/go-homedir"
  "github.com/spf13/viper"

)


var cfgFile string
var name string
var age int

// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
    Use:   "demo",
    Short: "A test demo",
    Long:  `Demo is a test appcation for print things`,
    // Uncomment the following line if your bare application
    // has an action associated with it:
    Run: func(cmd *cobra.Command, args []string) {
        if len(name) == 0 {
            cmd.Help()
            return
        }
        fmt.Println(name, age)
    },
}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}

func init() {
  cobra.OnInitialize(initConfig)

  // Here you will define your flags and configuration settings.
  // Cobra supports persistent flags, which, if defined here,
  // will be global for your application.

  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra-demo.yaml)")
  rootCmd.Flags().StringVarP(&name, "name", "n", "", "person's name")
  rootCmd.Flags().IntVarP(&age, "age", "a", 0, "person's age")

  // Cobra also supports local flags, which will only run
  // when this action is called directly.
  rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}


// initConfig reads in config file and ENV variables if set.
func initConfig() {
  if cfgFile != "" {
    // Use config file from the flag.
    viper.SetConfigFile(cfgFile)
  } else {
    // Find home directory.
    home, err := homedir.Dir()
    if err != nil {
      fmt.Println(err)
      os.Exit(1)
    }

    // Search config in home directory with name ".cobra-demo" (without extension).
    viper.AddConfigPath(home)
    viper.SetConfigName(".cobra-demo")
  }

  viper.AutomaticEnv() // read in environment variables that match

  // If a config file is found, read it in.
  if err := viper.ReadInConfig(); err == nil {
    fmt.Println("Using config file:", viper.ConfigFileUsed())
  }
}
```

到此我们的demo功能就编码完成了，下面编译运行

解决依赖

```maxima
go mod init
go mod vendor
```

运行demo

```maxima
$ go run main.go
Demo is a test appcation for print things

Usage:
  demo [flags]

Flags:
  -a, --age int       person's age
  -h, --help          help for demo
  -n, --name string   person's name
```

添加命令

```vim
$ ../../bin/cobra add  [cmdname] # 执行后会在cmd目录下生成一个cmdname.go的文件，具体处理函数在该文件中编辑
$ ../../bin/cobra add server
$ tree
.
├── cmd
│   ├── root.go
│   └── server.go # rootCmd的子命令
├── LICENSE
└── main.go

1 directory, 4 files
```

cmd/server.go

```awk
package cmd

import (
        "fmt"

        "github.com/spf13/cobra"
)

// serverCmd represents the server command
var serverCmd = &cobra.Command{
        Use:   "server",
        Short: "A brief description of your command",
        Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
        Run: func(cmd *cobra.Command, args []string) {
                fmt.Println("server called")
        },
}

func init() {
        rootCmd.AddCommand(serverCmd)

        // Here you will define your flags and configuration settings.

        // Cobra supports Persistent Flags which will work for this command
        // and all subcommands, e.g.:
        // serverCmd.PersistentFlags().String("foo", "", "A help for foo")

        // Cobra supports local flags which will only run when this command
        // is called directly, e.g.:
        // serverCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

$ ../../bin/cobra add create -p "serverCmd" #执行该命令是为server子命令再创建一级子命令，-p --parent
$ tree
.
├── cmd
│   ├── create.go # serverCmd的子命令
│   ├── root.go
│   └── server.go
├── LICENSE
└── main.go

1 directory, 5 files
```

cmd/create.go

```stylus
package cmd

import (
        "fmt"

        "github.com/spf13/cobra"
)

// creatCmd represents the creat command
var creatCmd = &cobra.Command{
        Use:   "creat",
        Short: "A brief description of your command",
        Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
        Run: func(cmd *cobra.Command, args []string) {
                fmt.Println("creat called")
        },
}

func init() {
        serverCmd.AddCommand(creatCmd)

        // Here you will define your flags and configuration settings.

        // Cobra supports Persistent Flags which will work for this command
        // and all subcommands, e.g.:
        // creatCmd.PersistentFlags().String("foo", "", "A help for foo")

        // Cobra supports local flags which will only run when this command
        // is called directly, e.g.:
        // creatCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}
```