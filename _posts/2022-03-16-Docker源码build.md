---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码build

## docker build
简单了解 docker build 的作用：
用户可以通过一个 自定义的 Dockerfile 文件以及相关内容，从一个基础镜像起步，对于 Dockerfile 中的每一 条命令，都在原先的镜像 layer 之上再额外构建一个新的镜像 layer ，直至构建出用户所需 的镜像。

由于 docker build 命令由 Docker 用户发起，故 docker build 的流程会贯穿 Docker Client Docker Server 以及 Docker Daemon 这三个重要的 Docker 模块。所以咱也是以这三个 Docker 模块为主题，分析 docker build 命令的执行，其中 Docker Daemon 最为重要。

1、Docker Client 作为用户请求的人口，自然第一个接收并处理 docker build 命令。主要包括：定义并解析flag参数、获取Dockerfile相关内容。

runBuild()函数只是从Client解析docker build命令和参数，对于我不怎么需要，先不分析。（点击runBuild可以跳转到代码）

2、docker server 负责根据请求类型以及请求的 URL ，路由转发 Docker 请求至相应的处理方法。在处理方法中， Docker Server 会创建相应的 Job ，为 Job 置相应的执行参数并触发该 Job 的运行。在此，将参数信息配置到JSON数据中。

其中，initRouter 将build命令的路由器初始化，initRouter.NewRouter.initRoutes.NewPostRoute.

NewPostRoute()调用 postBuild()函数

3、通过参数信息配置（newImageBuildOptions） 构建选项数据（buildOption）

getAuthConfig()请求header名为X-Registry-Config,值为用户的config信息（用户认证信息）

4、配置好选项数据，将其作为参数，调用imgID：= br.backend.Build()

5、return newBuilder(ctx, builderOptions).build(source, dockerfile)

newBuilder（）从可选的dockerfile和options创建一个新的Dockerfile构建器

build（）通过解析Dockerfile并执行文件中的指令来运行Dockerfile构建器

6、build（）函数调用dispatchFromDockerfile（），dispatchFromDockerfile（）调用dispatch（）函数。

dispatch（）函数找到每一种Dockerfile命令对应的handler处理函数并执行。

其中f, ok := evaluateTable[cmd]，对run（） ，即RUN指令。。。

7、run（）函数，是非常重要的，下面详细分析：
```
// RUN some command yo    //运行命令并提交image
//
// run a command and commit the image. Args are automatically prepended with
// the current SHELL which defaults to 'sh -c' under linux or 'cmd /S /C' under
// Windows, in the event there is only one argument The difference in processing:
//
// RUN echo hi          # sh -c echo hi       (Linux and LCOW)
// RUN echo hi          # cmd /S /C echo hi   (Windows)
// RUN [ "echo", "hi" ] # echo hi
//
func run(req dispatchRequest) error {
	if !req.state.hasFromImage() {
		return errors.New("Please provide a source image with `from` prior to run")
	}
 
	if err := req.flags.Parse(); err != nil {
		return err
	}
 
	//将最底层的Config结构体传入
	stateRunConfig := req.state.runConfig
	//处理JSON格式的参数，具体解释转至handleJSONArgs
	args := handleJSONArgs(req.args, req.attributes)
	if !req.attributes["json"] {
		args = append(getShell(stateRunConfig, req.builder.platform), args...)
	}
	//容器开始要执行的命令
	cmdFromArgs := strslice.StrSlice(args)
	//FilterAllowed返回所有允许的args，不带过滤的args
	//Env是要在容器中设置的环境变量列表
	buildArgs := req.builder.buildArgs.FilterAllowed(stateRunConfig.Env)
 
	//以下判断能否复用本地缓存
	saveCmd := cmdFromArgs
	if len(buildArgs) > 0 {
		saveCmd = prependEnvOnCmd(req.builder.buildArgs, buildArgs, cmdFromArgs)
	}
 
	runConfigForCacheProbe := copyRunConfig(stateRunConfig,
		withCmd(saveCmd),
		withEntrypointOverride(saveCmd, nil))
	hit, err := req.builder.probeCache(req.state, runConfigForCacheProbe)
	if err != nil || hit {
		return err
	}
 
	runConfig := copyRunConfig(stateRunConfig,
		withCmd(cmdFromArgs),
		withEnv(append(stateRunConfig.Env, buildArgs...)),
		withEntrypointOverride(saveCmd, strslice.StrSlice{""}))
 
	// set config as already being escaped, this prevents double escaping on windows
	//防止重复转义
	runConfig.ArgsEscaped = true
 
	logrus.Debugf("[BUILDER] Command to be executed: %v", runConfig.Cmd)
 
	//create根据基础镜像ID以及运行容器时所需的runconfig信息，来创建Container对象
	//进入create，调用Create，再调用ContainerCreate
	cID, err := req.builder.create(runConfig)
	if err != nil {
		return err
	}
	//Run运行Docker容器，其中包括创建容器文件系统、创建容器的命名空间进行资源隔离、为容器配置cgroups参数进行资源控制，还有运行用户指定的程序。
	if err := req.builder.containerManager.Run(req.builder.clientCtx, cID, req.builder.Stdout, req.builder.Stderr); err != nil {
		if err, ok := err.(*statusCodeError); ok {
			// TODO: change error type, because jsonmessage.JSONError assumes HTTP
			return &jsonmessage.JSONError{
				Message: fmt.Sprintf(
					"The command '%s' returned a non-zero code: %d",
					strings.Join(runConfig.Cmd, " "), err.StatusCode()),
				Code: err.StatusCode(),
			}
		}
		return err
	}
    //对运行后的容器进行commit操作，将运行的结果保存在一个新的镜像中。
    //需注意的是 commitContainer中的Commit（）函数是调用的/daemon/build.go 下面的
	return req.builder.commitContainer(req.state, cID, runConfigForCacheProbe)
}
```
先调用了Parse（）函数
```
// Parse reads lines from a Reader, parses the lines into an AST and returns
// the AST and escape token
//Parse从Reader读取行，将行解析为AST并返回AST和转义令牌（eacapeToken）
func Parse(rwc io.Reader) (*Result, error) {
	//NewDefaultDirective使用默认的eacapeToken标记返回一个新的Directive
	d := NewDefaultDirective()
	//当前行为0
	currentLine := 0
	//StartLines是Node开始的原始Dockerfile中的行，没明白-1是什么意思
	root := &Node{StartLine: -1}
	//从rws读取，返回一个新的scanner
	scanner := bufio.NewScanner(rwc)
	warnings := []string{}
 
	var err error
	//Scan将Scanner推进到下一个token，然后可通过Bytes或Text方法使用令牌
	for scanner.Scan() {
		//Bytes通过调用Scan生成最新的Token
		bytesRead := scanner.Bytes()
		if currentLine == 0 {
			// First line, strip the byte-order-marker if present
			//TrimPrefix返回没有包含字首的 对象s
			bytesRead = bytes.TrimPrefix(bytesRead, utf8bom)
		}
		//processLine 是弃用期后删除stripLeftWhitespace.
		//返回ReplaceAll，ReplaceAll返回src副本，将Regexp的匹配替换为替换文本repl
		//返回possibleParseDirective,possibleParseDirective查找一个或多个解析器指令'#escapeToken=<char>'和'#platform=<string>'.
		//解析器指令必须在任何构建器指令或其他注释之前，并且不能重复
		bytesRead, err = processLine(d, bytesRead, true)
		if err != nil {
			return nil, err
		}
		currentLine++
 
		startLine := currentLine
		//修改延续字符，应该是跟正则表达式的匹配有关系
		line, isEndOfLine := trimContinuationCharacter(string(bytesRead), d)
		if isEndOfLine && line == "" {
			continue
		}
 
		var hasEmptyContinuationLine bool
		for !isEndOfLine && scanner.Scan() {
			bytesRead, err := processLine(d, scanner.Bytes(), false)
			if err != nil {
				return nil, err
			}
			currentLine++
			//判断是不是注释
			if isComment(scanner.Bytes()) {
				// original line was a comment (processLine strips comments)
				continue
			}
			//判断是不是空的延续行
			if isEmptyContinuationLine(bytesRead) {
				hasEmptyContinuationLine = true
				continue
			}
 
			continuationLine := string(bytesRead)
			continuationLine, isEndOfLine = trimContinuationCharacter(continuationLine, d)
			line += continuationLine
		}
 
		if hasEmptyContinuationLine {
			warning := "[WARNING]: Empty continuation line found in:\n    " + line
			warnings = append(warnings, warning)
		}
		//newNodeFromLine将行拆分为多个部分，并根据命令和命令参数调度到一个函数（splitcommand()）。根据调度结果创建Node
		//具体解析转至newNodeFromLine
		child, err := newNodeFromLine(line, d)
		if err != nil {
			return nil, err
		}
		//AddChild添加一个新的子节点，并更新行的信息
		root.AddChild(child, startLine, currentLine)
	}
 
	if len(warnings) > 0 {
		warnings = append(warnings, "[WARNING]: Empty continuation lines will become errors in a future release.")
	}
	//Result是解析Dockerfile的结果（调用的本文件下的）
	return &Result{
		AST:         root,
		Warnings:    warnings,
		EscapeToken: d.escapeToken,
		Platform:    d.platformToken,
	}, nil
}
```
其中又调用了将行拆分的函数newNodeFromLine()
```
// newNodeFromLine splits the line into parts, and dispatches to a function
// based on the command and command arguments. A Node is created from the
// result of the dispatch.
func newNodeFromLine(line string, directive *Directive) (*Node, error) {
	//调用的splitCommand（）函数，他接受单行文本并解析cmd和args，这些用于调度到更精确的解析函数
	cmd, flags, args, err := splitCommand(line)
	if err != nil {
		return nil, err
	}
 
	fn := dispatch[cmd]
	// Ignore invalid Dockerfile instructions
	if fn == nil {
		fn = parseIgnore
	}
	next, attrs, err := fn(args, directive)
	if err != nil {
		return nil, err
	}
 
	return &Node{
		Value:      cmd,
		Original:   line,
		Flags:      flags,
		Next:       next,
		Attributes: attrs,
	}, nil
}
```
其中，又调用了splitCommand（）函数
```
// splitCommand takes a single line of text and parses out the cmd and args,
// which are used for dispatching to more exact parsing functions.
//splitCommand接受单行文本并解析cmd和args。这些用于调度到更精确的解析函数
func splitCommand(line string) (string, []string, string, error) {
	var args string
	var flags []string
 
	// Make sure we get the same results irrespective of leading/trailing spaces
	//无论leading/trailing（前导/后缀）有多少的空格，都得确保得到相同的结果
	//Split将切片拆分为由表达式分隔的子字符串，并返回这些表达式匹配之间的子字符串切片
	//TrimSpace 返回字符串s的一部分，删除所有的leading/trailing（前导/后缀）空格
	// 2 表示只有两个子字符串
	cmdline := tokenWhitespace.Split(strings.TrimSpace(line), 2)
	//cmdline[0]表示命令类型 如：groupadd
	//cmdline[1]表示命令参数 如：-f -g 842
	cmd := strings.ToLower(cmdline[0])
 
	if len(cmdline) == 2 {
		var err error
		//extractBuilderFlags() 解析BuilderFlags，并返回该行剩余部分
		args, flags, err = extractBuilderFlags(cmdline[1])
		if err != nil {
			return "", nil, "", err
		}
	}
	//返回 cmd  选项  参数  
	return cmd, flags, strings.TrimSpace(args), nil
}
```
其中解析命令参数的时候调用了extractBuilderFlags（）函数，这个函数就是解析剩下还有什么命令、选项、参数。
```
func extractBuilderFlags(line string) (string, []string, error) {
	// Parses the BuilderFlags and returns the remaining part of the line
	//解析BuilderFlags，并返回该行剩余部分
	const (
		inSpaces = iota // looking for start of a word  // 每次出现从 0 开始
		inWord											// 1
		inQuote											// 2
	)
 
	words := []string{}
	phase := inSpaces 
	word := ""
	quote := '\000'
	blankOK := false
	var ch rune
 
	for pos := 0; pos <= len(line); pos++ {
		if pos != len(line) {
			ch = rune(line[pos])
		}
 
		if phase == inSpaces { // Looking for start of word
			if pos == len(line) { // end of input
				break
			}
			if unicode.IsSpace(ch) { // skip spaces
				continue
			}
 
			// Only keep going if the next word starts with --
			if ch != '-' || pos+1 == len(line) || rune(line[pos+1]) != '-' {
				return line[pos:], words, nil
			}
 
			phase = inWord // found something with "--", fall through
		}
		if (phase == inWord || phase == inQuote) && (pos == len(line)) {
			if word != "--" && (blankOK || len(word) > 0) {
				words = append(words, word)
			}
			break
		}
		if phase == inWord {
			if unicode.IsSpace(ch) {
				phase = inSpaces
				if word == "--" {
					return line[pos:], words, nil
				}
				if blankOK || len(word) > 0 {
					words = append(words, word)
				}
				word = ""
				blankOK = false
				continue
			}
			if ch == '\'' || ch == '"' {
				quote = ch
				blankOK = true
				phase = inQuote
				continue
			}
			if ch == '\\' {
				if pos+1 == len(line) {
					continue // just skip \ at end
				}
				pos++
				ch = rune(line[pos])
			}
			word += string(ch)
			continue
		}
		if phase == inQuote {
			if ch == quote {
				phase = inWord
				continue
			}
			if ch == '\\' {
				if pos+1 == len(line) {
					phase = inWord
					continue // just skip \ at end
				}
				pos++
				ch = rune(line[pos])
			}
			word += string(ch)
		}
	}
 
	return "", words, nil
}
```
到此为止，Parse（）函数已经分析完毕。回到run（）函数上来：

（1）、create（）函数执行创建Container对象操作；

（2）、Run（）函数执行运行容器操作；

（3）、commitContainer（）函数执行提交新镜像操作。

注意：commitContainer（）调用的b.docker.Commit（），一定是/daemon/build.go下面的
