<!-- {%comment%} -->
# Docker Compose Networks 이름 설정하기
<!-- {%endcomment%} -->

Docker Compose networks 설정을 했을 때, `docker network ls`를 하여 network name을 확인해보면 특정 형태의 접두사가 포함되곤 합니다.

이 글에서는 network name 접두사가 어떻게 생성되는지 확인해보고, 접두사 없이 network name을 내가 원하는 name으로 설정할 수 있는 지를 알아보려고 합니다.

**`docker-compose-without-networks-name.yml`**

```yaml
version: '3.9'

services:
  hello-world:
    image: "hello-world:latest"

    networks:
      - foobar

# without name property
networks:
  foobar:
    driver: bridge
```

```console
amiere@LAPTOP-5BV4FPUC:~/mydir$ docker-compose -f ./docker-compose-without-networks-name.yml up
[+] Running 2/2
 ⠿ Network mydir_foobar           Created                                                    0.1s
 ⠿ Container mydir-hello-world-1  Created                                                    0.1s
Attaching to mydir-hello-world-1
mydir-hello-world-1  |
mydir-hello-world-1  | Hello from Docker!
mydir-hello-world-1  | This message shows that your installation appears to be working correctly.

...

```

```console
amiere@LAPTOP-5BV4FPUC:~/mydir$ pwd
/home/amiere/mydir

amiere@LAPTOP-5BV4FPUC:~/mydir$ docker network ls
NETWORK ID     NAME                                           DRIVER    SCOPE
23aa4273c50b   bridge                                         bridge    local
060edc7d35a8   host                                           host      local
08126381cd1f   mydir_foobar                                   bridge    local
b58714b49b7e   none                                           null      local

...

```

위와 같이 compose yaml을 설정하고, `mydir` directory에서 `docker-compose up`을 실행하면 NETWORK NAME이 `mydir_foobar`로 설정되어 생성되는 것을 확인할 수 있습니다.

이것을 봐서는 network name의 접두사는 기본적으로 `docker-compose` 명령어가 실행되는 directory로 보입니다. 정확히는 compose project name이 기본 접두사가 되는데, 자세한 내용은 이 글의 후반부에서 다루겠습니다.

이번에는 compose yaml을 아래와 같이 변경하고 실행을 해보겠습니다. 차이점은 networks 부분에 `name`이라는 하위 key를 추가로 설정해준 것입니다. 이 설정은 compose file version 3.5 이상에서 적용 가능합니다.

[Compose file version 3.5](https://docs.docker.com/compose/compose-file/compose-versioning/#version-35)

[compose-file-v3/#name-1](https://docs.docker.com/compose/compose-file/compose-file-v3/#name-1)

**`docker-compose-with-networks-name.yml`**

```yaml
version: '3.9'

services:
  hello-world:
    image: "hello-world:latest"

    networks:
      - foobar

# with name property
# Compose file version >= 3.5
# https://docs.docker.com/compose/compose-file/compose-versioning/#version-35
networks:
  foobar:
    driver: bridge
    name: foobar
```

```console
amiere@LAPTOP-5BV4FPUC:~/mydir$ docker-compose -f ./docker-compose-with-networks-name.yml up
[+] Running 2/2
 ⠿ Network foobar                 Created                                                    0.1s
 ⠿ Container mydir-hello-world-1  Created                                                    1.0s
Attaching to mydir-hello-world-1
mydir-hello-world-1  |
mydir-hello-world-1  | Hello from Docker!
mydir-hello-world-1  | This message shows that your installation appears to be working correctly.

...

```

```console
amiere@LAPTOP-5BV4FPUC:~/mydir$ docker network ls
NETWORK ID     NAME                                           DRIVER    SCOPE
23aa4273c50b   bridge                                         bridge    local
14f095601a79   foobar                                         bridge    local
060edc7d35a8   host                                           host      local
b58714b49b7e   none                                           null      local

...

```

network 하위 key로 `name`을 설정하면 위와 같이 접두사 없이 `name`에서 설정한 이름으로 NETWORK NAME이 설정됩니다.

compose file version 3.5 이상에서는 `networks`에 대해 하위 key `name`을 사용하여 기본적으로 생성되는 접두사 없이 NETWORK NAME을 설정할 수 있습니다.

한편으로 위의 과정을 살펴보시면 `docker-compose up`에서 생성되는 container의 이름 또한 자동으로 접두사가 생성되는 것을 확인할 수 있습니다.

이 부분을 변경하고자 할 경우에는 `services` 하위 key로 `container_name`을 설정하면 됩니다.

[compose-file-v3/#container_name](https://docs.docker.com/compose/compose-file/compose-file-v3/#container_name)

**`docker-compose-with-name.yml`**

```yaml
version: '3.9'

services:
  hello-world:
    container_name: hello-foobar
    image: "hello-world:latest"

    networks:
      - foobar

# with name property
# Compose file version >= 3.5
# https://docs.docker.com/compose/compose-file/compose-versioning/#version-35
networks:
  foobar:
    driver: bridge
    name: foobar
```

```console
amiere@LAPTOP-5BV4FPUC:~/mydir$ docker-compose -f docker-compose-with-name.yml up
[+] Running 2/2
 ⠿ Network foobar          Created                                                                                 0.0s
 ⠿ Container hello-foobar  Created                                                                                 0.2s
Attaching to hello-foobar
hello-foobar  |
hello-foobar  | Hello from Docker!
hello-foobar  | This message shows that your installation appears to be working correctly.

...

```

위와 같이 container 이름이 접두사 없이 `hello-foobar`로 설정된 것을 확인할 수 있습니다. `container_name`의 경우, 아래와 같은 주의사항이 있습니다.

> - Note when using docker stack deploy  
The container_name option is ignored when deploying a stack in swarm mode

---

## Docker Compose networks name default prefix

networks name 기본 접두사는 어떻게 설정되는 것인지 알아보고자 합니다.

docker-compose는 cobra library를 사용합니다.

[Cobra is a library for creating powerful modern CLI applications.](https://github.com/spf13/cobra)

우리가 사용하는 docker-compose 명령어는 아래에서 시작합니다.

아래의 github source code permalink는 2022년 3월 26일을 기준으로 하였습니다.

[docker compose v2 compose.go#L228](https://github.com/docker/compose/blob/v2/cmd/compose/compose.go#L228)

```go
func RootCommand(dockerCli command.Cli, backend api.Service) *cobra.Command {
	opts := projectOptions{}
	var (
		ansi    string
		noAnsi  bool
		verbose bool
		version bool
	)
	command := &cobra.Command{
		Short:            "Docker Compose",
		Use:              PluginName,
		TraverseChildren: true,

...

```

[docker compose v2 compose.go#L293](https://github.com/docker/compose/blob/v2/cmd/compose/compose.go#L293)

```go
	command.AddCommand(
		upCommand(&opts, backend),
		downCommand(&opts, backend),
		startCommand(&opts, backend),
		restartCommand(&opts, backend),
		stopCommand(&opts, backend),

...

```

`upCommand`, `downCommand` 등이 우리가 `docker-compose up`와 같은 명령어를 사용하는 것과 관련 있는 부분입니다.

[docker compose v2 up.go#L95](https://github.com/docker/compose/blob/be187bae649521f5776033284a511da0fbaf8058/cmd/compose/up.go#L95)

```go
func upCommand(p *projectOptions, backend api.Service) *cobra.Command {
	up := upOptions{}
	create := createOptions{}
	upCmd := &cobra.Command{
		Use:   "up [SERVICE...]",
		Short: "Create and start containers",
		PreRunE: AdaptCmd(func(ctx context.Context, cmd *cobra.Command, args []string) error {
			create.timeChanged = cmd.Flags().Changed("timeout")
			return validateFlags(&up, &create)
		}),
		RunE: p.WithServices(func(ctx context.Context, project *types.Project, services []string) error {
			create.ignoreOrphans = utils.StringToBool(project.Environment["COMPOSE_IGNORE_ORPHANS"])
			if create.ignoreOrphans && create.removeOrphans {
				return fmt.Errorf("COMPOSE_IGNORE_ORPHANS and --remove-orphans cannot be combined")
			}
			return runUp(ctx, backend, create, up, project, services)
		}),

...

```

여기서 `WithServices` 부분을 살펴보면 다음과 같습니다.

[docker compose v2 compose.go#L119](https://github.com/docker/compose/blob/be187bae649521f5776033284a511da0fbaf8058/cmd/compose/compose.go#L119)

```go
// WithServices creates a cobra run command from a ProjectFunc based on configured project options and selected services
func (o *projectOptions) WithServices(fn ProjectServicesFunc) func(cmd *cobra.Command, args []string) error {
	return Adapt(func(ctx context.Context, args []string) error {
		project, err := o.toProject(args, cli.WithResolvedPaths(true))
		if err != nil {
			return err
		}

		return fn(ctx, project, args)
	})
}
```

여기서 `toProject` 를 살펴보면 아래와 같습니다.

[docker compose v2 compose.go#L153](https://github.com/docker/compose/blob/be187bae649521f5776033284a511da0fbaf8058/cmd/compose/compose.go#L153)

```go
func (o *projectOptions) toProject(services []string, po ...cli.ProjectOptionsFn) (*types.Project, error) {
	options, err := o.toProjectOptions(po...)
	if err != nil {
		return nil, compose.WrapComposeError(err)
	}

	project, err := cli.ProjectFromOptions(options)
	if err != nil {
		return nil, compose.WrapComposeError(err)
	}

...

```

`toProjectOptions`와 `cli.ProjectFromOptions`를 살펴보겠습니다.

먼저 `toProjectOptions` 를 살펴보면 아래와 같습니다.

[docker compose v2 compose.go#L207](https://github.com/docker/compose/blob/be187bae649521f5776033284a511da0fbaf8058/cmd/compose/compose.go#L207)

```go
func (o *projectOptions) toProjectOptions(po ...cli.ProjectOptionsFn) (*cli.ProjectOptions, error) {
	return cli.NewProjectOptions(o.ConfigPaths,
		append(po,
			cli.WithWorkingDirectory(o.ProjectDir),
			cli.WithEnvFile(o.EnvFile),
			cli.WithDotEnv,
			cli.WithOsEnv,
			cli.WithConfigFileEnv,
			cli.WithDefaultConfigPath,
			cli.WithName(o.ProjectName))...)
}
```

`cli.WithName(o.ProjectNme)`과 관련하여 아래와 같이 살펴볼 수 있습니다.

우선 `o.ProjectName`은 `addProjectFlags`에서 설정이 되는데, 이는 우리가 `docker-compose -p NAME` 옵션을 사용했을 때 설정이 되는 부분입니다.

우리가 지정한 NAME 을 갖거나, default로 ""으로 설정됩니다.

```console
Define and run multi-container applications with Docker.

Usage:
  docker-compose [-f <arg>...] [--profile <name>...] [options] [COMMAND] [ARGS...]
  docker-compose -h|--help

Options:
  -f, --file FILE             Specify an alternate compose file
                              (default: docker-compose.yml)
  -p, --project-name NAME     Specify an alternate project name
                              (default: directory name)

...

```

[docker compose v2 compose.go#L130](https://github.com/docker/compose/blob/be187bae649521f5776033284a511da0fbaf8058/cmd/compose/compose.go#L130)

```go
func (o *projectOptions) addProjectFlags(f *pflag.FlagSet) {
	f.StringArrayVar(&o.Profiles, "profile", []string{}, "Specify a profile to enable")
	f.StringVarP(&o.ProjectName, "project-name", "p", "", "Project name")

...

```

`cli.WithName(o.ProjectNme)`의 `WithName`은 아래와 같습니다.

[compose-spec compose-go cli options.go#L63](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/cli/options.go#L63)

```go
func WithName(name string) ProjectOptionsFn {
	return func(o *ProjectOptions) error {
		o.Name = name
		return nil
	}
}
```

이제 약간 다시 돌아와서 `cli.ProjectFromOptions`를 살펴보겠습니다.

[compose-spec compose-go cli options.go#L296](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/cli/options.go#L296)

```go
func ProjectFromOptions(options *ProjectOptions) (*types.Project, error) {
	configPaths, err := getConfigPathsFromOptions(options)

...

	workingDir, err := options.GetWorkingDir()
	if err != nil {
		return nil, err
	}
	absWorkingDir, err := filepath.Abs(workingDir)
	if err != nil {
		return nil, err
	}

	options.loadOptions = append(options.loadOptions, withNamePrecedenceLoad(absWorkingDir, options))

	project, err := loader.Load(types.ConfigDetails{
		ConfigFiles: configs,
		WorkingDir:  workingDir,
		Environment: options.Environment,
	}, options.loadOptions...)

...

```

`absWorkingDir`, `withNamePrecedenceLoad(absWorkingDir, options)`을 살펴보겠습니다. 그리고 이어서 `loader.Load`를 살펴보겠습니다.

[compose-spec compose-go cli options.go#L350](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/cli/options.go#L350)

```go
func withNamePrecedenceLoad(absWorkingDir string, options *ProjectOptions) func(*loader.Options) {
	return func(opts *loader.Options) {
		if options.Name != "" {
			opts.SetProjectName(options.Name, true)
		} else if nameFromEnv, ok := options.Environment[consts.ComposeProjectName]; ok && nameFromEnv != "" {
			opts.SetProjectName(nameFromEnv, true)
		} else {
			opts.SetProjectName(filepath.Base(absWorkingDir), false)
		}
	}
}
```

바로 여기서 `options.Name == ""` 일 경우에 `opts.SetProjcetName` 기본 설정이 이루어지게 됩니다.
`docker compose -p NAME`의 NAME을 전달하지 않을 경우 이곳에서 환경설정을 통해 설정이 되거나 `filepath.Base(absWorkingDir)`에 의해 설정되게 됩니다.

즉 docker compose project name이 절대경로 이름의 base가 되는 것입니다.

[compose-spec compose-go loader loader.go#L70](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/loader/loader.go#L70)

```go
func (o *Options) SetProjectName(name string, imperativelySet bool) {
	o.projectName = normalizeProjectName(name)
	o.projectNameImperativelySet = imperativelySet
}
```

이제 `cli.ProjectFromOptions` 내부의 `loader.Load`를 살펴보겠습니다.

[compose-spec compose-go loader loader.go#L145](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/loader/loader.go#L145)

```go
func Load(configDetails types.ConfigDetails, options ...func(*Options)) (*types.Project, error) {
	if len(configDetails.ConfigFiles) < 1 {
		return nil, errors.Errorf("No files specified")
	}

...

	if !opts.SkipNormalization {
		err = normalize(project, opts.ResolvePaths)
		if err != nil {
			return nil, err
		}
	}

...

```

`opts.SkipNormalization`의 기본값은 false이며 `normalize`가 실행되게 됩니다.

[compose-spec compose-go loader normalize.go#L31](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/loader/normalize.go#L31)

```go
// normalize compose project by moving deprecated attributes to their canonical position and injecting implicit defaults
func normalize(project *types.Project, resolvePaths bool) error {
	absWorkingDir, err := filepath.Abs(project.WorkingDir)
	if err != nil {
		return err
	}
	project.WorkingDir = absWorkingDir

...

	setNameFromKey(project)

...

```

`setNameFromKey` 여기서 default Name을 project.Name으로 설정하는 것을 확인할 수 있습니다.

[compose-spec compose-go loader normalize.go#L146](https://github.com/compose-spec/compose-go/blob/e3cf2790536b2a6110a5f9cd81da580775d1f9c4/loader/normalize.go#L146)

```go
func setNameFromKey(project *types.Project) {
	for i, n := range project.Networks {
		if n.Name == "" {
			n.Name = fmt.Sprintf("%s_%s", project.Name, i)
			project.Networks[i] = n
		}
	}
```

참고자료

[stack overflow: docker-compose v3 prepend folder name to network name](https://stackoverflow.com/questions/42570539/docker-compose-v3-prepend-folder-name-to-network-name)

[https://docs.docker.com/compose/compose-file/compose-file-v3/#name-1](https://docs.docker.com/compose/compose-file/compose-file-v3/#name-1)

[https://docs.docker.com/compose/compose-file/compose-versioning/#version-35](https://docs.docker.com/compose/compose-file/compose-versioning/#version-35)

[https://docs.docker.com/compose/compose-file/compose-file-v3/#container_name](https://docs.docker.com/compose/compose-file/compose-file-v3/#container_name)

[https://github.com/docker/compose/issues/3736](https://github.com/docker/compose/issues/3736)
